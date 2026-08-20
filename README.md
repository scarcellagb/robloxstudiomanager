# Roblox Studio Manager

Roblox Studio Manager is a macOS terminal utility for discovering, downloading, testing and organizing multiple Roblox Studio versions.

The manager separates currently usable builds from historical archive builds and keeps discovery, installation, maintenance and configuration grouped into dedicated sections.

## Features

- Discover Roblox Studio builds
- Download available Studio versions from the Roblox CDN
- Identify builds that are still functional
- Detect `OUT OF DATE` builds through runtime checks
- Check macOS and CPU compatibility
- Compare two catalogued builds
- View a summary of verified, installed and historical versions
- Manage multiple installed versions
- Keep legacy builds in a separate archive section
- Import signed historical copies from a local archive
- Block unwanted automatic updates on installed copies
- Verify CDN availability
- Track previous test results locally
- Restore required internal manager files while the manager is running
- Use an adaptive terminal interface

## Requirements

- macOS
- zsh
- Internet connection for discovery and downloads
- Terminal access to the Downloads folder
- Accessibility permission for Terminal when runtime verification is used
- Automation permission for Terminal when requested

The current terminal release is for macOS. Windows is not supported by this build.

## Installation

Clone the repository:

```bash
cd ~/Downloads
git clone https://github.com/scarcellagb/robloxstudiomanager.git
cd robloxstudiomanager
```

Make the manager executable:

```bash
chmod 755 Strumenti/robloxstudio.command
```

If macOS applies a quarantine attribute to the launcher, remove it:

```bash
xattr -d com.apple.quarantine Strumenti/robloxstudio.command 2>/dev/null
```

Start the manager:

```bash
./Strumenti/robloxstudio.command
```

## Permissions

On first launch, macOS may request permissions for Terminal under:

```text
System Settings
Privacy & Security
```

The manager may require Files and Folders, Accessibility and Automation permissions. These permissions are used for local file management and Roblox Studio runtime checks.

## Main Sections

### Versions

Contains the version overview, verified downloadable builds, version discovery, individual verification and build comparison.

A build is shown as functional only after a recent runtime test and a compatible Mac result. Package availability alone is not treated as proof that a build still works.

### Installed

Shows Roblox Studio versions stored locally and groups launch, runtime verification, compatibility, update blocking and removal in one section.

### Archive

Contains older builds intended primarily for archival and preservation purposes and supports importing locally stored signed copies. Package availability does not guarantee that Roblox currently accepts a historical version online.

### Tools

Contains diagnostics, maintenance, integrity verification and manager update checks.

### Settings

Contains paths, required macOS permissions, network and CDN cache information, and local data information.

## Security

Version 4.3.0 keeps the security hardening introduced in 4.2.1 and reorganizes operations so related actions are easier to identify and audit.

The manager:

- restricts network transfers to HTTPS
- accepts only strictly formatted Roblox build and version identifiers
- validates ZIP archives before extraction
- verifies the macOS code signature of Roblox Studio applications before execution
- requires the Roblox Corporation signing team identifier `2CFABCH843`
- verifies that the application bundle version matches the requested version
- checks installed applications again before launch or runtime testing
- does not automatically execute manager updates downloaded from the Internet
- limits cleanup operations to files owned by the manager workflow
- defers terminal resize layout changes until a safe render point

The internal SHA-256 integrity system detects accidental corruption and supports runtime recovery. It is not a substitute for operating-system code signing and cannot make a shell script impossible for the local computer owner or sufficiently privileged malware to modify.

## Updating

Check for stable releases under `Tools > Updates` or update the repository manually:

```bash
cd ~/Downloads/robloxstudiomanager
git pull
chmod 755 Strumenti/robloxstudio.command
```

Before using a downloaded release archive, verify its checksum when `SHA256SUMS` is provided:

```bash
shasum -a 256 -c SHA256SUMS
```

## Uninstall

Quit the manager and remove the complete `robloxstudiomanager` folder. Removing the whole project folder is treated as a normal uninstall.

## Data

Compatibility results, verification status and cache data are stored locally. Runtime files and downloaded version folders are excluded by `.gitignore` so they are not added to the repository by default. Public release archives do not contain Git metadata, personal runtime history, credentials, tokens or machine-specific paths.

## Disclaimer

Roblox Studio Manager is an independent community utility and is not affiliated with, endorsed by or maintained by Roblox Corporation.

Roblox, Roblox Studio and related marks are property of Roblox Corporation.
