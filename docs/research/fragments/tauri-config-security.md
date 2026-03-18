# Tauri 2 Configuration, Security, and Build System — Research Document

> **Research date**: 2026-03-18
> **Sources**: Official Tauri v2 documentation (v2.tauri.app), GitHub repositories
> **Scope**: Configuration, permissions, security, build system, mobile targets, migration, CI/CD

---

## Table of Contents

1. [tauri.conf.json Structure](#1-tauriconfjson-structure)
2. [Permissions System](#2-permissions-system-new-in-v2)
3. [Capabilities System](#3-capabilities-system)
4. [Content Security Policy (CSP)](#4-content-security-policy-csp)
5. [Build System](#5-build-system)
6. [Code Signing](#6-code-signing)
7. [Auto-Updater](#7-auto-updater)
8. [Mobile Targets](#8-mobile-targets)
9. [Project Structure](#9-project-structure)
10. [Migration from Tauri 1 to 2](#10-migration-from-tauri-1-to-2)
11. [CI/CD Patterns](#11-cicd-patterns)
12. [Official Plugins Reference](#12-official-plugins-reference)
13. [Common Mistakes and Anti-Patterns](#13-common-mistakes-and-anti-patterns)

---

## 1. tauri.conf.json Structure

The main configuration file lives at `src-tauri/tauri.conf.json`. Tauri generates JSON schemas in `gen/schemas/` for IDE autocompletion.

### 1.1 Complete Example

```json
{
  "$schema": "./gen/schemas/desktop-schema.json",
  "productName": "My App",
  "version": "0.1.0",
  "identifier": "com.example.myapp",
  "build": {
    "devUrl": "http://localhost:5173",
    "frontendDist": "../dist",
    "beforeDevCommand": "npm run dev",
    "beforeBuildCommand": "npm run build",
    "beforeBundleCommand": "",
    "features": [],
    "additionalWatchFolders": []
  },
  "app": {
    "windows": [
      {
        "label": "main",
        "title": "My App",
        "url": "/",
        "width": 800,
        "height": 600,
        "minWidth": 400,
        "minHeight": 300,
        "resizable": true,
        "fullscreen": false,
        "decorations": true,
        "transparent": false,
        "alwaysOnTop": false,
        "focus": true,
        "create": true
      }
    ],
    "security": {
      "csp": null,
      "capabilities": [],
      "freezePrototype": false,
      "dangerousDisableAssetCspModification": false,
      "assetProtocol": {
        "enable": false,
        "scope": []
      },
      "pattern": {
        "use": "brownfield"
      }
    },
    "withGlobalTauri": false,
    "trayIcon": null,
    "enableGTKAppId": false,
    "macOSPrivateApi": false
  },
  "bundle": {
    "active": true,
    "targets": "all",
    "icon": [
      "icons/32x32.png",
      "icons/128x128.png",
      "icons/128x128@2x.png",
      "icons/icon.icns",
      "icons/icon.ico"
    ],
    "identifier": "com.example.myapp",
    "resources": {},
    "externalBin": [],
    "category": "Utility",
    "copyright": "",
    "shortDescription": "",
    "longDescription": "",
    "publisher": "",
    "license": "",
    "licenseFile": "",
    "createUpdaterArtifacts": false,
    "fileAssociations": [],
    "windows": {
      "certificateThumbprint": null,
      "digestAlgorithm": "sha256",
      "timestampUrl": "",
      "webviewInstallMode": "downloadBootstrapper",
      "allowDowngrades": false,
      "nsis": {},
      "wix": {}
    },
    "macOS": {
      "dmg": {},
      "hardenedRuntime": true,
      "minimumSystemVersion": "10.13",
      "signingIdentity": null
    },
    "linux": {
      "deb": {},
      "rpm": {},
      "appimage": {
        "bundleMediaFramework": false
      }
    },
    "iOS": {
      "minimumSystemVersion": "14.0"
    },
    "android": {
      "minSdkVersion": 24
    }
  },
  "plugins": {}
}
```

### 1.2 Top-Level Keys

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `identifier` | `string` | **Yes** | Reverse domain notation (e.g., `com.tauri.example`). Must be unique, alphanumeric + hyphens + periods only. |
| `productName` | `string \| null` | No | Display name of the application. |
| `version` | `string \| null` | No | Semver version or path to `package.json`. Falls back to `Cargo.toml` version. |
| `mainBinaryName` | `string \| null` | No | Override the compiled binary filename (no extension). |
| `build` | `object` | No | Build configuration. |
| `app` | `object` | No | Application runtime configuration. |
| `bundle` | `object` | No | Bundling and distribution configuration. |
| `plugins` | `object` | No | Per-plugin configuration. Default: `{}`. |

### 1.3 Build Section

| Property | Type | Description |
|----------|------|-------------|
| `devUrl` | `string \| null` | Dev server URL (e.g., `http://localhost:5173`). Used during `tauri dev`. |
| `frontendDist` | `string \| array` | Path to compiled frontend assets (e.g., `../dist`). Can be a custom protocol URL. |
| `beforeDevCommand` | `string \| object` | Shell command before `tauri dev`. Object form: `{ "script": "...", "cwd": "...", "wait": true }`. |
| `beforeBuildCommand` | `string \| object` | Shell command before `tauri build`. |
| `beforeBundleCommand` | `string \| object` | Shell command before the bundling phase. |
| `features` | `string[] \| null` | Cargo feature flags to enable. |
| `additionalWatchFolders` | `string[]` | Extra paths to watch during `tauri dev`. Default: `[]`. |
| `removeUnusedCommands` | `boolean` | Remove unused commands during build based on ACL config. |

### 1.4 App Section

| Property | Type | Description |
|----------|------|-------------|
| `windows` | `WindowConfig[]` | Windows created at startup. Default: `[]`. |
| `security` | `object` | Security settings (CSP, capabilities, asset protocol). |
| `trayIcon` | `TrayIconConfig \| null` | System tray icon configuration. |
| `withGlobalTauri` | `boolean` | Inject Tauri API on `window.__TAURI__`. |
| `enableGTKAppId` | `boolean` | Set identifier as GTK app ID. |
| `macOSPrivateApi` | `boolean` | Enable transparent background and fullscreen on macOS. |

### 1.5 Window Configuration (`app.windows[]`)

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `label` | `string` | `"main"` | Unique window identifier |
| `title` | `string` | — | Window title bar text |
| `url` | `string` | `"/"` | URL or path to load |
| `width` | `number` | — | Width in logical pixels |
| `height` | `number` | — | Height in logical pixels |
| `x` / `y` | `number` | — | Window position |
| `minWidth` / `minHeight` | `number` | — | Minimum dimensions |
| `maxWidth` / `maxHeight` | `number` | — | Maximum dimensions |
| `resizable` | `boolean` | `true` | Allow resizing |
| `fullscreen` | `boolean` | `false` | Start fullscreen |
| `focus` | `boolean` | `true` | Focus on creation |
| `create` | `boolean` | `true` | Create at startup (set `false` for programmatic creation) |
| `transparent` | `boolean` | `false` | Enable transparency |
| `decorations` | `boolean` | `true` | Show title bar / borders |
| `alwaysOnTop` | `boolean` | `false` | Stay above other windows |

### 1.6 Bundle Section

| Property | Type | Description |
|----------|------|-------------|
| `active` | `boolean` | Enable/disable bundling. |
| `targets` | `string \| string[]` | `"all"`, `"linux"`, `"windows"`, `"macos"`, `"android"`, `"ios"`. |
| `icon` | `string[]` | Icon file paths. |
| `resources` | `object` | Additional files/directories to bundle. |
| `externalBin` | `string[]` | External binaries to embed as sidecars. |
| `fileAssociations` | `array` | File type associations. |
| `category` | `string` | App category (e.g., `"Utility"`, `"Productivity"`). |
| `copyright` | `string` | Copyright notice. |
| `publisher` | `string` | Publisher name. |
| `license` | `string` | SPDX license identifier. |
| `licenseFile` | `string` | Path to license file. |
| `createUpdaterArtifacts` | `boolean \| "v1Compatible"` | Generate update signatures. |

### 1.7 Platform-Specific Bundle Configuration

**Windows** (`bundle.windows`):
- `nsis`: NSIS installer configuration
- `wix`: WiX (MSI) configuration
- `webviewInstallMode`: `"downloadBootstrapper"` | `"offlineInstallerFromUrl"`
- `signCommand`: Custom signing command (use `%1` for file path placeholder)
- `certificateThumbprint`: Certificate thumbprint for Authenticode
- `digestAlgorithm`: Hash algorithm (typically `"sha256"`)
- `timestampUrl`: Timestamp server URL
- `allowDowngrades`: Permit version downgrades

**macOS** (`bundle.macOS`):
- `dmg`: DMG configuration (window size, positions)
- `hardenedRuntime`: Enable hardened runtime
- `minimumSystemVersion`: Minimum macOS version
- `signingIdentity`: Code signing identity

**Linux** (`bundle.linux`):
- `deb`: Debian package config
- `rpm`: RPM package config
- `appimage`: AppImage config with `bundleMediaFramework`

**iOS** (`bundle.iOS`):
- `minimumSystemVersion`: Default `"14.0"`

**Android** (`bundle.android`):
- `minSdkVersion`: Default `24`
- `versionCode`: Auto-calculated from semver
- `autoIncrementVersionCode`: Increment on each build

---

## 2. Permissions System (NEW IN V2)

The permissions system replaces the Tauri v1 allowlist. It provides fine-grained control over which IPC commands the frontend can invoke.

### 2.1 Core Concepts

- **Permissions** define which commands are allowed or denied
- **Scopes** restrict *what data* those commands can access (e.g., file paths, URLs)
- **Capabilities** group permissions and assign them to specific windows/webviews

### 2.2 Permission Format

Permissions follow the naming convention:
- `<plugin-name>:default` — Default permission set for a plugin
- `<plugin-name>:allow-<command>` — Allow a specific command
- `<plugin-name>:deny-<command>` — Deny a specific command
- `<plugin-name>:<custom-set>` — Custom permission set

The system auto-prepends `tauri-plugin-` to plugin identifiers at compile time. Identifiers are limited to lowercase ASCII `[a-z]`, max 116 characters.

### 2.3 Defining Permissions (TOML format)

Plugin and application permissions live in `src-tauri/permissions/<identifier>.toml`:

```toml
# Basic command permission
[[permission]]
identifier = "allow-read-file"
description = "Enables the read_file command"
commands.allow = ["read_file"]

# Permission with scope
[[permission]]
identifier = "scope-home"
description = "Access to files in $HOME"

[[scope.allow]]
path = "$HOME/*"

[[scope.deny]]
path = "$HOME/.ssh/*"
```

### 2.4 Permission Sets

Bundle multiple permissions under a single identifier:

```toml
[[set]]
identifier = "allow-home-read-extended"
description = "Read access + directory creation in $HOME"
permissions = [
    "fs:read-files",
    "fs:scope-home",
    "fs:allow-mkdir"
]
```

### 2.5 Custom Command Permissions

For your own `#[tauri::command]` functions, create permission files in `src-tauri/permissions/`:

```toml
# src-tauri/permissions/my-commands.toml
[[permission]]
identifier = "allow-my-command"
description = "Allows invoking the my_command function"
commands.allow = ["my_command"]

[[permission]]
identifier = "deny-my-command"
description = "Denies the my_command function"
commands.deny = ["my_command"]
```

Then reference them in capabilities without a plugin prefix:

```json
{
  "permissions": ["allow-my-command"]
}
```

### 2.6 File Structure

```
tauri-plugin/
├── permissions/
│   ├── <identifier>.json/toml    # Custom permission definitions
│   └── default.json/toml         # Default permissions for the plugin

tauri-app/src-tauri/
├── permissions/
│   └── <identifier>.toml         # App-level permissions (TOML only)
├── capabilities/
│   └── <identifier>.json/.toml   # Capability definitions
```

**Important**: Application developers write permissions in TOML format only. Capabilities support both JSON and TOML.

---

## 3. Capabilities System

Capabilities tie permissions to specific windows and webviews, forming the access control bridge.

### 3.1 Capability File Structure

Files in `src-tauri/capabilities/` are automatically enabled unless configured otherwise in `tauri.conf.json`.

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "main-capability",
  "description": "Capability for the main window",
  "windows": ["main"],
  "permissions": [
    "core:path:default",
    "core:event:default",
    "core:window:default",
    "core:app:default",
    "core:resources:default",
    "core:menu:default",
    "core:tray:default",
    "core:window:allow-set-title"
  ]
}
```

### 3.2 Key Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `identifier` | `string` | Yes | Unique capability name |
| `description` | `string` | No | Purpose description |
| `windows` | `string[]` | Yes | Window labels to apply to (supports `"*"` wildcard) |
| `permissions` | `string[]` | Yes | Array of permission identifiers |
| `platforms` | `string[]` | No | Target OS: `"linux"`, `"macOS"`, `"windows"`, `"iOS"`, `"android"` |
| `remote` | `object` | No | Remote URL access configuration |

### 3.3 Platform-Specific Capabilities

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "desktop-capability",
  "windows": ["main"],
  "platforms": ["linux", "macOS", "windows"],
  "permissions": ["global-shortcut:allow-register"]
}
```

```json
{
  "$schema": "../gen/schemas/mobile-schema.json",
  "identifier": "mobile-capability",
  "windows": ["main"],
  "platforms": ["iOS", "android"],
  "permissions": [
    "nfc:allow-scan",
    "biometric:allow-authenticate",
    "barcode-scanner:allow-scan"
  ]
}
```

### 3.4 Remote API Access

Grant Tauri API access to remote URLs (useful for hybrid apps):

```json
{
  "$schema": "../gen/schemas/remote-schema.json",
  "identifier": "remote-tag-capability",
  "windows": ["main"],
  "remote": {
    "urls": ["https://*.tauri.app"]
  },
  "platforms": ["iOS", "android"],
  "permissions": ["nfc:allow-scan", "barcode-scanner:allow-scan"]
}
```

### 3.5 Inline Capabilities in tauri.conf.json

Capabilities can be defined inline or referenced by identifier:

```json
{
  "app": {
    "security": {
      "capabilities": [
        {
          "identifier": "inline-capability",
          "windows": ["*"],
          "permissions": ["fs:default"]
        },
        "my-external-capability"
      ]
    }
  }
}
```

### 3.6 Window Merging Behavior

Windows that appear in multiple capabilities merge all permissions from all involved capabilities. This is additive — there is no override or conflict resolution.

### 3.7 Schema Generation

Tauri generates platform-specific JSON schemas in `gen/schemas/`:
- `desktop-schema.json`
- `mobile-schema.json`
- `remote-schema.json`

Reference them via `$schema` for IDE autocompletion.

---

## 4. Content Security Policy (CSP)

### 4.1 Overview

Tauri applies a Content Security Policy to restrict resource loading in the webview. The CSP is configured in `app.security.csp`.

### 4.2 Configuration

```json
{
  "app": {
    "security": {
      "csp": "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' asset: https://asset.localhost; connect-src ipc: http://ipc.localhost"
    }
  }
}
```

The CSP can also be set to `null` to disable it entirely (not recommended for production).

### 4.3 Tauri-Specific Protocols

| Protocol | Purpose | CSP Usage |
|----------|---------|-----------|
| `tauri:` | Internal Tauri protocol for serving frontend assets | Used in `default-src` |
| `asset:` / `https://asset.localhost` | Access bundled resources and filesystem assets | Used in `img-src`, `media-src` |
| `ipc:` / `http://ipc.localhost` | IPC communication channel between frontend and Rust | Used in `connect-src` |

### 4.4 Security Settings

```json
{
  "app": {
    "security": {
      "csp": "default-src 'self'",
      "freezePrototype": true,
      "dangerousDisableAssetCspModification": false,
      "assetProtocol": {
        "enable": true,
        "scope": ["$APPDATA/**", "$RESOURCE/**"]
      },
      "pattern": {
        "use": "brownfield"
      }
    }
  }
}
```

| Property | Description |
|----------|-------------|
| `freezePrototype` | Prevents JavaScript prototype pollution attacks by freezing `Object.prototype` |
| `dangerousDisableAssetCspModification` | Disables automatic CSP nonce injection for the asset protocol. **Dangerous**: only use if you understand the implications. |
| `assetProtocol.enable` | Enable the `asset:` protocol for file access |
| `assetProtocol.scope` | Glob patterns defining allowed asset paths |
| `pattern.use` | Security pattern: `"brownfield"` (default) or isolation patterns |

### 4.5 Common CSP Directives for Tauri

```
default-src 'self'
script-src 'self'
style-src 'self' 'unsafe-inline'
img-src 'self' asset: https://asset.localhost data:
font-src 'self' data:
connect-src ipc: http://ipc.localhost https://api.example.com
media-src 'self' asset: https://asset.localhost
```

### 4.6 Windows URL Change (v2)

On Windows, production frontend files now load from `http://tauri.localhost` instead of `https://tauri.localhost`. This change resets IndexedDB, LocalStorage, and Cookies unless:
- `dangerousUseHttpScheme` was enabled in v1
- `app.windows[].useHttpsScheme` is set to `true` in v2

---

## 5. Build System

### 5.1 Build Commands

```bash
# Standard desktop build
tauri build

# Build without bundling
tauri build --no-bundle

# Bundle specific formats only
tauri bundle --bundles app,dmg

# Use alternate config
tauri bundle --bundles app --config src-tauri/tauri.appstore.conf.json

# Mobile builds
tauri android build
tauri ios build
```

### 5.2 Platform-Specific Bundlers

| Platform | Formats | Notes |
|----------|---------|-------|
| **Windows** | NSIS installer, MSI (WiX), portable exe | WebView2 installation mode configurable |
| **macOS** | App bundle (.app), DMG | Code signing + notarization required for distribution |
| **Linux** | AppImage, .deb, .rpm, Snap, Flatpak, AUR | AppImage is the most universal |
| **Android** | APK, AAB | Play Store requires AAB |
| **iOS** | IPA | App Store or TestFlight distribution |

### 5.3 Resource Bundling

Include additional files in the application bundle:

```json
{
  "bundle": {
    "resources": {
      "models/": "models/",
      "config.toml": "config.toml"
    },
    "externalBin": [
      "binaries/ffmpeg"
    ]
  }
}
```

External binaries are embedded as "sidecars" and can be spawned at runtime.

---

## 6. Code Signing

### 6.1 Windows (Authenticode)

**OV Certificate configuration in tauri.conf.json:**

```json
{
  "bundle": {
    "windows": {
      "certificateThumbprint": "A1B1A2B2A3B3A4B4A5B5A6B6A7B7A8B8A9B9A0B0",
      "digestAlgorithm": "sha256",
      "timestampUrl": "http://timestamp.comodoca.com"
    }
  }
}
```

**Custom signing (Azure Key Vault, etc.):**

```json
{
  "bundle": {
    "windows": {
      "signCommand": "relic sign --file %1 --key azure --config relic.conf"
    }
  }
}
```

**GitHub Actions setup** requires secrets:
- `WINDOWS_CERTIFICATE` — Base64-encoded `.pfx` file
- `WINDOWS_CERTIFICATE_PASSWORD` — Export password

### 6.2 macOS (Apple Developer ID + Notarization)

**Local signing:**

```json
{
  "bundle": {
    "macOS": {
      "signingIdentity": "Developer ID Application: Your Name (TEAMID)"
    }
  }
}
```

Or use `APPLE_SIGNING_IDENTITY` environment variable.

**Notarization environment variables:**

| Method | Variables |
|--------|-----------|
| App Store Connect API | `APPLE_API_ISSUER`, `APPLE_API_KEY`, `APPLE_API_KEY_PATH` |
| Apple ID | `APPLE_ID`, `APPLE_PASSWORD` (app-specific), `APPLE_TEAM_ID` |

**CI/CD signing** requires:
- `APPLE_CERTIFICATE` — Base64-encoded `.p12`
- `APPLE_CERTIFICATE_PASSWORD`
- `APPLE_SIGNING_IDENTITY`
- `KEYCHAIN_PASSWORD`

**Ad-hoc signing** (no Apple account, for testing):

```json
{
  "bundle": {
    "macOS": {
      "signingIdentity": "-"
    }
  }
}
```

### 6.3 Linux

GPG signatures for packages. Platform-specific package managers handle verification.

---

## 7. Auto-Updater

### 7.1 Setup

Install the updater plugin:

```bash
npm run tauri add updater
cargo add tauri-plugin-updater --target 'cfg(any(target_os = "macos", windows, target_os = "linux"))'
```

Initialize in Rust:

```rust
#[cfg(desktop)]
app.handle().plugin(tauri_plugin_updater::Builder::new().build());
```

### 7.2 Key Generation

```bash
npm run tauri signer generate -- -w ~/.tauri/myapp.key
```

This creates a public/private keypair. **Never share the private key.** Losing it prevents all future updates to existing installations.

Set build environment variables:

```bash
export TAURI_SIGNING_PRIVATE_KEY="path or content"
export TAURI_SIGNING_PRIVATE_KEY_PASSWORD=""
```

### 7.3 Configuration

```json
{
  "bundle": {
    "createUpdaterArtifacts": true
  },
  "plugins": {
    "updater": {
      "pubkey": "dW50cnVzdGVkIGNvbW1lbnQ6...",
      "endpoints": [
        "https://releases.myapp.com/{{target}}/{{arch}}/{{current_version}}"
      ],
      "windows": {
        "installMode": "passive"
      }
    }
  }
}
```

**URL template variables:**
- `{{current_version}}` — Current app version
- `{{target}}` — OS (`linux`, `windows`, `darwin`)
- `{{arch}}` — Architecture (`x86_64`, `i686`, `aarch64`, `armv7`)

**Windows install modes:**
- `"passive"` — No user interaction (default)
- `"basicUi"` — Requires user interaction
- `"quiet"` — No feedback at all

### 7.4 Server Response Format

**Static JSON (for CDN/GitHub Releases):**

```json
{
  "version": "1.0.0",
  "notes": "Bug fixes and improvements",
  "pub_date": "2024-01-15T10:30:00Z",
  "platforms": {
    "linux-x86_64": {
      "signature": "...",
      "url": "https://cdn.example.com/app-1.0.0.AppImage.tar.gz"
    },
    "windows-x86_64": {
      "signature": "...",
      "url": "https://cdn.example.com/app-1.0.0-setup.exe"
    },
    "darwin-x86_64": {
      "signature": "...",
      "url": "https://cdn.example.com/app-1.0.0.app.tar.gz"
    },
    "darwin-aarch64": {
      "signature": "...",
      "url": "https://cdn.example.com/app-1.0.0.app.tar.gz"
    }
  }
}
```

**Dynamic server**: Return HTTP 204 (no update) or HTTP 200 with:

```json
{
  "version": "2.0.0",
  "url": "https://cdn.example.com/update.zip",
  "signature": "...",
  "pub_date": "2024-01-20T08:00:00Z",
  "notes": "New features"
}
```

### 7.5 JavaScript Update Check

```javascript
import { check } from '@tauri-apps/plugin-updater';
import { relaunch } from '@tauri-apps/plugin-process';

const update = await check();
if (update) {
  await update.downloadAndInstall((event) => {
    switch (event.event) {
      case 'Started':
        console.log(`Downloading: ${event.data.contentLength} bytes`);
        break;
      case 'Progress':
        console.log(`Downloaded: ${event.data.chunkLength}`);
        break;
      case 'Finished':
        console.log('Download complete');
        break;
    }
  });
  await relaunch();
}
```

### 7.6 Rust Update Check

```rust
use tauri_plugin_updater::UpdaterExt;

async fn update(app: tauri::AppHandle) -> tauri_plugin_updater::Result<()> {
    if let Some(update) = app.updater()?.check().await? {
        update.download_and_install(
            |chunk_length, content_length| {
                println!("Downloaded: {chunk_length}/{content_length:?}");
            },
            || println!("Finished"),
        ).await?;
        app.restart();
    }
    Ok(())
}
```

### 7.7 Generated Artifacts

| Platform | Update Artifact | Signature |
|----------|----------------|-----------|
| Linux | `.AppImage.tar.gz` | `.AppImage.tar.gz.sig` |
| macOS | `.app.tar.gz` | `.app.tar.gz.sig` |
| Windows | `.exe` or `.msi` | `.exe.sig` or `.msi.sig` |

### 7.8 Capability Permission

```json
{
  "permissions": ["updater:default"]
}
```

Individual permissions: `updater:allow-check`, `updater:allow-download`, `updater:allow-install`, `updater:allow-download-and-install`.

---

## 8. Mobile Targets

### 8.1 Prerequisites

**Android:**
- Android Studio with SDK Platform, Platform-Tools, NDK, Build-Tools, Command-line Tools
- Environment variables: `JAVA_HOME`, `ANDROID_HOME`, `NDK_HOME`
- Rust targets: `aarch64-linux-android`, `armv7-linux-androideabi`, `i686-linux-android`, `x86_64-linux-android`

**iOS (macOS only):**
- Full Xcode installation (not just Command Line Tools)
- Cocoapods (via Homebrew)
- Rust targets: `aarch64-apple-ios`, `x86_64-apple-ios`, `aarch64-apple-ios-sim`

### 8.2 Commands

```bash
# Initialize mobile projects
tauri android init
tauri ios init

# Development
tauri android dev
tauri ios dev

# Production builds
tauri android build
tauri ios build
```

### 8.3 Cargo.toml for Mobile Support

```toml
[lib]
name = "app_lib"
crate-type = ["staticlib", "cdylib", "rlib"]
```

### 8.4 Entry Point for Mobile

Rename `src-tauri/src/main.rs` to `lib.rs` and restructure:

```rust
// src-tauri/src/lib.rs
#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

```rust
// src-tauri/src/main.rs (new, minimal)
fn main() {
    app_lib::run();
}
```

### 8.5 Platform-Specific Rust Code

```rust
#[cfg(target_os = "android")]
fn android_specific() { /* ... */ }

#[cfg(target_os = "ios")]
fn ios_specific() { /* ... */ }

#[cfg(desktop)]
fn desktop_only() { /* ... */ }

#[cfg(mobile)]
fn mobile_only() { /* ... */ }
```

### 8.6 Bundle Configuration

```json
{
  "bundle": {
    "iOS": {
      "minimumSystemVersion": "14.0"
    },
    "android": {
      "minSdkVersion": 24,
      "versionCode": 1
    }
  }
}
```

---

## 9. Project Structure

### 9.1 `create-tauri-app` Output

```
my-tauri-app/
├── src/                        # Frontend source (varies by framework)
│   ├── index.html
│   ├── main.ts
│   └── styles.css
├── src-tauri/                  # Rust backend
│   ├── src/
│   │   ├── lib.rs              # Main app logic (entry point for mobile)
│   │   └── main.rs             # Desktop entry point (calls lib.rs)
│   ├── capabilities/           # Permission capability files
│   │   └── default.json        # Default capabilities
│   ├── icons/                  # Application icons (all sizes)
│   │   ├── icon.ico
│   │   ├── icon.icns
│   │   ├── 32x32.png
│   │   ├── 128x128.png
│   │   └── 128x128@2x.png
│   ├── gen/                    # Generated files (schemas, etc.)
│   │   └── schemas/
│   │       ├── desktop-schema.json
│   │       ├── mobile-schema.json
│   │       └── remote-schema.json
│   ├── permissions/            # Custom command permissions (optional)
│   ├── Cargo.toml              # Rust dependencies
│   ├── Cargo.lock              # Deterministic build lock (commit this!)
│   ├── tauri.conf.json         # Main configuration
│   ├── build.rs                # Cargo build script
│   └── .taurignore             # Exclude files from watch
├── package.json                # Frontend dependencies
├── tsconfig.json               # TypeScript config (if applicable)
└── vite.config.ts              # Bundler config (if Vite)
```

### 9.2 Key package.json Dependencies

```json
{
  "devDependencies": {
    "@tauri-apps/cli": "^2"
  },
  "dependencies": {
    "@tauri-apps/api": "^2",
    "@tauri-apps/plugin-shell": "^2",
    "@tauri-apps/plugin-fs": "^2"
  }
}
```

### 9.3 Source Control

- **Commit**: `src-tauri/Cargo.lock` (deterministic builds)
- **Ignore**: `src-tauri/target/` (build artifacts)

---

## 10. Migration from Tauri 1 to 2

### 10.1 Configuration Restructuring

| Tauri v1 | Tauri v2 |
|----------|----------|
| `package.productName` | `productName` (top-level) |
| `package.version` | `version` (top-level) |
| `tauri` | `app` |
| `tauri.allowlist` | Removed — replaced by capabilities/permissions |
| `tauri.bundle` | `bundle` (top-level) |
| `tauri.cli` | `plugins.cli` |
| `tauri.updater` | `plugins.updater` |
| `tauri.systemTray` | `app.trayIcon` |
| `tauri.allowlist.protocol.assetScope` | `app.security.assetProtocol.scope` |
| `tauri.pattern` | `app.security.pattern` |
| `build.distDir` | `build.frontendDist` |
| `build.devPath` | `build.devUrl` |
| `build.withGlobalTauri` | `app.withGlobalTauri` |
| `tauri.windows[].fileDropEnabled` | `app.windows[].dragDropEnabled` |

### 10.2 Allowlist to Permissions Migration

The v1 allowlist is completely replaced by the capability-based permission system. Use the migration CLI:

```bash
npx @tauri-apps/cli migrate
```

This auto-converts v1 allowlist entries to capability files in `src-tauri/capabilities/`.

### 10.3 Rust API Changes

**Type renames:**
- `Window` → `WebviewWindow`
- `WindowBuilder` → `WebviewWindowBuilder`
- `WindowUrl` → `WebviewUrl`
- `Manager::get_window()` → `Manager::get_webview_window()`

**Removed modules** (now plugins):
- `api::dialog` → `tauri-plugin-dialog`
- `api::http` → `tauri-plugin-http`
- `api::path` → `Manager::path()` methods
- `api::process` → `tauri-plugin-process`
- `api::shell` → `tauri-plugin-shell`

**Removed manager methods:**
- `App::clipboard_manager()` → `tauri-plugin-clipboard-manager`
- `App::get_cli_matches()` → `tauri-plugin-cli`
- `App::global_shortcut_manager()` → `tauri-plugin-global-shortcut`

### 10.4 JavaScript API Changes

**Module renames:**
- `@tauri-apps/api/tauri` → `@tauri-apps/api/core`
- `@tauri-apps/api/window` → `@tauri-apps/api/webviewWindow`

**All non-core modules removed** from the base `@tauri-apps/api` package. Install per-plugin packages:
- `@tauri-apps/plugin-dialog`
- `@tauri-apps/plugin-fs`
- `@tauri-apps/plugin-http`
- `@tauri-apps/plugin-shell`
- etc.

### 10.5 Event System Changes

- `emit()` now broadcasts to all listeners
- New `emit_to()` for targeted events
- `listen_global()` renamed to `listen_any()`
- `WebviewWindow.listen()` only receives events for that specific window

### 10.6 Menu System Overhaul

- `Menu` → `MenuBuilder`
- `CustomMenuItem` → `MenuItemBuilder`
- `Submenu` → `SubmenuBuilder`
- `MenuItem` → `PredefinedMenuItem`
- `SystemTray` → `TrayIconBuilder`
- Event handling: `Builder::on_menu_event()` removed; use `App::on_menu_event()` or `AppHandle::on_menu_event()`

### 10.7 Feature Flag Changes

**Removed**: `reqwest-client`, `reqwest-native-tls-vendored`, `process-command-api`, `shell-open-api`, `windows7-compat`, `updater`, `linux-protocol-headers`, `system-tray`

**Added**: `linux-protocol-body`

### 10.8 Mobile Entry Point

```rust
#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    // app builder
}
```

### 10.9 Plugin Migration Table

| Feature | Plugin Crate | JS Package |
|---------|-------------|------------|
| Clipboard | `tauri-plugin-clipboard-manager = "2"` | `@tauri-apps/plugin-clipboard-manager` |
| Dialogs | `tauri-plugin-dialog = "2"` | `@tauri-apps/plugin-dialog` |
| File System | `tauri-plugin-fs = "2"` | `@tauri-apps/plugin-fs` |
| Global Shortcuts | `tauri-plugin-global-shortcut = "2"` | `@tauri-apps/plugin-global-shortcut` |
| HTTP | `tauri-plugin-http = "2"` | `@tauri-apps/plugin-http` |
| Notifications | `tauri-plugin-notification = "2"` | `@tauri-apps/plugin-notification` |
| OS Info | `tauri-plugin-os = "2"` | `@tauri-apps/plugin-os` |
| Process | `tauri-plugin-process = "2"` | `@tauri-apps/plugin-process` |
| Shell | `tauri-plugin-shell = "2"` | `@tauri-apps/plugin-shell` |
| CLI Parsing | `tauri-plugin-cli = "2"` | `@tauri-apps/plugin-cli` |
| Updater | `tauri-plugin-updater = "2"` | `@tauri-apps/plugin-updater` |

**Critical**: The automatic update dialog was removed in v2. You must explicitly implement update checking and installation UI.

---

## 11. CI/CD Patterns

### 11.1 GitHub Actions with `tauri-action`

The official `tauri-apps/tauri-action` automates cross-platform builds and releases.

### 11.2 Cross-Platform Build Matrix

```yaml
name: Release
on:
  push:
    branches: [release]
  workflow_dispatch:

permissions:
  contents: write

jobs:
  release:
    strategy:
      fail-fast: false
      matrix:
        include:
          - platform: macos-latest
            args: --target aarch64-apple-darwin
          - platform: macos-latest
            args: --target x86_64-apple-darwin
          - platform: ubuntu-22.04
            args: ''
          - platform: ubuntu-22.04-arm
            args: ''
          - platform: windows-latest
            args: ''

    runs-on: ${{ matrix.platform }}
    steps:
      - uses: actions/checkout@v4

      - name: Install Linux dependencies
        if: runner.os == 'Linux'
        run: |
          sudo apt-get update
          sudo apt-get install -y libwebkit2gtk-4.1-dev libappindicator3-dev librsvg2-dev

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: lts/*

      - name: Install Rust stable
        uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.platform == 'macos-latest' && 'aarch64-apple-darwin,x86_64-apple-darwin' || '' }}

      - name: Rust cache
        uses: swatinem/rust-cache@v2
        with:
          workspaces: './src-tauri -> target'

      - name: Install frontend dependencies
        run: npm install

      - name: Build Tauri app
        uses: tauri-apps/tauri-action@v0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tagName: app-v__VERSION__
          releaseName: 'App v__VERSION__'
          releaseBody: 'See the assets to download and install.'
          releaseDraft: true
          prerelease: false
          args: ${{ matrix.args }}
```

### 11.3 Key Configuration Notes

- `__VERSION__` is auto-replaced with the app version from `tauri.conf.json`
- `releaseDraft: true` creates draft releases for review before publishing
- Linux dependencies (`libwebkit2gtk-4.1-dev`, `libappindicator3-dev`, `librsvg2-dev`) must be installed explicitly
- GitHub token permissions must be set to "Read and write" in repository settings
- ARM Linux builds: use `ubuntu-22.04-arm` or `ubuntu-24.04-arm` runners (GA since August 2025)

### 11.4 Windows Code Signing in CI

```yaml
- name: Import Windows certificate
  if: matrix.platform == 'windows-latest'
  env:
    WINDOWS_CERTIFICATE: ${{ secrets.WINDOWS_CERTIFICATE }}
    WINDOWS_CERTIFICATE_PASSWORD: ${{ secrets.WINDOWS_CERTIFICATE_PASSWORD }}
  run: |
    New-Item -ItemType directory -Path certificate
    Set-Content -Path certificate/tempCert.txt -Value $env:WINDOWS_CERTIFICATE
    certutil -decode certificate/tempCert.txt certificate/certificate.pfx
    Import-PfxCertificate -FilePath certificate/certificate.pfx -CertStoreLocation Cert:\CurrentUser\My -Password (ConvertTo-SecureString -String $env:WINDOWS_CERTIFICATE_PASSWORD -Force -AsPlainText)
```

### 11.5 macOS Code Signing in CI

Required secrets:
- `APPLE_CERTIFICATE` (base64 `.p12`)
- `APPLE_CERTIFICATE_PASSWORD`
- `APPLE_SIGNING_IDENTITY`
- `KEYCHAIN_PASSWORD`
- `APPLE_API_ISSUER`, `APPLE_API_KEY`, `APPLE_API_KEY_PATH` (for notarization)

---

## 12. Official Plugins Reference

### 12.1 Complete Plugin List

| Plugin | Description | Permission Prefix |
|--------|-------------|-------------------|
| **Autostart** | Auto-launch at system startup | `autostart` |
| **Barcode Scanner** | QR/barcode scanning (mobile) | `barcode-scanner` |
| **Biometric** | Biometric auth (mobile) | `biometric` |
| **CLI** | Command-line argument parsing | `cli` |
| **Clipboard Manager** | System clipboard read/write | `clipboard-manager` |
| **Deep Linking** | URL handler registration | `deep-link` |
| **Dialog** | Native file/message dialogs | `dialog` |
| **File System** | File system access | `fs` |
| **Geolocation** | Device position tracking | `geolocation` |
| **Global Shortcut** | System-wide keyboard shortcuts | `global-shortcut` |
| **Haptics** | Vibration/haptic feedback (mobile) | `haptics` |
| **HTTP Client** | HTTP requests via Rust | `http` |
| **Localhost** | Embedded localhost server | `localhost` |
| **Log** | Configurable logging | `log` |
| **NFC** | NFC tag read/write (mobile) | `nfc` |
| **Notification** | Native system notifications | `notification` |
| **Opener** | Open files/URLs in external apps | `opener` |
| **OS** | Operating system information | `os` |
| **Persisted Scope** | Persist runtime scope changes | `persisted-scope` |
| **Positioner** | Window positioning helpers | `positioner` |
| **Process** | Current process info/control | `process` |
| **Shell** | Spawn child processes | `shell` |
| **Single Instance** | Enforce single app instance | `single-instance` |
| **SQL** | Database access via sqlx | `sql` |
| **Store** | Persistent key-value storage | `store` |
| **Stronghold** | Encrypted secure database | `stronghold` |
| **Updater** | In-app auto-updates | `updater` |
| **Upload** | HTTP file uploads | `upload` |
| **WebSocket** | WebSocket connections via Rust | `websocket` |
| **Window State** | Persist window size/position | `window-state` |

### 12.2 Core Permissions (built-in, no plugin needed)

```
core:path:default
core:event:default
core:window:default
core:app:default
core:resources:default
core:menu:default
core:tray:default
core:window:allow-set-title
core:window:allow-close
core:window:allow-minimize
core:window:allow-maximize
```

### 12.3 Permission Usage Pattern

```json
{
  "permissions": [
    "core:window:default",
    "fs:default",
    "fs:allow-read-file",
    "fs:deny-write-file",
    "dialog:default",
    "dialog:allow-open",
    "shell:default",
    "http:default",
    "updater:default",
    "clipboard-manager:allow-read",
    "clipboard-manager:allow-write",
    "notification:default",
    "process:allow-restart"
  ]
}
```

### 12.4 Typical Permission Suffixes

For each plugin, auto-generated permissions typically follow this pattern:
- `<plugin>:default` — Safe default permissions
- `<plugin>:allow-<command>` — Allow a specific command
- `<plugin>:deny-<command>` — Deny a specific command

---

## 13. Common Mistakes and Anti-Patterns

### 13.1 Security

1. **Setting CSP to `null` in production** — Disables all content security restrictions. Always define a restrictive CSP.
2. **Using `"windows": ["*"]` with broad permissions** — Grants all windows the same permissions. Use specific window labels.
3. **Using `dangerousDisableAssetCspModification: true`** — Removes automatic CSP nonce injection. Only use if you fully understand the implications.
4. **Not enabling `freezePrototype`** — Leaves your app vulnerable to prototype pollution attacks.
5. **Granting `shell:default` without scope restrictions** — Allows arbitrary command execution. Always define scopes.

### 13.2 Configuration

6. **Not setting `identifier` to a unique reverse-domain** — Causes conflicts with other apps on the system.
7. **Forgetting `beforeBuildCommand`** — Frontend assets won't be built before bundling.
8. **Using `build.devUrl` without `beforeDevCommand`** — The dev server won't start automatically.
9. **Not committing `Cargo.lock`** — Leads to non-deterministic builds across environments.

### 13.3 Migration (v1 to v2)

10. **Not running `npx @tauri-apps/cli migrate`** — Manual migration is error-prone; the CLI handles most conversions.
11. **Forgetting the updater UI** — v2 removed the automatic update dialog. You must implement update checking explicitly or users will never receive updates.
12. **Using old import paths** — `@tauri-apps/api/tauri` is now `@tauri-apps/api/core`; `@tauri-apps/api/window` is now `@tauri-apps/api/webviewWindow`.
13. **Not restructuring for mobile** — If targeting mobile, you must split `main.rs` into `lib.rs` + `main.rs` and add the `#[cfg_attr(mobile, tauri::mobile_entry_point)]` macro.

### 13.4 Build & Distribution

14. **Not signing macOS builds** — Unsigned apps are quarantined by Gatekeeper. At minimum use ad-hoc signing (`"-"`).
15. **Losing the updater private key** — Makes it impossible to push updates to existing installations. Back it up securely.
16. **Setting `createUpdaterArtifacts: true` without signing keys** — Build will fail. Set `TAURI_SIGNING_PRIVATE_KEY` environment variable.
17. **Using `"v1Compatible"` for new projects** — Only needed when migrating from v1 updater. New projects should use `true`.

### 13.5 Permissions

18. **Not adding capabilities for plugins** — Installing a plugin does not automatically grant permissions. You must add permissions to a capability file.
19. **Mixing up TOML vs JSON** — Application permissions must be TOML. Capabilities can be JSON or TOML.
20. **Forgetting `core:` prefixed permissions** — Core features (window management, events, paths) also require explicit permissions in v2.

---

## Appendix: Quick Reference

### Minimal Viable tauri.conf.json

```json
{
  "identifier": "com.example.myapp",
  "build": {
    "devUrl": "http://localhost:5173",
    "frontendDist": "../dist",
    "beforeDevCommand": "npm run dev",
    "beforeBuildCommand": "npm run build"
  },
  "app": {
    "windows": [
      {
        "title": "My App",
        "width": 800,
        "height": 600
      }
    ]
  },
  "bundle": {
    "active": true,
    "icon": [
      "icons/32x32.png",
      "icons/128x128.png",
      "icons/128x128@2x.png",
      "icons/icon.icns",
      "icons/icon.ico"
    ]
  }
}
```

### Minimal Viable Capability File

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "default",
  "description": "Default capabilities for the main window",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "core:window:default",
    "core:app:default"
  ]
}
```

### Custom Command — Full Flow

1. Define the command in Rust:
```rust
#[tauri::command]
fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}
```

2. Register in the builder:
```rust
tauri::Builder::default()
    .invoke_handler(tauri::generate_handler![greet])
```

3. Define permission (`src-tauri/permissions/commands.toml`):
```toml
[[permission]]
identifier = "allow-greet"
description = "Allow the greet command"
commands.allow = ["greet"]
```

4. Add to capability (`src-tauri/capabilities/default.json`):
```json
{
  "identifier": "default",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "allow-greet"
  ]
}
```

5. Call from JavaScript:
```javascript
import { invoke } from '@tauri-apps/api/core';
const greeting = await invoke('greet', { name: 'World' });
```
