# Tauri 2 Claude Skill Package — Raw Masterplan

> **Fase 1 output** — Voorlopige skill-inventaris met scope-definities.
> Wordt verfijnd in Fase 3 na het onderzoek.

---

## 1. Executive Summary

- 30 skills verdeeld over 5 categorieen voor Tauri 2 desktop app development
- Enkel technologiepakket (Rust backend + TypeScript/JavaScript frontend)
- Dekt: Commands, Events, Plugins, Window, State, Mobile, Build, Security
- Doel: Claude in staat stellen correcte Tauri 2 code te schrijven zonder API-hallucinaties

## 2. Technology Scope

- **Tauri 2.x** (latest stable)
- **Rust:** tauri crate, tauri-build, tauri-plugin
- **TypeScript/JavaScript:** @tauri-apps/api v2
- **Config:** tauri.conf.json, capability files
- **Platforms:** Windows, macOS, Linux, Android, iOS

## 3. Skill Inventory (VOORLOPIG — wordt verfijnd in Fase 3)

### tauri-core/ (4 skills)

#### tauri-core-architecture

- **Scope:** Tauri 2 architectuuroverzicht, Rust core, webview layer, IPC bridge, procesmodel (main vs webview), event loop
- **Key APIs:** App, AppHandle, Manager, Runtime, Webview, WebviewWindow
- **Complexity:** M
- **Dependencies:** Geen

#### tauri-core-versions

- **Scope:** Versiematrix Tauri 1.x vs 2.x, breaking changes, migratie-overzicht, nieuwe v2-concepten
- **Key APIs:** Versie-specifieke verschillen, deprecated APIs
- **Complexity:** M
- **Dependencies:** Geen

#### tauri-core-config

- **Scope:** tauri.conf.json complete referentie, alle secties (app, build, bundle, plugins, security), env vars, platform overrides
- **Key APIs:** Configuration schema, JSON-structuur
- **Complexity:** L
- **Dependencies:** Geen

#### tauri-core-runtime

- **Scope:** Rust runtime, tokio async, App/AppHandle lifecycle, setup hooks, run callbacks, Manager trait, restart/exit
- **Key APIs:** Builder, App, AppHandle, Manager, setup(), on_window_event()
- **Complexity:** M
- **Dependencies:** core-architecture

---

### tauri-syntax/ (10 skills)

#### tauri-syntax-commands

- **Scope:** #[tauri::command] macro, sync/async commands, argument types, return types, Result/Error, State<>, Window, AppHandle params
- **Key APIs:** #[tauri::command], invoke_handler, generate_handler!
- **Complexity:** L (centraal in Tauri)
- **Dependencies:** core-architecture

#### tauri-syntax-invoke

- **Scope:** Frontend invoke() API, @tauri-apps/api/core, TypeScript types, argument passing, response handling, error handling, convertFileSrc()
- **Key APIs:** invoke(), InvokeArgs, convertFileSrc
- **Complexity:** M
- **Dependencies:** syntax-commands

#### tauri-syntax-events

- **Scope:** Event system, emit/listen/once/emit_to, event payloads, serde serialization, global vs window-scoped, frontend+backend APIs
- **Key APIs:** Emitter, Listener, emit(), listen(), once(), Event
- **Complexity:** M
- **Dependencies:** core-architecture

#### tauri-syntax-state

- **Scope:** Managed state, app.manage(), State<T> in commands, AppHandle.state(), concurrent state met Mutex/RwLock
- **Key APIs:** manage(), State<T>, AppHandle::state()
- **Complexity:** S
- **Dependencies:** syntax-commands

#### tauri-syntax-window

- **Scope:** Window API, WebviewWindowBuilder, window labels, multi-window creatie, window events, JS WebviewWindow methods, config opties
- **Key APIs:** WebviewWindow, WebviewWindowBuilder, WindowBuilder, WindowEvent
- **Complexity:** M
- **Dependencies:** core-architecture

#### tauri-syntax-menu

- **Scope:** Menu system, Menu/MenuItem/Submenu/PredefinedMenuItem/CheckMenuItem/IconMenuItem, system tray, context menus, event handling
- **Key APIs:** Menu, MenuBuilder, TrayIcon, TrayIconBuilder, MenuEvent
- **Complexity:** M
- **Dependencies:** core-architecture

#### tauri-syntax-permissions

