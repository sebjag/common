# Bluefin-Common Copilot Instructions

This document provides essential information for coding agents working with the bluefin-common repository.

## Repository Overview

**Bluefin-Common** is a shared OCI layer containing common configuration files used across all Bluefin variants (bluefin, bluefin-dx, bluefin-lts).

- **Type**: Minimal OCI container layer (system files only)
- **Purpose**: Centralize shared configuration to reduce duplication across bluefin and bluefin-lts
- **Base**: Built from scratch with COPY directive
- **Languages**: Configuration files (JSON, shell scripts, markdown)
- **Build System**: GitHub Actions with podman/buildah

## Repository Structure

### Root Directory Files
- `Containerfile` - Multi-stage build (scratch → ctx stage with system_files)
- `cosign.pub` - Container signing public key (shared with bluefin/bluefin-lts)
- `README.md` - Basic repository description

### Key Directories
- `system_files/` - All configuration files that get copied into bluefin images
  - `etc/ublue-os/` - System configuration files (bling.json, fastfetch.json, setup.json)
  - `usr/share/ublue-os/` - User-space configurations
    - `firefox-config/` - Firefox default settings
    - `flatpak-overrides/` - Flatpak app overrides
    - `just/` - Just recipe additions
    - `motd/` - Message of the day templates and tips
    - `privileged-setup.hooks.d/` - Privileged setup hooks
    - `system-setup.hooks.d/` - System setup hooks
    - `user-setup.hooks.d/` - User setup hooks

### GitHub Actions
- `.github/workflows/build.yml` - Simple build workflow using podman/buildah

## Build Instructions

### Prerequisites
This repository requires minimal tooling:
- **podman** and **buildah** (usually pre-installed on development systems)
- No Just, no pre-commit, no complex build dependencies

### Build Commands

**Build locally:**
```bash
# Build the container
buildah build -t bluefin-common:latest -f ./Containerfile .

# Inspect the built image
podman images bluefin-common
```

**Test the image:**
```bash
# Copy files from the container to verify structure
podman create --name test bluefin-common:latest
podman cp test:/system_files ./test-output
podman rm test
tree ./test-output
```

### Build Process
1. GitHub Actions triggers on push to main or PR
2. `buildah build` creates image from `Containerfile`
3. Image is pushed to `ghcr.io/projectbluefin/bluefin-common:latest`
4. Bluefin and bluefin-lts reference this image with `COPY --from=ghcr.io/ublue-os/bluefin-common:latest`

## Usage in Downstream Projects

Bluefin and bluefin-lts use this layer in their Containerfiles:

```dockerfile
FROM ghcr.io/ublue-os/bluefin-common:latest AS bluefin-common

# Later in the build:
COPY --from=bluefin-common /system_files /desired/destination
```

## Making Changes

### Modifying Configuration Files

1. **Edit files in `system_files/`** - Maintain the existing directory structure
2. **Test locally** with buildah to ensure no syntax errors
3. **Create PR** - GitHub Actions will build and validate

### Adding New Configuration Files

1. Place files in the appropriate subdirectory under `system_files/`
2. Follow the existing path conventions:
   - System configs: `system_files/etc/ublue-os/`
   - User configs: `system_files/usr/share/ublue-os/`
3. Ensure file permissions are correct (executables for scripts)

### Common Modification Patterns
- **Firefox configs**: Edit `system_files/usr/share/ublue-os/firefox-config/`
- **Setup hooks**: Modify scripts in `system_files/usr/share/ublue-os/*-setup.hooks.d/`
- **System settings**: Update JSON files in `system_files/etc/ublue-os/`
- **Just recipes**: Add/modify `.just` files in `system_files/usr/share/ublue-os/just/`

## Validation

### Manual Validation
```bash
# Check Containerfile syntax
buildah build --dry-run -f ./Containerfile .

# Validate JSON files
find system_files -name "*.json" -exec sh -c 'echo "Checking {}"; cat {} | jq . > /dev/null' \;

# Check shell script syntax
find system_files -name "*.sh" -exec bash -n {} \;
```

### GitHub Actions
The build workflow automatically:
- Builds the container with buildah
- Pushes to GHCR on merge to main
- Validates build succeeds on PRs

## Development Guidelines

### Making Changes
1. **Keep it simple** - This repo contains only configuration files
2. **Maintain structure** - Follow existing directory patterns
3. **Test locally** - Build with buildah before pushing
4. **No complex dependencies** - This is intentionally minimal

### File Editing Best Practices
- **JSON files**: Validate syntax with `jq` before committing
- **Shell scripts**: Check syntax with `bash -n script.sh`
- **Keep files small** - Each file should have a single, clear purpose
- **Document changes** - Update comments in configuration files

## Trust These Instructions

**This repository is intentionally simple.** It contains only:
- Configuration files in `system_files/`
- A minimal Containerfile
- A simple GitHub Actions workflow

There are no complex build systems, no package management, no multi-stage builds beyond the scratch→ctx pattern.

## Other Rules

- Use [conventional commits](https://www.conventionalcommits.org/en/v1.0.0/#specification) for all commits and PR titles
- Keep changes minimal and surgical
- This layer is used by both bluefin (ublue-os/bluefin) and bluefin-lts (ublue-os/bluefin-lts)
- Changes here affect all downstream Bluefin variants

## Attribution Requirements

AI agents must disclose what tool and model they are using in the "Assisted-by" commit footer:

```text
Assisted-by: [Model Name] via [Tool Name]
```

Example:

```text
Assisted-by: Claude 3.5 Sonnet via GitHub Copilot
```
