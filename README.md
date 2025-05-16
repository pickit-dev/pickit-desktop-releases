# Pickit Desktop

Welcome to the **Pickit Desktop** application documentation. This guide provides an overview of the app, deployment options for Windows and macOS, installer flags, required permissions, auto‑update behavior, and recommended best practices for your IT team.

## Table of Contents

1. [Introduction](#introduction)
2. [Deployment Overview](#deployment-overview)
3. [Windows Installer (NSIS)](#windows-installer-nsis)

   * [Supported Flags](#supported-flags)
   * [Usage Examples](#usage-examples)
   * [Custom Flag Implementation](#custom-flag-implementation)
4. [macOS Installation & Permissions](#macos-installation--permissions)

   * [Built‑in Entitlements](#built‑in-entitlements)
   * [Configuration Profiles (MDM)](#configuration-profiles-mdm)
   * [PPPC (Privacy Preferences)](#pppc-privacy-preferences)
5. [Auto‑Updates](#auto-updates)
6. [Appendix & References](#appendix--references)

---

## Introduction

Pickit Desktop is a cross‑platform Electron application for quickly accessing and managing digital assets on Windows and macOS. It runs in the tray (macOS) or floating window (Windows) and supports silent installation, auto‑update, drag‑and‑drop workflows, and file synchronization.

## Deployment Overview

* **Windows**: Distributed as an NSIS installer (`PickitSetup.exe`). Supports both interactive and silent modes with customizable command‑line flags.
* **macOS**: Packaged as `.zip`, and `.dmg`, hardened and notarized. Requires system entitlements and optional MDM profiles for privacy permissions.
* **Auto‑Updates**: Updates are downloaded and installed automatically in the background; no manual intervention from the end user is required. You can control update channels (stable vs. beta) via a simple setting in the application UI.

---

## Windows Installer (NSIS)

Pickit Desktop’s Windows installer is built with NSIS. You can control installation behavior via command‑line flags.

### Supported Flags

| Flag                       | Description                                                                    | Example                                               |
| -------------------------- | ------------------------------------------------------------------------------ | ----------------------------------------------------- |
| `/S`                       | **Silent install**—runs without any UI (must be the first flag)                | `PickitSetup.exe /S`                                  |
| `/D=<install_dir>`         | **Target directory** (only valid in silent mode; must be last flag, no quotes) | `PickitSetup.exe /S /D=C:\Program Files\Pickit`       |
| `--no-desktop-shortcut`    | (*Custom*) skip creating a desktop shortcut                                    | `PickitSetup.exe /S --no-desktop-shortcut`            |
| `--no-start-menu-shortcut` | (*Custom*) skip creating a Start menu shortcut                                 | `PickitSetup.exe /S --no-start-menu-shortcut`         |
| `--no-autostart`           | (*Custom*) disable adding Pickit to Windows Startup                            | `PickitSetup.exe /S --no-autostart`                   |
| `--log=<filename>`         | (*Custom*) write an installation log to the specified file                     | `PickitSetup.exe /S --log=C:\temp\pickit-install.log` |
| `/LANG=<language_code>`    | choose UI language (if multiple languages are compiled into the installer)     | `PickitSetup.exe /S /D=C:\Pickit /LANG=sv`            |

> **Notes**
>
> * Built‑in NSIS flags: `/S`, `/D=`.
> * Custom flags (`--…`) must be parsed in your NSIS script.
> * Ensure `/D=` is placed last when used.

### Usage Examples

#### 1. Silent installation to Program Files

```powershell
Start-Process -FilePath "\\share\PickitSetup.exe" -ArgumentList "/S", "/D=C:\Program Files\Pickit" -Wait
```

#### 2. Silent install without shortcuts or autostart

```powershell
Start-Process -FilePath "\\share\PickitSetup.exe" -ArgumentList "/S", "--no-desktop-shortcut", "--no-start-menu-shortcut", "--no-autostart" -Wait
```

#### 3. Silent install with log file for troubleshooting

```powershell
Start-Process -FilePath "\\share\PickitSetup.exe" -ArgumentList "/S", "--log=C:\temp\pickit-install.log" -Wait
```

### Custom Flag Implementation

In `installer.nsi`, parse flags during `.onInit`:

```nsis
!include "LogicLib.nsh"

Var NoDesktopShortcut
Var NoStartMenuShortcut
Var NoAutostart
Var LogFile

Function .onInit
  ${DoWhile} $CMDLINE <> ""
    ; process /S and /D automatically
    ${WordFind} $CMDLINE "--no-desktop-shortcut" $R0
    ${If} $R0 <> "-1" StrCpy $NoDesktopShortcut 1 ${EndIf}
    ; repeat for other custom flags...
    StrCpy $CMDLINE ""
  ${Loop}
FunctionEnd
```

Refer to the [NSIS Command‑Line Docs](https://nsis.sourceforge.io/Docs/Chapter4.html#section4.1).

---

## macOS Installation & Permissions

On macOS, Pickit Desktop ships with built‑in entitlements. To run under strict privacy settings, deploy an MDM Configuration Profile for system‑level permissions.

### Built‑in Entitlements

Located in `entitlements.mac.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
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

These are embedded at build time—no additional configuration required for JIT or file I/O entitlements.

### Configuration Profiles (MDM)

Deploy a `.mobileconfig` with PPPC rules. Example snippet:

```xml
<dict>
  <key>PayloadContent</key>
  <array>
    <dict>
      <key>PayloadType</key>
      <string>com.apple.security.privacy</string>
      <key>Services</key>
      <dict>
        <key>SystemPolicyNetworkClient</key><true/>
        <key>SystemPolicyFilesUserSelectedReadWrite</key><true/>
      </dict>
      <key>IdentifierType</key>
      <string>bundleID</string>
      <key>Identifier</key>
      <string>com.pickit.app</string>
    </dict>
  </array>
</dict>
```

* **Deploy via MDM** alongside `.dmg`.
* **Notarization**: Ensure your MDM trusts notarized packages.

### PPPC (Privacy Preferences)

Add any additional services your workflows require (e.g., camera, microphone, screen recording) under the `Services` dict.

---

## Auto‑Updates

Pickit Desktop automatically checks for and installs updates in the background. Updates download and install silently; the user is prompted to restart only if required.

No additional scripts or manual steps are needed—updates just happen.

---

## Appendix & References

* NSIS command‑line: [https://nsis.sourceforge.io/Docs/Chapter4.html#section4.1](https://nsis.sourceforge.io/Docs/Chapter4.html#section4.1)
* Apple PPPC docs: [https://developer.apple.com/documentation/macos-release-notes/](https://developer.apple.com/documentation/macos-release-notes/)

---

*Document provided for IT teams and maintained externally — send any feedback to [support@pickit.com](mailto:support@pickit.com)*
](https://images.squarespace-cdn.com/content/v1/5d10c9db34a45d000113f09c/1674640497463-0J2HLD3H3VVM2CCB3K8B/320425181_947161692930968_3137701122410422743_n.jpg?format=1000w)
