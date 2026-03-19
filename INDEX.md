# Tauri 2 Claude Skill Package — Skill Index

27 skills across 5 categories for Tauri 2 desktop application development.

## tauri-core/ (3 skills)

| Skill | Description | Triggers When... |
|-------|-------------|------------------|
| tauri-core-architecture | Guides Tauri 2 application architecture including Rust backend structure, webview layer, IPC bridge model, process model, project layout, and key type hierarchy. | Creating new Tauri apps, understanding project structure, or reasoning about Tauri's component model. |
| tauri-core-config | Guides tauri.conf.json configuration including build settings, app settings, window configuration, bundle options, plugin configuration, and security settings. | Editing tauri.conf.json, configuring build options, or setting up platform-specific bundle configuration. |
| tauri-core-runtime | Guides Tauri 2 application lifecycle including Builder configuration, setup hook, AppHandle usage, Manager trait, tokio async runtime, window event callbacks, and app exit/restart patterns. | Configuring app initialization, using setup hooks, spawning background tasks, or managing app lifecycle. |

## tauri-syntax/ (8 skills)

| Skill | Description | Triggers When... |
|-------|-------------|------------------|
| tauri-syntax-commands | Guides Tauri 2 command system including #[tauri::command] macro, sync/async commands, argument types, return types, error handling with thiserror, special injected parameters, command registration, and the frontend invoke() API. | Writing Tauri commands, calling invoke(), handling IPC errors, or streaming data from Rust to JavaScript. |
| tauri-syntax-events | Guides Tauri 2 event system including emit/listen/once patterns, Emitter and Listener traits, global vs window-scoped events, event payloads with serde, frontend event API, and unlisten cleanup. | Implementing event-driven communication between Rust and JavaScript or between windows. |
| tauri-syntax-menu | Guides Tauri 2 menu and system tray including MenuBuilder, menu item types, PredefinedMenuItem, context menus, menu events, TrayIconBuilder, tray events, and JavaScript Menu/TrayIcon APIs. | Creating application menus, system tray icons, context menus, or handling menu events. |
| tauri-syntax-permissions | Guides Tauri 2 permissions and capabilities system including capability file structure, permission definitions in TOML, scope configuration, plugin permissions, custom command permissions, and remote API access. | Configuring permissions, creating capability files, setting up plugin access control, or debugging permission denied errors. |
| tauri-syntax-plugins-api | Guides using Tauri 2 official plugins including fs, dialog, http, notification, shell, clipboard, os, process, updater, store, and opener. | Using any official Tauri plugin, accessing the file system, making HTTP requests, showing dialogs, or resolving app directories. |
| tauri-syntax-state | Guides Tauri 2 state management including app.manage(), State<T> injection in commands, AppHandle.state() access, thread-safe state with Mutex/RwLock, and state initialization patterns. | Managing application state, sharing data between commands, or debugging state-related panics. |
| tauri-syntax-webview | Guides Tauri 2 Webview API including multi-webview per window, getCurrentWebview(), webview sizing and positioning, reparenting between windows, webview events, and zoom control. | Working with multiple webviews, embedding web content, or managing webview lifecycle. |
| tauri-syntax-window | Guides Tauri 2 window management including WebviewWindowBuilder, window labels, window configuration, WindowEvent handling, JavaScript Window class methods, and monitor information queries. | Creating windows, handling window events, or managing multi-window layouts. |

## tauri-impl/ (10 skills)

