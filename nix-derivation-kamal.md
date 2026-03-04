# Updating the Nix Derivation for Kamal 2.10.1-hostari

This document provides instructions for updating your Nix package derivation for Kamal to use the new 2.10.1-hostari version with proper path resolution support.

## Overview

The 2.10.1-hostari version includes a critical fix for Nix compatibility through the `KAMAL_CWD` environment variable. This ensures that config file paths are resolved relative to where the user invokes kamal, not where the Nix package is installed.

## Changes Required

### 1. Update the Kamal Source

Update your Nix derivation to point to the new version:

```nix
kamal = buildRubyGem rec {
  pname = "kamal";
  version = "2.10.1-hostari";
  
  src = fetchFromGitHub {
    owner = "hostari";
    repo = "kamal";
    rev = "v${version}";  # or specific commit/branch like "update-to-2.10.1"
    sha256 = "...";  # Run nix-prefetch-url or similar to get the correct hash
  };
  
  # ... rest of your derivation
};
```

**Note**: Until this branch is tagged/released, you may want to use the branch name:
```nix
rev = "update-to-2.10.1";  # or the specific commit hash
```

### 2. Add the KAMAL_CWD Wrapper

The key fix requires setting `KAMAL_CWD` to preserve the working directory. Add a wrapper script to your derivation:

```nix
{ buildRubyGem, fetchFromGitHub, makeWrapper, ... }:

buildRubyGem rec {
  pname = "kamal";
  version = "2.10.1-hostari";
  
  # ... source configuration ...
  
  nativeBuildInputs = [ makeWrapper ];
  
  postInstall = ''
    # Wrap the kamal binary to set KAMAL_CWD
    wrapProgram $out/bin/kamal \
      --run 'export KAMAL_CWD="$PWD"'
  '';
}
```

### 3. Alternative: Shell Script Wrapper

If you can't modify the derivation directly, create a wrapper script:

```bash
#!/usr/bin/env bash
# Save as /usr/local/bin/kamal or ~/bin/kamal

# Preserve the current working directory
export KAMAL_CWD="$PWD"

# Call the Nix-installed kamal with all arguments
exec /nix/store/.../kamal-2.10.1-hostari/bin/kamal "$@"
```

Make it executable:
```bash
chmod +x /usr/local/bin/kamal
```

## How It Works

### The Problem

When Nix packages Ruby gems, it installs them in the Nix store (e.g., `/nix/store/...`). If Nix changes the working directory context (which can happen with certain wrapper configurations), several issues occur:

1. **Config file paths fail**: `File.expand_path` resolves relative to the wrong directory
2. **Git commands fail**: `git rev-parse` and other git operations run in the Nix store instead of the user's project directory
3. **File operations fail**: Any code using `Dir.pwd` gets the Nix store path instead of the user's working directory

This causes errors like:
- `Configuration file not found in /nix/store/.../config/deploy.yml`
- `fatal: not a git repository (or any of the parent directories): .git`

### The Solution

The updated code now changes the Ruby process's working directory at startup:

```ruby
# In lib/kamal/cli/base.rb
def initialize_commander
  # Change to the directory where kamal was invoked
  Dir.chdir(ENV["KAMAL_CWD"]) if ENV["KAMAL_CWD"]
  
  KAMAL.tap do |commander|
    commander.configure \
      config_file: Pathname.new(File.expand_path(options[:config_file])),
      # ...
  end
end
```

When `KAMAL_CWD` is set by the Nix wrapper, Kamal will:
- **Change the Ruby process's working directory** to `$KAMAL_CWD` (where the user invoked kamal)
- **All operations work correctly**:
  - Config file paths (both relative and absolute)
  - Git repository detection and commands
  - File operations using `Dir.pwd`
  - Hook execution directory
  - Relative path resolution

This is a complete fix that makes Kamal work exactly as if it were installed normally outside of Nix.

## Testing the Fix

After updating your derivation, test these scenarios:

### Test 1: Default Config (Relative Path)
```bash
cd /path/to/your/project
kamal deploy
# Should find: /path/to/your/project/config/deploy.yml
```

### Test 2: Explicit Relative Path
```bash
cd /path/to/your/project
kamal deploy -c config/deploy.yml
# Should find: /path/to/your/project/config/deploy.yml
```

### Test 3: Absolute Path
```bash
cd /any/directory
kamal deploy -c /path/to/your/project/config/deploy.yml
# Should find: /path/to/your/project/config/deploy.yml
```

