# Roblox Studio Manager

Roblox Studio Manager is a macOS utility for discovering, downloading, testing and managing multiple Roblox Studio versions from a single terminal interface.

The manager separates verified functional builds from legacy archive builds and keeps local compatibility, availability and runtime checks organized in one place.

## Features

- Discover Roblox Studio builds
- Download available Studio versions
- Identify builds that are still functional
- Detect `OUT OF DATE` builds through runtime checks
- Check macOS compatibility
- Manage multiple installed versions
- Keep legacy builds in a separate archive section
- Block unwanted automatic updates on archived copies
- Verify CDN availability
- Track previous test results
- Repair required internal files and folders
- Remove temporary files safely
- Use a terminal interface with adaptive layout

## Requirements

- macOS
- zsh
- Internet connection for version discovery and downloads
- Terminal access to the Downloads folder
- Accessibility permission for Terminal
- Automation permission for Terminal when requested

The current terminal release is designed for macOS. Windows is not supported by this build.

## Installation

Clone the repository:

```bash
cd ~/Downloads
git clone https://github.com/scarcellagb/robloxstudiomanager.git
```

Enter the repository:

```bash
cd ~/Downloads/robloxstudiomanager
```

Make the manager executable:

```bash
chmod 755 "Roblox Studio/Strumenti/robloxstudio.command"
```

Remove the macOS quarantine attribute if macOS blocks the file:

```bash
xattr -d com.apple.quarantine "Roblox Studio/Strumenti/robloxstudio.command" 2>/dev/null
```

Start the manager:

```bash
"Roblox Studio/Strumenti/robloxstudio.command"
```

## Permissions

On first launch, macOS may require permissions for Terminal.

Enable the requested permissions in:

```text
System Settings
Privacy & Security
```

The manager may require:

- Files and Folders
- Accessibility
- Automation

These permissions are used for local file management, Studio runtime checks and interface detection.

## Main Sections

### Downloadable and Functional

Contains builds that have passed the required checks and are considered usable on the current Mac.

A build must be:

- downloadable
- compatible with the Mac
- successfully launched
- not detected as `OUT OF DATE`
- recently verified

### Installed Versions

Shows the Roblox Studio versions already stored locally and their current status.

### Verification

Allows individual builds to be tested and updates their compatibility and runtime status.

### Historical Archive

Contains older downloadable builds intended primarily for archival and preservation purposes.

Historical availability does not guarantee that Roblox currently accepts the build online.

### Maintenance

Provides cleanup, integrity checks and local manager maintenance.

## Updating

Update the repository with:

```bash
cd ~/Downloads/robloxstudiomanager
git pull
```

Then ensure the manager remains executable:

```bash
chmod 755 "Roblox Studio/Strumenti/robloxstudio.command"
```

If macOS applies quarantine again:

```bash
xattr -d com.apple.quarantine "Roblox Studio/Strumenti/robloxstudio.command" 2>/dev/null
```

## Integrity

Important runtime files are checked by the manager and required internal folders can be recreated when missing.

Before using a downloaded release, verify its checksum when a checksum file is provided:

```bash
shasum -a 256 -c SHA256SUMS
```

Do not modify internal manager files unless you know exactly what the change does.

Removing the entire Roblox Studio Manager folder is treated as a normal uninstall.

## Data

Runtime data such as compatibility results, verification status and local cache files are stored locally.

Public releases should not contain personal runtime history, credentials, tokens or machine-specific paths.

## Disclaimer

Roblox Studio Manager is an independent community utility and is not affiliated with, endorsed by or maintained by Roblox Corporation.

Roblox, Roblox Studio and related marks are property of Roblox Corporation.

Availability of an older Studio package does not guarantee that Roblox services will continue to accept that version.