| Skill | Description | Triggers When... |
|-------|-------------|------------------|
| tauri-impl-build-deploy | Guides Tauri 2 build and deployment including tauri build command, platform-specific bundlers, resource bundling, external binaries (sidecars), code signing, auto-updater setup, and CI/CD with GitHub Actions. | Building for production, configuring installers, setting up code signing, or creating CI/CD pipelines. |
| tauri-impl-database | Guides database integration in Tauri 2 including SQLite via tauri-plugin-sql, persistent key-value storage via tauri-plugin-store, and custom database integration with sqlx/diesel/rusqlite. | Adding database storage, implementing key-value persistence, or integrating SQL databases in Tauri apps. |
| tauri-impl-design-patterns | Guides architectural design patterns for Tauri 2 including when to use commands vs events vs channels, state architecture, frontend-backend responsibility split, offline-first patterns, and common application archetypes. | Designing Tauri app architecture, choosing between IPC approaches, or planning application structure. |
| tauri-impl-migration | Guides Tauri 1.x to 2.x migration including config restructuring, allowlist to permissions conversion, Rust API renames, JavaScript API renames, event system changes, and plugin migration table. | Migrating from Tauri 1 to Tauri 2, updating legacy Tauri code, or comparing v1 vs v2 APIs. |
| tauri-impl-mobile | Guides Tauri 2 mobile development including Android and iOS target setup, platform-specific Rust code with cfg attributes, lib.rs restructuring, mobile entry point, and Cargo.toml crate-type configuration. | Targeting Android or iOS, writing platform-specific code, or setting up mobile development environment. |
| tauri-impl-multi-window | Guides multi-window patterns in Tauri 2 including creating windows from Rust and JavaScript, inter-window communication via events, show/hide patterns, splashscreen implementation, and parent-child relationships. | Creating secondary windows, implementing splashscreen flows, or communicating between windows. |
| tauri-impl-plugin-development | Guides creating custom Tauri 2 plugins including plugin::Builder API, plugin commands and state, lifecycle hooks, mobile plugin development, plugin permissions, and configuration patterns. | Building custom Tauri plugins, adding lifecycle hooks, or defining plugin permissions. |
| tauri-impl-project-setup | Guides creating new Tauri 2 projects including create-tauri-app scaffolding, project structure, frontend framework integration, src-tauri directory layout, lib.rs and main.rs split pattern, and source control best practices. | Starting a new Tauri project, choosing a frontend framework, or understanding project structure. |
| tauri-impl-security | Guides Tauri 2 security implementation including CSP configuration, Tauri-specific protocols, freezePrototype, isolation pattern, scope-based access control, and dangerous permissions auditing. | Hardening Tauri app security, configuring CSP, reviewing permissions, or implementing isolation patterns. |
| tauri-impl-testing | Guides testing strategies for Tauri 2 including Rust unit testing of commands, frontend IPC mocking with mockIPC and mockWindows, WebDriver-based E2E testing patterns, and integration testing. | Writing tests for Tauri commands, mocking IPC calls, or setting up E2E test suites. |

## tauri-errors/ (4 skills)

| Skill | Description | Triggers When... |
|-------|-------------|------------------|
| tauri-errors-build | Guides debugging and resolving Tauri 2 build errors including Cargo compilation failures, bundler errors, code signing failures, missing system dependencies, mobile build issues, and CI/CD build failures. | Encountering build errors, bundler failures, or platform-specific compilation issues. |
| tauri-errors-ipc | Guides debugging and resolving Tauri 2 IPC errors including serialization failures, command not found, argument type mismatches, permission denied, async command panics, and frontend error handling. | Encountering invoke errors, serialization failures, or IPC-related panics. |
| tauri-errors-permissions | Guides debugging and resolving Tauri 2 permission errors including capability misconfiguration, missing plugin permissions, scope violations, CSP violations, and step-by-step debugging workflows. | Encountering permission denied errors, CSP violations, or capability configuration issues. |
| tauri-errors-runtime | Guides debugging and resolving Tauri 2 runtime errors including window not found, state not managed panics, plugin not initialized, asset resolution failures, event name validation panics, and webview crashes. | Encountering runtime panics, state errors, or unexpected app crashes. |

## tauri-agents/ (2 skills)

| Skill | Description | Triggers When... |
|-------|-------------|------------------|
| tauri-agents-project-scaffolder | Generates complete Tauri 2 project structures including configured plugins, permission capability files, initial Rust commands with matching TypeScript invoke calls, build pipeline configuration, and frontend framework integration. | Scaffolding a new Tauri project, setting up initial project structure, or generating boilerplate code. |
| tauri-agents-review | Provides a comprehensive validation checklist for reviewing Tauri 2 code including command signature correctness, permission coverage verification, state management patterns, error handling completeness, IPC bridge completeness, security audit, and anti-pattern detection. | Reviewing Tauri code, auditing permissions, or validating a Tauri project before deployment. |

---

## Installatie

### Claude Code

Kopieer de `skills/source/` directory naar je workspace:

```bash
# Optie 1: Clone het volledige pakket
git clone https://github.com/OpenAEC-Foundation/Tauri-2-Claude-Skill-Package.git
cp -r Tauri-2-Claude-Skill-Package/skills/source/ ~/.claude/skills/tauri/

# Optie 2: Voeg toe als git submodule
git submodule add https://github.com/OpenAEC-Foundation/Tauri-2-Claude-Skill-Package.git .claude/skills/tauri
```

### Claude.ai (Web)

Upload individuele SKILL.md bestanden als project knowledge.