### Test 4: Subdirectory Relative Path
```bash
cd /path/to/your/project
kamal deploy -c ../other-project/config/deploy.yml
# Should find: /path/to/other-project/config/deploy.yml
```

### Test 5: Git Version Detection
```bash
cd /path/to/your/git/repo
kamal deploy -c /tmp/deploy-xxx.yml
# Should detect git repository and use commit hash as version
# No "fatal: not a git repository" error
# No "no git repository found in /nix/store/..." error
```

All of these should work correctly with the `KAMAL_CWD` wrapper in place.

## Debugging

If kamal still can't find your config file:

### Check if KAMAL_CWD is Set
```bash
kamal deploy --verbose 2>&1 | grep -i "KAMAL_CWD"
```

### Verify the Wrapper
```bash
which kamal
# Should show your wrapper location, not the Nix store directly

cat $(which kamal)
# Should show the wrapper script with KAMAL_CWD export
```

### Test Manually
```bash
cd /path/to/your/project
KAMAL_CWD="$PWD" /nix/store/.../kamal-2.10.1-hostari/bin/kamal deploy
```

If this works but `kamal deploy` doesn't, your wrapper isn't being invoked correctly.

## Example Complete Derivation

Here's a complete example of what your Nix derivation might look like:

```nix
{ lib
, buildRubyGem
, fetchFromGitHub
, makeWrapper
, ruby
}:

buildRubyGem rec {
  pname = "kamal";
  version = "2.10.1-hostari";
  
  src = fetchFromGitHub {
    owner = "hostari";
    repo = "kamal";
    rev = "update-to-2.10.1";
    sha256 = lib.fakeSha256;  # Replace with actual hash
  };
  
  gemName = "kamal";
  
  nativeBuildInputs = [ makeWrapper ];
  
  propagatedBuildInputs = [
    # Add any Ruby gem dependencies here
  ];
  
  postInstall = ''
    # Wrap kamal to preserve working directory
    wrapProgram $out/bin/kamal \
      --run 'export KAMAL_CWD="$PWD"'
  '';
  
  meta = with lib; {
    description = "Deploy Docker containers to production servers with Kamal (Hostari fork)";
    homepage = "https://github.com/hostari/kamal";
    license = licenses.mit;
    maintainers = with maintainers; [ /* your name */ ];
  };
}
```

## What's New in 2.10.1-hostari

In addition to the Nix compatibility fix, this version includes:

1. **Basecamp v2.10.1 Features**: All improvements from Kamal v2.7.4 → v2.10.1
   - ERB trim mode support for cleaner templates
   - Configurable secrets path
   - Pre-connect secrets handling
   - Alias preprocessing with destination support
   - Better proxy configuration
   - Many bug fixes and improvements

2. **Hostari Enhancements**:
   - **Arguments tracking**: Hooks receive full command invocation history
   - **Post-deploy hook enhancements**: Hooks now know which accessory operation triggered them (boot, start, stop, restart, reboot) via the `subaction` parameter
   - **Better hook control**: Operations like restart and reboot fire a single hook after completion, not multiple hooks for sub-operations

3. **Nix Compatibility**:
   - `KAMAL_CWD` environment variable support with automatic `chdir`
   - **Fixes git repository detection**: No more "fatal: not a git repository" errors
   - **Fixes all path operations**: Config files, git commands, hooks, and file operations all work correctly
   - Works seamlessly with Nix package managers and wrappers
   - Ruby process runs in the correct working directory context

## Migration Notes

If you're upgrading from an older Kamal version:

1. **Review your deploy.yml**: Check the [Kamal changelog](https://github.com/basecamp/kamal/releases) for any breaking changes between your current version and v2.10.1

2. **Update hooks**: If you use post-deploy hooks with accessories, you can now use the `KAMAL_SUBACTION` environment variable to determine which operation triggered the hook

3. **Test in staging**: Always test the new version in your staging environment before deploying to production

## Support

If you encounter issues:

1. Check that your wrapper is correctly setting `KAMAL_CWD`
2. Verify you're using the 2.10.1-hostari version: `kamal version`
3. Try running with `--verbose` flag for detailed logging
4. Check the [hostari/kamal repository](https://github.com/hostari/kamal) for issues and updates

## Building the Nix Package

To get the correct sha256 hash for your derivation:

```bash
nix-prefetch-url --unpack https://github.com/hostari/kamal/archive/update-to-2.10.1.tar.gz
```

Or if using fetchFromGitHub:

```bash
nix-prefetch-git https://github.com/hostari/kamal --rev update-to-2.10.1
```

Then update your derivation with the resulting hash.
