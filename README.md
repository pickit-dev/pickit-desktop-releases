# Pickit Desktop

Welcome to the **Pickit Desktop** application documentation. This guide provides an overview of the app, deployment options for Windows and macOS, installer flags, required permissions, auto-update behavior, and recommended best practices for your IT team.

Latest version when this document was written is **1.3.1** (released **2025-05-22**).
You can find the latest version here: [https://github.com/pickit-dev/pickit-desktop-releases/releases](https://github.com/pickit-dev/pickit-desktop-releases/releases)

## Table of Contents

1. [Introduction](#introduction)
2. [Deployment Overview](#deployment-overview)
3. [Windows Installer (NSIS)](#windows-installer-nsis)
    - [Supported Flags](#supported-flags)
    - [Usage Examples](#usage-examples)
    - [Custom Flag Implementation](#custom-flag-implementation)
1. [macOS Installation & Permissions](#macos-installation--permissions)
    - [Built-in Entitlements](#built-in-entitlements)
    - [Configuration Profiles (MDM)](#configuration-profiles-mdm)
1. [Auto-Updates](#auto-updates)
2. [Appendix & References](#appendix--references)

---

## Introduction

Pickit Desktop is a cross-platform Electron application for quickly accessing and managing digital assets on Windows and macOS. By default, it opens in a classic window, but you can switch to tray mode if you prefer. It supports silent installation, auto-update, drag-and-drop workflows, and file synchronization.

---

## Deployment Overview

- **Windows**: Distributed as an NSIS installer (`Pickit-Setup-1.3.1.exe`). Supports both interactive and silent modes with customizable command-line flags.
- **macOS**: Packaged as `.dmg` (x64 and arm64), hardened, and notarized. Requires system entitlements and optional MDM profiles for privacy permissions.
- **Auto-Updates**: Updates are downloaded and installed automatically in the background; no manual intervention from the end user is required.

---

## Windows Installer (NSIS)

Pickit Desktop’s Windows installer uses NSIS. You can control installation behavior via command-line flags.

### Supported Flags

| **Flag**                   | **Description**                                                                | **Example**                                                  |
| -------------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------------ |
| `/S`                       | **Silent install**—runs without any UI (must be the first flag)                | `Pickit-Setup-1.3.1.exe /S`                                  |
| `/D=<install_dir>`         | **Target directory** (only valid in silent mode; must be last flag, no quotes) | `Pickit-Setup-1.3.1.exe /S /D=C:\Program Files\Pickit`       |
| `--no-desktop-shortcut`    | (*Custom*) skip creating a desktop shortcut                                    | `Pickit-Setup-1.3.1.exe /S --no-desktop-shortcut`            |
| `--no-start-menu-shortcut` | (*Custom*) skip creating a Start menu shortcut                                 | `Pickit-Setup-1.3.1.exe /S --no-start-menu-shortcut`         |
| `--no-autostart`           | (*Custom*) disable adding Pickit to Windows Startup                            | `Pickit-Setup-1.3.1.exe /S --no-autostart`                   |
| `--log=<filename>`         | (*Custom*) write an installation log to the specified file                     | `Pickit-Setup-1.3.1.exe /S --log=C:\temp\pickit-install.log` |
| `/LANG=<language_code>`    | Choose UI language (if multiple languages are compiled into the installer)     | `Pickit-Setup-1.3.1.exe /S /D=C:\Pickit /LANG=sv`            |

> **Notes:**

- > Built-in NSIS flags are `/S` and `/D=`.
- > Custom flags (those beginning with `--`) must be explicitly parsed in the installer script.
- > Ensure that `/D=<install_dir>` is placed last when used.

---

### Usage Examples

#### 1. Silent installation to Program Files

```other
Start-Process -FilePath "\\share\Pickit-Setup-1.3.1.exe" -ArgumentList "/S", "/D=C:\Program Files\Pickit" -Wait
```

#### 2. Silent install without shortcuts or autostart

```other
Start-Process -FilePath "\\share\Pickit-Setup-1.3.1.exe" -ArgumentList "/S", "--no-desktop-shortcut", "--no-start-menu-shortcut", "--no-autostart" -Wait
```

#### 3. Silent install with log file for troubleshooting

```other
Start-Process -FilePath "\\share\Pickit-Setup-1.3.1.exe" -ArgumentList "/S", "--log=C:\temp\pickit-install.log" -Wait
```

---

### Custom Flag Implementation

Below is an example snippet for parsing custom flags in the NSIS installer. Include this in the **`.onInit`** section so you can conditionally skip creating shortcuts, disabling autostart, or writing a log file.

```other
!include "LogicLib.nsh"

Var NoDesktopShortcut
Var NoStartMenuShortcut
Var NoAutostart
Var LogFile

Function .onInit
  ${DoWhile} $CMDLINE <> ""
    StrCpy $R0 $CMDLINE
    ${WordFind} $R0 "--no-desktop-shortcut" $R1
    ${If} $R1 <> "-1"
      StrCpy $NoDesktopShortcut 1
    ${EndIf}

    ${WordFind} $R0 "--no-start-menu-shortcut" $R2
    ${If} $R2 <> "-1"
      StrCpy $NoStartMenuShortcut 1
    ${EndIf}

    ${WordFind} $R0 "--no-autostart" $R3
    ${If} $R3 <> "-1"
      StrCpy $NoAutostart 1
    ${EndIf}

    ${WordFind} $R0 "--log=" $R4
    ${If} $R4 <> "-1"
      ; Extract the filename after "--log="
      StrCpy $LogFile "$CMDLINE" "" "--log="
    ${EndIf}

    ; Remove the processed token from $CMDLINE to avoid infinite loop
    StrCpy $CMDLINE ""  
  ${Loop}
FunctionEnd
```

In the rest of your script, wrap the relevant sections accordingly:

```other
Section "Create Shortcuts"
  ${If} ${Not} $NoDesktopShortcut
    ; create desktop shortcut here
  ${EndIf}
  
  ${If} ${Not} $NoStartMenuShortcut
    ; create Start menu shortcut here
  ${EndIf}
SectionEnd

Section "Autostart Entry"
  ${If} ${Not} $NoAutostart
    ; add registry or shortcut to Startup folder
  ${EndIf}
SectionEnd

Section "Write Install Log"
  ${If} $LogFile <> ""
    ; open $LogFile for writing and redirect installation output
  ${EndIf}
SectionEnd
```

Refer to the [NSIS Command-Line Docs](https://nsis.sourceforge.io/Docs/Chapter4.html#section4.1) for more details on parsing tokens and handling `/S` vs. `/D=`.

---

## macOS Installation & Permissions

On macOS, Pickit Desktop ships with built-in entitlements and must be notarized. To run under strict privacy settings (e.g., in a managed enterprise environment), deploy an MDM Configuration Profile for system-level permissions.

### Built-in Entitlements

Below is the exact `entitlements.mac.plist` used in version **1.3.1**. This file must be included at build time so that the app can perform JIT compilation, allow unsigned executable memory, and access files the user explicitly selects.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" 
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>com.apple.security.cs.allow-jit</key>
    <true/>
    <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
    <true/>
    <key>com.apple.security.files.user-selected.read-write</key>
    <true/>
    <key>com.apple.security.cs.allow-dyld-environment-variables</key>
    <true/>
    <key>com.apple.security.cs.disable-library-validation</key>
    <true/>
  </dict>
</plist>
```

- **com.apple.security.cs.allow-jit**: Allows just-in-time compilation.
- **com.apple.security.cs.allow-unsigned-executable-memory**: Permits loading unsigned executable memory.
- **com.apple.security.files.user-selected.read-write**: Grants the app permission to read/write files chosen by the user.
- **com.apple.security.cs.allow-dyld-environment-variables**: Allows setting certain environment variables.
- **com.apple.security.cs.disable-library-validation**: Permits loading non-Apple-signed code (e.g., native modules).

---

### Configuration Profiles (MDM)

If you manage macOS devices via an MDM (e.g., Jamf, Microsoft Intune, Workspace ONE), you can deploy a `.mobileconfig` file to grant Pickit the required privacy permissions upfront. Below is a sample snippet:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" 
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>PayloadContent</key>
    <array>
      <dict>
        <key>PayloadType</key>
        <string>com.apple.security.privacy</string>
        <key>PayloadVersion</key>
        <integer>1</integer>
        <key>Services</key>
        <dict>
          <key>SystemPolicyNetworkClient</key>
            <true/>
          <key>SystemPolicyFilesUserSelectedReadWrite</key>
            <true/>
        </dict>
        <key>IdentifierType</key>
        <string>bundleID</string>
        <key>Identifier</key>
        <string>com.pickit.app</string>
      </dict>
    </array>
    <key>PayloadType</key>
    <string>Configuration</string>
    <key>PayloadVersion</key>
    <integer>1</integer>
    <key>PayloadIdentifier</key>
    <string>com.pickit.app.privacy</string>
    <key>PayloadUUID</key>
    <string>XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX</string>
    <key>PayloadDisplayName</key>
    <string>Pickit Desktop Privacy Permissions</string>
  </dict>
</plist>
```

- **SystemPolicyNetworkClient** (`com.apple.security.privacy.network.client`): Grants unrestricted outbound network access (required for sync and auto-update).
- **SystemPolicyFilesUserSelectedReadWrite** (`com.apple.security.privacy.files.user-selected.read-write`): Allows the user to pick files in Finder dialogs, and for the app to read/write them.
- **Identifier (bundleID)**: Must match the app’s bundle ID (`com.pickit.app`).

Deploy this `.mobileconfig` alongside your `.dmg` via your MDM of choice. Since the DMG is already notarized, macOS will trust it, once installed, the app will start with these permissions granted.

---

## Auto-Updates

Pickit Desktop automatically checks for and installs updates in the background. Updates download and install silently, the user is prompted to restart only if required. No additional scripts or manual steps are needed, updates just happen.

---

## Appendix & References

- **NSIS command-line docs** (how to parse and handle flags):
[https://nsis.sourceforge.io/Docs/Chapter4.html#section4.1](https://nsis.sourceforge.io/Docs/Chapter4.html#section4.1)
- **Apple Developer PPPC documentation** (for in-depth details on privacy entitlements and MDM profiles):
[https://developer.apple.com/documentation/macos-release-notes/](https://developer.apple.com/documentation/macos-release-notes/)
- **Electron-builder NSIS configuration** (details on Windows/NSIS settings):
[https://www.electron.build/configuration/nsis](https://www.electron.build/configuration/nsis)
- **Electron-builder macOS configuration** (entitlements, hardened runtime, DMG layout):
[https://www.electron.build/configuration/mac](https://www.electron.build/configuration/mac)

---

*Document provided for IT teams and maintained externally — send any feedback to [support@pickit.com](mailto:support@pickit.com)*