- **Scope:** Permissions system, capability files, permission definitions, scope definitions, default permissions, plugin permissions, ACL
- **Key APIs:** Capabilities, Permissions, Scopes, default.json, capability files
- **Complexity:** L (nieuw in v2, complex)
- **Dependencies:** core-config

#### tauri-syntax-path

- **Scope:** Path en file APIs, @tauri-apps/api/path, app directories, resource resolution, BaseDirectory enum
- **Key APIs:** resolve(), appDataDir(), BaseDirectory, resourceDir()
- **Complexity:** S
- **Dependencies:** core-architecture

#### tauri-syntax-plugins-api

- **Scope:** Gebruik officiele plugins (fs, http, dialog, notification, shell, clipboard, os, process, updater, store), Cargo.toml + permissions setup
- **Key APIs:** tauri-plugin-fs, tauri-plugin-http, tauri-plugin-dialog, etc.
- **Complexity:** L (veel plugins)
- **Dependencies:** syntax-permissions

#### tauri-syntax-ipc-advanced

- **Scope:** Geavanceerde IPC, channels/streaming, Channel<T>, raw IPC payloads, binary data transfer, custom protocols (tauri://, asset://)
- **Key APIs:** Channel, ipc::Channel, tauri://, asset://, convertFileSrc
- **Complexity:** M
- **Dependencies:** syntax-commands, syntax-invoke

---

### tauri-impl/ (9 skills)

#### tauri-impl-project-setup

- **Scope:** Nieuw project workflow, create-tauri-app, projectstructuur, frontend framework integratie, src-tauri/ layout, dev/build commands
- **Key APIs:** create-tauri-app, tauri dev, tauri build, project scaffolding
- **Complexity:** M
- **Dependencies:** core-config

#### tauri-impl-plugin-development

- **Scope:** Custom plugin creatie, tauri::plugin::Builder, plugin commands/state/lifecycle, mobile hooks, plugin permissions
- **Key APIs:** plugin::Builder, PluginApi, on_event, mobile plugin hooks
- **Complexity:** L
- **Dependencies:** syntax-commands, syntax-permissions

#### tauri-impl-multi-window

- **Scope:** Multi-window workflows, vensters aanmaken vanuit Rust/JS, venstercommunicatie, parent-child, splashscreen, modale dialogen
- **Key APIs:** WebviewWindowBuilder, emit_to(), window labels, window events
- **Complexity:** M
- **Dependencies:** syntax-window, syntax-events

#### tauri-impl-mobile

- **Scope:** Mobiele ontwikkeling, Android/iOS targets, tauri android/ios dev, platformspecifieke code, mobiele permissions
- **Key APIs:** #[cfg(target_os)], mobile plugin hooks, Android/iOS lifecycle
- **Complexity:** L
- **Dependencies:** core-runtime, impl-project-setup

#### tauri-impl-build-deploy

- **Scope:** Build en deployment, tauri build, bundler config, installers, code signing, auto-updater, CI/CD
- **Key APIs:** tauri build, bundler, NSIS/MSI/DMG/AppImage, updater plugin, GitHub Actions
- **Complexity:** L
- **Dependencies:** core-config

#### tauri-impl-testing

- **Scope:** Teststrategieen, unit testing Rust commands, IPC testing, frontend mocks, WebDriver, E2E
- **Key APIs:** @tauri-apps/api/mocks, mockIPC, WebDriver
- **Complexity:** M
- **Dependencies:** syntax-commands, syntax-invoke

#### tauri-impl-migration

- **Scope:** Tauri 1->2 migratie, API-hernoemingen, allowlist naar permissions, package renames, config wijzigingen
- **Key APIs:** Migratiestappen, tauri-compat, hernoemde APIs
- **Complexity:** M
- **Dependencies:** core-versions

#### tauri-impl-security

- **Scope:** Security-implementatie, CSP, custom protocols, isolation pattern, file/HTTP/shell scopes, dangerous permissions audit
- **Key APIs:** CSP headers, isolation pattern, scope configuratie, dangerous permissions
- **Complexity:** L
- **Dependencies:** syntax-permissions, core-config

#### tauri-impl-database

- **Scope:** Database-integratie, SQLite via tauri-plugin-sql, file storage via tauri-plugin-store, custom DB (sqlx/diesel/rusqlite)
- **Key APIs:** tauri-plugin-sql, tauri-plugin-store, Database trait
- **Complexity:** M
- **Dependencies:** syntax-commands, syntax-plugins-api

---

### tauri-errors/ (4 skills)

#### tauri-errors-ipc

- **Scope:** IPC-fouten, serialisatiefouten, command not found, type mismatches, permission denied, async panics
- **Key APIs:** InvokeError, serde errors, permission errors
- **Complexity:** M
- **Dependencies:** syntax-commands, syntax-invoke

#### tauri-errors-build

- **Scope:** Build-fouten, Cargo failures, bundler errors, code signing, ontbrekende systeemafhankelijkheden, mobiele build-problemen
- **Key APIs:** Build error messages, system dependency list, linker errors
- **Complexity:** M
- **Dependencies:** impl-build-deploy

#### tauri-errors-permissions

- **Scope:** Permissiefouten, capability-misconfiguratie, scope-schendingen, CSP-schendingen, debugging
- **Key APIs:** Permission error messages, capability validation, scope debugging
- **Complexity:** M
- **Dependencies:** syntax-permissions

#### tauri-errors-runtime

- **Scope:** Runtime-fouten, window not found, state not managed, plugin not initialized, asset resolution, panics
- **Key APIs:** Runtime error types, panic handling, state errors
- **Complexity:** M
- **Dependencies:** core-runtime

---

### tauri-agents/ (3 skills)

#### tauri-agents-code-validator

- **Scope:** Validatiechecklist voor Tauri 2 code, command signatures, permission coverage, state management, error handling, anti-patterns
- **Complexity:** M
- **Dependencies:** ALLE syntax + impl skills

#### tauri-agents-project-scaffolder

- **Scope:** Genereer complete Tauri 2 projectstructuur, configureer plugins, permissions, commands, frontend
- **Complexity:** L
- **Dependencies:** ALLE core + syntax skills

#### tauri-agents-migration-validator

- **Scope:** Valideer Tauri 1->2 migratie, controleer deprecated APIs, valideer permissions, test IPC
- **Complexity:** M
- **Dependencies:** core-versions, impl-migration

---

## 4. Batch Execution Plan (VOORLOPIG)

| Batch | Skills | Aantal | Dependencies |
|-------|--------|--------|-------------|
| 1 | core-architecture, core-versions, core-config | 3 | Geen |
| 2 | core-runtime, syntax-commands, syntax-invoke | 3 | Batch 1 |
| 3 | syntax-events, syntax-state, syntax-window | 3 | Batch 1 |
| 4 | syntax-permissions, syntax-menu, syntax-path | 3 | Batch 1 |
| 5 | syntax-plugins-api, syntax-ipc-advanced, impl-project-setup | 3 | Batch 2 |
| 6 | impl-plugin-development, impl-multi-window, impl-mobile | 3 | Batch 2-3 |
| 7 | impl-build-deploy, impl-testing, impl-security | 3 | Batch 2-4 |
| 8 | impl-migration, impl-database, errors-ipc | 3 | Batch 2-4 |
| 9 | errors-build, errors-permissions, errors-runtime | 3 | Batch 5-7 |
| 10 | agents-code-validator, agents-project-scaffolder, agents-migration-validator | 3 | Alles hierboven |

## 5. Onderzoek Nodig (Fase 2)

Drie onderzoeksdomeinen:

1. **Rust Backend API** — Commands, State, Events, Plugins, Mobile, AppHandle
2. **Frontend/JS/TS API** — invoke, events, window, path, menu, mocks
3. **Config/Security/Build** — tauri.conf.json, permissions, bundler, CI/CD

## 6. Scope-afbakening

### IN SCOPE:

- Tauri 2.x stabiele APIs
- Zowel Rust als TypeScript/JavaScript
- Officieel plugin-ecosysteem
- Mobiele targets (Android, iOS)
- Permissions/capabilities systeem
- Build en deployment
- Migratie vanuit v1

### OUT OF SCOPE:

- Dedicated Tauri 1.x skills
- Electron-migratie
- Specifieke frontend frameworks (React/Vue/Svelte details)
- Third-party plugin development guides
- OS-level APIs die niet via Tauri worden blootgesteld

## 7. Risicofactoren

1. **Complexiteit permissions-systeem** — Nieuw in v2, kan extra onderzoekstijd vereisen
2. **Maturiteit mobiele API** — Sommige mobiele features zijn mogelijk nog in beta
3. **Snelheid plugin-ecosysteem** — Plugins worden regelmatig bijgewerkt, skills hebben mogelijk version pinning nodig
4. **Dual-language dekking** — Elke IPC-skill heeft zowel Rust ALS TypeScript nodig, wat de content verdubbelt
