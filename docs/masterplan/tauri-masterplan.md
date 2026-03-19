# Tauri 2 Skill Package — Definitive Masterplan

## Status

Phase 3 complete. Finalized from raw masterplan after vooronderzoek review.
Date: 2026-03-18

---

## Decisions Made During Refinement

| # | Decision | Rationale |
|---|----------|-----------|
| D-01 | **Merged** `tauri-core-versions` into `tauri-impl-migration` | Version differences are only useful in migration context. Standalone version matrix skill has no actionable scope — all content is "use v2, here is what changed." Migration skill already covers this. |
| D-02 | **Merged** `tauri-syntax-invoke` into `tauri-syntax-commands` | Invoke is the JS side of commands. Splitting them forces users to load two skills for one operation. The combined skill covers both Rust `#[tauri::command]` and TS `invoke()` — matching the REQUIREMENTS.md dual-language mandate (D-008). |
| D-03 | **Merged** `tauri-syntax-path` into `tauri-syntax-plugins-api` | Path API is thin (directory functions + path manipulation). It is better covered as part of the plugins overview alongside fs, dialog, etc. |
| D-04 | **Merged** `tauri-syntax-menu` tray content stays but scope clarified | Menu and tray are documented together in research (sections 2.7, 3.6, 3.7). Keep as single skill covering both. |
| D-05 | **Added** `tauri-syntax-webview` | Research section 3.4 reveals a distinct Webview API (multi-webview per window, reparenting, separate from Window API). Not covered by any raw masterplan skill. |
| D-06 | **Reordered** batches to respect actual dependency chains | `syntax-permissions` moved earlier (batch 3) because `syntax-plugins-api` depends on it. `impl-security` moved to batch 7 with `impl-build-deploy`. |
| D-07 | **Renamed** `tauri-agents-code-validator` to `tauri-agents-review` | Better describes its function: reviewing generated Tauri code for correctness. |
| D-08 | **Removed** `tauri-agents-migration-validator` | Overlap with `tauri-impl-migration`. The migration skill itself should contain validation checklists. Two agents is sufficient. |

**Result**: 30 raw skills → **27 definitive skills** (3 merges, 1 addition, 1 removal).

---

## Definitive Skill Inventory (27 skills)

### tauri-core/ (3 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `tauri-core-architecture` | Tauri 2 architecture overview; Rust core + webview layer; IPC bridge model; process model (main vs webview); project structure; key dependencies | `App`, `AppHandle`, `Manager`, `Runtime`, `Webview`, `WebviewWindow` | Vooronderzoek §1 | M | None |
| `tauri-core-config` | `tauri.conf.json` complete reference; all sections (build, app, bundle, plugins, security); window config; env vars; platform overrides; minimal viable config | Configuration schema, JSON structure, all top-level keys | Vooronderzoek §4 | L | None |
| `tauri-core-runtime` | Rust runtime; tokio async; App/AppHandle lifecycle; `setup()` hook; Builder API; `run()` callbacks; Manager trait methods; restart/exit; async runtime | `Builder`, `App`, `AppHandle`, `Manager`, `setup()`, `on_window_event()`, `on_menu_event()` | Vooronderzoek §2.4 | M | core-architecture |

### tauri-syntax/ (8 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `tauri-syntax-commands` | `#[tauri::command]` macro (sync/async); argument types; return types; `Result<T,E>` + error serialization; special injected params (`State`, `Window`, `AppHandle`, `Request`); command registration; `generate_handler!`; **Frontend `invoke()` API**; argument passing (camelCase); TypeScript typing; error handling; `Channel` for streaming | `#[tauri::command]`, `generate_handler![]`, `invoke<T>()`, `Channel<T>`, `InvokeArgs`, `InvokeOptions`, `ipc::Response`, `ipc::Request` | Vooronderzoek §2.1, §2.8, §3.1 | L | core-architecture |
| `tauri-syntax-events` | Event system (emit/listen/once); Emitter + Listener traits; event payloads (Serialize+Clone); global vs window-scoped; `emit_to`/`emit_filter`; frontend listen/emit/emitTo/once; TauriEvent built-in events; unlisten cleanup; event name validation | `Emitter`, `Listener`, `emit()`, `listen()`, `once()`, `Event`, `UnlistenFn`, `EventTarget` | Vooronderzoek §2.3, §3.2 | M | core-architecture |
| `tauri-syntax-state` | Managed state; `app.manage()`; `State<T>` in commands; `AppHandle.state()`/`try_state()`; mutable state with `Mutex`/`RwLock`; `std::sync::Mutex` vs `tokio::sync::Mutex`; state pitfalls (type mismatch panic, deadlocks) | `manage()`, `State<'_, T>`, `AppHandle::state()`, `Mutex<T>`, `RwLock<T>` | Vooronderzoek §2.2 | S | syntax-commands |
| `tauri-syntax-window` | Window API (Rust + JS); `WebviewWindowBuilder`; window labels; window config in tauri.conf.json; window events (`WindowEvent` enum); JS `Window` class methods; creating/manipulating windows; monitor info | `WebviewWindow`, `WebviewWindowBuilder`, `WindowEvent`, `getCurrentWindow()`, `Window`, `LogicalSize`, `PhysicalSize` | Vooronderzoek §2.6, §3.3 | M | core-architecture |
| `tauri-syntax-webview` | Webview API; multi-webview per window; `getCurrentWebview()`; webview sizing/positioning; reparenting; webview events; creating webviews inside windows | `Webview`, `getCurrentWebview()`, `getAllWebviews()`, `setZoom()`, `reparent()`, `setAutoResize()` | Vooronderzoek §3.4 | S | syntax-window |
| `tauri-syntax-menu` | Menu system (Rust + JS); `MenuBuilder`/`SubmenuBuilder`; menu item types; `PredefinedMenuItem`; context menus; menu events; system tray (`TrayIconBuilder`); tray events; JS Menu/TrayIcon APIs | `Menu`, `MenuBuilder`, `TrayIcon`, `TrayIconBuilder`, `MenuEvent`, `MenuItem`, `CheckMenuItem`, `Submenu`, `PredefinedMenuItem` | Vooronderzoek §2.7, §3.6, §3.7 | M | core-architecture |
| `tauri-syntax-permissions` | Permissions system (new in v2); capability files (JSON); permission definitions (TOML); scope definitions (allow/deny); permission sets; core permissions; plugin permission naming; custom command permissions; platform-specific capabilities; remote API access; schema generation; inline capabilities | Capabilities, Permissions, Scopes, `default.json`, capability files, `CommandScope`, `GlobalScope` | Vooronderzoek §5.1, §5.2, §9.4, §9.5 | L | core-config |
| `tauri-syntax-plugins-api` | Using official plugins (fs, dialog, http, notification, shell, clipboard, os, process, updater, store, opener); Cargo.toml + npm setup; permissions setup per plugin; **path API** (`appDataDir`, `join`, `resolve`, `BaseDirectory`); `convertFileSrc`; `isTauri()` | All `@tauri-apps/plugin-*` packages, `@tauri-apps/api/path`, `convertFileSrc`, `BaseDirectory` | Vooronderzoek §3.5, §3.8, §9.1–§9.6 | L | syntax-permissions |

### tauri-impl/ (10 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `tauri-impl-project-setup` | New project workflow; `create-tauri-app`; project structure; frontend framework integration; `src-tauri/` layout; `lib.rs` + `main.rs` split; dev/build commands; source control rules | `create-tauri-app`, `tauri dev`, `tauri build`, project scaffolding | Vooronderzoek §1 (Project Structure), §4.1 (Minimal Config) | M | core-config |
| `tauri-impl-plugin-development` | Custom plugin creation; `tauri::plugin::Builder`; plugin commands/state/lifecycle hooks; plugin configuration; mobile hooks; plugin permissions in `build.rs`; plugin commands invocation pattern (`plugin:<name>\|<cmd>`); scopes in plugin commands | `plugin::Builder`, `TauriPlugin`, `PluginApi`, `on_event`, `on_navigation`, `on_webview_ready`, `on_drop`, `CommandScope`, `GlobalScope` | Vooronderzoek §2.5 | L | syntax-commands, syntax-permissions |
| `tauri-impl-multi-window` | Multi-window workflows; creating windows from Rust/JS; window communication via events; show/hide patterns; splashscreen pattern; window lifecycle management | `WebviewWindowBuilder`, `emit_to()`, window labels, `get_webview_window()` | Vooronderzoek §2.6 (Multi-Window Pattern), §3.3 | M | syntax-window, syntax-events |
| `tauri-impl-mobile` | Mobile development; Android/iOS targets; prerequisites; `tauri android/ios init/dev/build`; platform-specific Rust code (`#[cfg]`); `lib.rs` restructuring; mobile entry point; bundle config; Cargo.toml crate-type | `#[cfg(mobile)]`, `#[cfg(desktop)]`, `#[cfg_attr(mobile, tauri::mobile_entry_point)]`, `staticlib`, `cdylib` | Vooronderzoek §7 | L | core-runtime, impl-project-setup |
| `tauri-impl-build-deploy` | Build and deployment; `tauri build`; bundler config per platform; installers (NSIS/MSI/DMG/AppImage); resource bundling; external binaries (sidecars); code signing (Windows/macOS/Linux); auto-updater setup + server response format; CI/CD with GitHub Actions + `tauri-action` | `tauri build`, `tauri bundle`, bundler config, updater plugin, `tauri-apps/tauri-action`, signing env vars | Vooronderzoek §6 | L | core-config |
| `tauri-impl-testing` | Test strategies; Rust unit testing of commands; frontend mocking (`mockIPC`, `mockWindows`, `clearMocks`, `mockConvertFileSrc`); E2E patterns | `@tauri-apps/api/mocks`, `mockIPC`, `mockWindows`, `clearMocks` | Vooronderzoek §3.9 | M | syntax-commands |
| `tauri-impl-migration` | Tauri 1→2 migration; config restructuring; allowlist→permissions; Rust API renames; JS API renames; event system changes; menu system overhaul; feature flag changes; plugin migration table; migration CLI; **version differences** (v1 vs v2 feature matrix) | Migration steps, `npx @tauri-apps/cli migrate`, renamed APIs | Vooronderzoek §8 | M | core-config |
| `tauri-impl-security` | Security implementation; CSP configuration; Tauri-specific protocols (`tauri:`, `asset:`, `ipc:`); asset protocol; `freezePrototype`; `dangerousDisableAssetCspModification`; isolation pattern; scope configuration; dangerous permissions audit; Windows URL scheme change | CSP headers, `assetProtocol`, `pattern`, protocol URLs | Vooronderzoek §5.3 | L | syntax-permissions, core-config |
| `tauri-impl-database` | Database integration; SQLite via `tauri-plugin-sql`; persistent storage via `tauri-plugin-store` (eager + lazy loading); custom DB integration patterns (sqlx/diesel/rusqlite via commands) | `tauri-plugin-sql`, `tauri-plugin-store`, `load()`, `LazyStore` | Vooronderzoek §9.3 (Store section) | M | syntax-commands, syntax-plugins-api |
| `tauri-impl-design-patterns` | Common Tauri 2 design patterns; component communication; state sharing; error handling patterns; repository/service patterns; command organization | Pattern implementations, trait patterns, module organization | Vooronderzoek §2, §10 | M | core-architecture, syntax-commands |

### tauri-errors/ (4 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `tauri-errors-ipc` | IPC errors; serialization failures; command not found; type mismatches; permission denied; async panics; `thiserror` + `Serialize` pattern; structured vs string errors; frontend error handling | `InvokeError`, serde errors, `thiserror::Error`, permission errors | Vooronderzoek §2.1 (Error Handling), §10.1 (#1–#9), §10.2 (#10–#17) | M | syntax-commands |
| `tauri-errors-build` | Build errors; Cargo failures; bundler errors; code signing failures; missing system dependencies (Linux libs); mobile build problems; updater artifact issues | Build error messages, system dependency list, linker errors | Vooronderzoek §6, §7.1, §10.6 (#30–#33) | M | impl-build-deploy |
| `tauri-errors-permissions` | Permission errors; capability misconfiguration; scope violations; CSP violations; debugging missing permissions; core vs plugin permission confusion | Permission error messages, capability validation, scope debugging | Vooronderzoek §5, §10.3 (#18–#21), §10.5 (#27–#29) | M | syntax-permissions |
| `tauri-errors-runtime` | Runtime errors; window not found; state not managed (type mismatch panic); plugin not initialized; asset resolution failures; event name validation panics | Runtime error types, panic handling, state errors | Vooronderzoek §2.2 (State Pitfalls), §10.1, §10.4 (#23–#26) | M | core-runtime |

### tauri-agents/ (2 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `tauri-agents-review` | Validation checklist for generated Tauri 2 code; command signature correctness; permission coverage; state management patterns; error handling; anti-pattern detection; IPC bridge completeness (Rust+TS pair) | All validation rules from all skills | Vooronderzoek §10 (all anti-patterns) | M | ALL syntax + impl skills |
| `tauri-agents-project-scaffolder` | Generate complete Tauri 2 project structure; configure plugins; set up permissions/capabilities; create commands with matching invoke calls; configure build pipeline | All scaffolding patterns | Vooronderzoek §1, §4, §9.6 | L | ALL core + syntax skills |

---

## Batch Execution Plan (DEFINITIVE)

| Batch | Skills | Count | Dependencies | Notes |
|-------|--------|-------|-------------|-------|
| 1 | `core-architecture`, `core-config` | 2 | None | Foundation skills, no dependencies |
| 2 | `core-runtime`, `syntax-commands` | 2 | Batch 1 | Runtime + primary IPC mechanism |
| 3 | `syntax-events`, `syntax-state`, `syntax-permissions` | 3 | Batch 1–2 | Core syntax patterns; permissions early for plugin deps |
| 4 | `syntax-window`, `syntax-menu`, `syntax-plugins-api` | 3 | Batch 1–3 | Window/menu/plugins depend on permissions |
| 5 | `syntax-webview`, `impl-project-setup`, `impl-testing` | 3 | Batch 2–4 | Webview extends window; project setup + testing |
| 6 | `impl-plugin-development`, `impl-multi-window`, `impl-mobile` | 3 | Batch 2–5 | Implementation patterns |
| 7 | `impl-build-deploy`, `impl-security`, `impl-migration`, `impl-design-patterns` | 4 | Batch 1–4 | Build/security/migration/patterns reference skills |
| 8 | `impl-database`, `errors-ipc`, `errors-permissions` | 3 | Batch 2–4 | Database + first error skills |
| 9 | `errors-build`, `errors-runtime` | 2 | Batch 7, Batch 2 | Remaining error skills |
| 10 | `agents-review`, `agents-project-scaffolder` | 2 | ALL above | Agent skills reference all other skills |

**Total**: 27 skills across 10 batches.

---

## Per-Skill Agent Prompts

### Constants

```
PROJECT_ROOT = C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package
RESEARCH_FILE = C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\docs\research\vooronderzoek-tauri.md
REQUIREMENTS_FILE = C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\REQUIREMENTS.md
REFERENCE_SKILL = C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package\skills\blender\core\blender-core-api\SKILL.md
```

---

### Batch 1

#### Prompt: tauri-core-architecture

```
## Task: Create the tauri-core-architecture skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures for App, AppHandle, Manager, Runtime, Webview, WebviewWindow)
3. references/examples.md (working code examples)
4. references/anti-patterns.md (what NOT to do)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package\skills\blender\core\blender-core-api\SKILL.md

### YAML Frontmatter
---
name: tauri-core-architecture
description: "Guides Tauri 2 application architecture including Rust backend structure, webview layer, IPC bridge model, process model, project layout, and key type hierarchy. Activates when creating new Tauri apps, understanding Tauri project structure, or reasoning about Tauri's component model."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x with Rust and TypeScript."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Tauri 2 architecture overview: Rust core, webview layer, IPC bridge
- Process model: main process vs webview process
- Project structure: src-tauri/, capabilities/, permissions/, gen/schemas/
- Key type hierarchy: App, AppHandle, Manager trait, WebviewWindow, Webview, Window
- Key dependencies: @tauri-apps/api, @tauri-apps/cli, tauri crate
- Source control rules: commit Cargo.lock, ignore target/

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 1: Tauri 2 Architecture Overview (lines 28-90)
- Section 2.4: App Lifecycle & Runtime (lines 556-672) — for type hierarchy only
- Section 11: API Surface Summary (lines 3646-3714) — for quick reference tables

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language, not "you should" or "consider"
- Cover both Rust and TypeScript perspectives where applicable
- All code examples must be verified against vooronderzoek research
- Include version annotations (e.g., "Tauri 2.x")
- Include Critical Warnings section with NEVER rules
```

#### Prompt: tauri-core-config

```
## Task: Create the tauri-core-config skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-config\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (complete tauri.conf.json schema reference)
3. references/examples.md (config examples: minimal, full, platform-specific)
4. references/anti-patterns.md (common config mistakes)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package\skills\blender\core\blender-core-api\SKILL.md

### YAML Frontmatter
---
name: tauri-core-config
description: "Guides tauri.conf.json configuration including build settings, app settings, window configuration, bundle options, plugin configuration, and security settings. Activates when editing tauri.conf.json, configuring Tauri build options, or setting up platform-specific bundle configuration."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x with Rust and TypeScript."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- tauri.conf.json complete structure: top-level keys, build, app, bundle, plugins
- Window configuration: all properties with types and defaults
- Bundle section: platform-specific options (Windows/macOS/Linux/iOS/Android)
- Build section: devUrl, frontendDist, beforeDevCommand, beforeBuildCommand
- Security section overview (detail in syntax-permissions skill)
- Minimal viable config vs complete config
- Schema references ($schema for IDE autocompletion)

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 4: Configuration (lines 1918-2169)
- Section 6.3: Resource Bundling (lines 2511-2529) — for bundle.resources
- Section 10.4: Configuration anti-patterns (lines 3606-3614)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include a Quick Reference table of all top-level keys
- Include both minimal and complete config examples
- Include Critical Warnings section
```

---

### Batch 2

#### Prompt: tauri-core-runtime

```
## Task: Create the tauri-core-runtime skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-runtime\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (Builder methods, Manager trait methods, AppHandle methods)
3. references/examples.md (lifecycle patterns, setup hook patterns, background tasks)
4. references/anti-patterns.md (lifecycle mistakes)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package\skills\blender\core\blender-core-api\SKILL.md

### YAML Frontmatter
---
name: tauri-core-runtime
description: "Guides Tauri 2 application lifecycle including Builder configuration, setup hook, AppHandle usage, Manager trait, tokio async runtime, window event callbacks, and app exit/restart patterns. Activates when configuring app initialization, using setup hooks, spawning background tasks, or managing app lifecycle."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x with Rust and TypeScript."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Builder API: all methods (setup, invoke_handler, manage, plugin, menu, on_window_event, etc.)
- setup() hook: receives &mut App, return type, error handling, spawning threads
- AppHandle: Clone+Send+Sync, usage in threads, state access, window access
- Manager trait: unifying interface, key methods (state, get_webview_window, path, config)
- Tokio async runtime: automatic for async commands, tauri::async_runtime module
- Run callbacks: on_window_event, on_menu_event, on_tray_icon_event
- Exit/restart patterns

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 2.4: App Lifecycle & Runtime (lines 556-672)
- Section 2.5: Plugin System — setup pattern (lines 674-800) — lifecycle hooks only

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include complete Builder method table
- Include Manager trait method signatures
- Cover the setup() → AppHandle → thread spawning pattern
```

#### Prompt: tauri-syntax-commands

```
## Task: Create the tauri-syntax-commands skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-syntax\tauri-syntax-commands\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (command macro attributes, invoke signature, Channel API, Response/Request)
3. references/examples.md (sync commands, async commands, binary data, channels, error patterns)
4. references/anti-patterns.md (all command + invoke mistakes)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package\skills\blender\core\blender-core-api\SKILL.md

### YAML Frontmatter
---
name: tauri-syntax-commands
description: "Guides Tauri 2 command system including #[tauri::command] macro, sync/async commands, argument types, return types, error handling with thiserror, special injected parameters, command registration, and the frontend invoke() API with TypeScript typing, argument passing, error handling, and Channel streaming. Activates when writing Tauri commands, calling invoke(), handling IPC errors, or streaming data from Rust to JavaScript."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x with Rust and TypeScript."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- **Rust side**: #[tauri::command] macro and attributes; sync vs async commands; argument types (primitives, String, Vec, Option, custom structs, PathBuf); return types ((), T, Result<T,E>); error handling with thiserror + manual Serialize; binary data via ipc::Response; special params (AppHandle, WebviewWindow, State, ipc::Request); command registration (generate_handler!); runtime generics
- **TypeScript side**: invoke<T>() signature; InvokeArgs; InvokeOptions (headers); argument passing (camelCase rule); TypeScript return typing; error handling (try/catch); Channel<T> class for streaming; convertFileSrc; isTauri()
- **IPC Channels**: Channel<T> Rust type; streaming pattern; binary streaming; Channel vs Events comparison

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 2.1: Commands System (lines 96-341)
- Section 2.8: IPC Channels (lines 1045-1139)
- Section 3.1: Core invoke() API (lines 1146-1258)
- Section 3.8: Utilities (lines 1763-1830) — convertFileSrc, isTauri only
- Section 10.1: Rust anti-patterns #1-#9 (lines 3493-3511)
- Section 10.2: Frontend anti-patterns #10-#17 (lines 3513-3592)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- MUST show both Rust and TypeScript sides for every IPC pattern
- Include argument type mapping table (Rust ↔ JS)
- Include the thiserror + Serialize error pattern as Essential Pattern
- Include Channel streaming as Essential Pattern
```

---

### Batch 3

#### Prompt: tauri-syntax-events

```
## Task: Create the tauri-syntax-events skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-syntax\tauri-syntax-events\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (Emitter trait, Listener trait, frontend event functions, TauriEvent enum)
3. references/examples.md (broadcast, targeted, filtered events; React cleanup pattern)
4. references/anti-patterns.md (memory leaks, missing await, name validation)

### YAML Frontmatter
---
name: tauri-syntax-events
description: "Guides Tauri 2 event system including Emitter and Listener traits, emit/listen/once patterns, global vs window-scoped events, event payloads with serde, frontend event API, TauriEvent built-in events, and event cleanup patterns. Activates when implementing pub/sub messaging, broadcasting events, listening for window events, or handling bidirectional Rust-JS communication."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x with Rust and TypeScript."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- Rust Emitter trait: emit, emit_to, emit_filter, emit_str
- Rust Listener trait: listen, once, unlisten, listen_any, once_any
- Event payloads: Serialize + Clone requirement, custom structs
- Frontend: listen, emit, emitTo, once; UnlistenFn; Event<T> interface; EventTarget type
- TauriEvent enum (built-in events)
- Event name validation rules
- Cleanup patterns (React useEffect, generic teardown)
- Events vs Channels vs invoke comparison

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 2.3: Event System Rust Side (lines 458-554)
- Section 3.2: Event System Frontend Side (lines 1259-1398)
- Section 10.2: anti-pattern #10-#11 (listener cleanup)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- MUST show both Rust and TypeScript for each event pattern
- Include Event<T> type definition
- Include cleanup pattern as Critical Warning
```

#### Prompt: tauri-syntax-state

```
## Task: Create the tauri-syntax-state skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-syntax\tauri-syntax-state\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (manage, State<T>, state(), try_state(), Mutex, RwLock)
3. references/examples.md (immutable state, mutable state, async state, multi-state)
4. references/anti-patterns.md (type mismatch, deadlocks, redundant Arc)

### YAML Frontmatter
---
name: tauri-syntax-state
description: "Guides Tauri 2 state management including manage() registration, State<T> extraction in commands, AppHandle state access, mutable state with Mutex and RwLock, std vs tokio Mutex, and state pitfalls. Activates when managing application state, sharing data between commands, or implementing concurrent state access."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x with Rust."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- Registering state: Builder.manage() vs setup() hook
- Accessing state in commands: State<'_, T>
- Accessing state outside commands: AppHandle.state(), try_state()
- Mutable state: std::sync::Mutex, std::sync::RwLock
- When to use tokio::sync::Mutex (holding across .await)
- Internal Arc wrapping (do NOT add your own Arc)
- State pitfalls: type mismatch panic, deadlock, nested locks

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 2.2: State Management (lines 343-456)
- Section 10.1: anti-patterns #4 (Arc), #5 (deadlock)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include decision tree: when Mutex vs RwLock vs tokio::Mutex
- Type mismatch panic is Critical Warning
```

#### Prompt: tauri-syntax-permissions

```
## Task: Create the tauri-syntax-permissions skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-syntax\tauri-syntax-permissions\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (permission TOML format, capability JSON format, scope syntax, core permissions list)
3. references/examples.md (minimal capability, platform-specific, remote access, custom command permissions, full flow)
4. references/anti-patterns.md (permission mistakes, TOML vs JSON confusion, missing core: prefix)

### YAML Frontmatter
---
name: tauri-syntax-permissions
description: "Guides Tauri 2 permission and capability system including capability file format, permission definitions in TOML, scope configuration, permission sets, core permissions, plugin permission naming, platform-specific capabilities, remote API access, and schema generation. Activates when configuring permissions, creating capability files, setting up plugin access control, or debugging permission denied errors."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- Core concepts: permissions, scopes, capabilities
- Permission format: naming convention, TOML syntax, commands.allow/deny
- Scope definitions: allow/deny paths, URL patterns
- Permission sets: bundling multiple permissions
- Custom command permissions: TOML files in src-tauri/permissions/
- Capability files: JSON format, windows, permissions, platforms, remote
- Core permissions: core:path:default, core:event:default, etc.
- Plugin permission naming: auto-prepend tauri-plugin-
- Platform-specific capabilities
- Remote URL access
- Inline capabilities in tauri.conf.json
- Window merging behavior
- Schema generation (gen/schemas/)

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 5.1: Permissions System (lines 2173-2272)
- Section 5.2: Capabilities System (lines 2275-2402)
- Section 9.2: Core Permissions (lines 3115-3129)
- Section 9.4-9.5: Permission patterns (lines 3418-3446)
- Section 9.6: Custom Command Full Flow (lines 3447-3487)
- Section 10.3: Security anti-patterns #18-#21 (lines 3594-3603)
- Section 10.5: Permission anti-patterns #27-#29 (lines 3616-3623)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- MUST include capability file JSON examples
- MUST include TOML permission definition examples
- Include the full custom command flow (define → register → permission → capability → invoke)
- Include Critical Warnings about TOML-only for app permissions
```

---

### Batch 4

#### Prompt: tauri-syntax-window

```
## Task: Create the tauri-syntax-window skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-syntax\tauri-syntax-window\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (WebviewWindowBuilder, Window JS class, WindowEvent enum, monitor API)
3. references/examples.md (create window, manipulate, events, config)
4. references/anti-patterns.md (window management mistakes)

### YAML Frontmatter
---
name: tauri-syntax-window
description: "Guides Tauri 2 window management including WebviewWindowBuilder in Rust, Window class in TypeScript, window configuration in tauri.conf.json, window events, window manipulation methods, monitor information, and window lifecycle. Activates when creating windows, configuring window properties, handling window events, or managing window state."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x with Rust and TypeScript."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- Rust: WebviewWindowBuilder methods, WebviewUrl, window labels
- Rust: WindowEvent enum (all variants with fields)
- Rust: on_window_event handler pattern
- JS: Window class, getCurrentWindow, getAllWindows, Window.getByLabel, getFocusedWindow
- JS: Window manipulation (position, size, visibility, state, properties, lifecycle)
- JS: Window events (onCloseRequested, onResized, onMoved, etc.)
- JS: Creating windows from frontend
- Config: app.windows[] properties
- Monitor info: availableMonitors, currentMonitor, primaryMonitor, cursorPosition
- Key enums: CursorIcon, Theme, TitleBarStyle, UserAttentionType, ProgressBarStatus

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 2.6: Window Management Rust Side (lines 800-927)
- Section 3.3: Window API (lines 1400-1532)
- Section 4.1: Window Configuration (lines 2070-2089)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- MUST show both Rust and TypeScript for window creation
- Include WindowEvent variant table
- Include window config properties table
```

#### Prompt: tauri-syntax-menu

```
## Task: Create the tauri-syntax-menu skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-syntax\tauri-syntax-menu\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (MenuBuilder, SubmenuBuilder, TrayIconBuilder, all item types, JS Menu API)
3. references/examples.md (app menu, context menu, tray icon, dynamic menu)
4. references/anti-patterns.md (menu mistakes)

### YAML Frontmatter
---
name: tauri-syntax-menu
description: "Guides Tauri 2 menu and system tray including MenuBuilder, SubmenuBuilder, menu item types, PredefinedMenuItem, context menus, menu event handling, TrayIconBuilder, tray events, and frontend Menu/TrayIcon APIs. Activates when creating application menus, context menus, system tray icons, or handling menu events."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x with Rust and TypeScript."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- Rust MenuBuilder, SubmenuBuilder, PredefinedMenuItem
- Rust menu item types: MenuItem, CheckMenuItem, IconMenuItem
- Rust on_menu_event handler
- Rust TrayIconBuilder: methods, events, icon handling
- JS Menu.new(), MenuItem.new(), Submenu.new(), CheckMenuItem.new()
- JS PredefinedMenuItem types (all 18 types)
- JS context menus: menu.popup()
- JS menu item management (append, prepend, insert, remove)
- JS TrayIcon.new(), tray events (Click, DoubleClick, Enter, Move, Leave)
- Builder.menu() and Builder.on_menu_event()

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 2.7: Menu & Tray Rust Side (lines 929-1043)
- Section 3.6: Menu API (lines 1656-1727)
- Section 3.7: Tray Icon API (lines 1728-1761)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- MUST show both Rust and TypeScript for menu creation
- Include PredefinedMenuItem type list
- Include TrayIconBuilder method table
```

#### Prompt: tauri-syntax-plugins-api

```
## Task: Create the tauri-syntax-plugins-api skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-syntax\tauri-syntax-plugins-api\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (per-plugin API: fs, dialog, http, notification, shell, clipboard, os, process, updater, store, opener; path API)
3. references/examples.md (plugin usage patterns for each major plugin; path examples)
4. references/anti-patterns.md (plugin configuration mistakes, missing permissions, path mistakes)

### YAML Frontmatter
---
name: tauri-syntax-plugins-api
description: "Guides usage of official Tauri 2 plugins including filesystem, dialog, HTTP, notification, shell, clipboard, OS info, process, updater, store, and opener. Also covers the path API (appDataDir, join, resolve, BaseDirectory) and convertFileSrc utility. Activates when using any official Tauri plugin, working with file paths, accessing system information, or configuring plugin permissions."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x with Rust and TypeScript."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- Complete plugin list with Cargo crate, npm package, and default permission
- Per-plugin usage: fs, dialog, http, notification, shell, clipboard, os, process, updater, store, opener
- Plugin setup: Cargo.toml dependency + npm install + permission in capability file
- Path API: all directory functions, path manipulation (join, resolve, dirname, basename, extname), BaseDirectory enum, resolveResource
- convertFileSrc utility
- Plugin permission patterns (default, allow-*, deny-*)
- Scope configuration for fs, http, shell plugins

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 3.5: Path API (lines 1591-1654)
- Section 3.8: Utilities — convertFileSrc (lines 1763-1830)
- Section 9: Official Plugins Reference (lines 3078-3487)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include plugin quick reference table (crate, npm, permission)
- Include per-plugin code examples (at minimum in references/)
- Include BaseDirectory enum member table
- Include path manipulation function signatures
- Anti-patterns MUST cover: missing permissions, relative paths without BaseDirectory
```

---

### Batch 5

#### Prompt: tauri-syntax-webview

```
## Task: Create the tauri-syntax-webview skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-syntax\tauri-syntax-webview\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (Webview class methods, creation options)
3. references/examples.md (multi-webview, reparenting, sizing)
4. references/anti-patterns.md (webview vs window confusion)

### YAML Frontmatter
---
name: tauri-syntax-webview
description: "Guides Tauri 2 Webview API including multi-webview per window, getCurrentWebview, webview sizing and positioning, reparenting, auto-resize, webview events, and creating webviews inside existing windows. Activates when working with multiple webviews, reparenting webviews, or needing finer control than the Window API provides."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x with TypeScript."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- Webview vs Window distinction
- getCurrentWebview(), getAllWebviews()
- Webview methods: setZoom, clearAllBrowsingData, size, position, setSize, setPosition, setAutoResize, show, hide, setFocus, reparent, close
- Creating a Webview inside a Window (JS constructor)
- Webview events: listen, onDragDropEvent
- When to use Webview API vs Window API

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 3.4: Webview API (lines 1534-1589)
- Section 3.10: Package Information (lines 1899-1914) — for @tauri-apps/api/webview

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include clear distinction table: Window vs Webview
- This is a smaller skill (S complexity)
```

#### Prompt: tauri-impl-project-setup

```
## Task: Create the tauri-impl-project-setup skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-impl\tauri-impl-project-setup\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (CLI commands, project structure reference)
3. references/examples.md (new project workflow, framework integration)
4. references/anti-patterns.md (setup mistakes)

### YAML Frontmatter
---
name: tauri-impl-project-setup
description: "Guides Tauri 2 project setup including create-tauri-app scaffolding, project structure conventions, lib.rs and main.rs split for mobile support, frontend framework integration, dev and build commands, and source control rules. Activates when creating a new Tauri project, setting up an existing web app with Tauri, or configuring the development workflow."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x with Rust and TypeScript."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- create-tauri-app workflow
- Project structure: src/, src-tauri/, capabilities/, permissions/, icons/, gen/
- lib.rs + main.rs split (required for mobile support)
- #[cfg_attr(mobile, tauri::mobile_entry_point)] macro
- Cargo.toml: crate-type for mobile, tauri features
- package.json: @tauri-apps/cli, @tauri-apps/api
- Dev workflow: tauri dev, tauri build
- Source control: commit Cargo.lock, .taurignore
- Frontend framework integration (generic, not framework-specific)
- Minimal viable tauri.conf.json

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 1: Architecture Overview — project structure (lines 39-90)
- Section 4.1: Minimal Viable Config (lines 2138-2169)
- Section 7.3-7.4: Mobile entry point (lines 2908-2936)
- Section 10.4: Config anti-patterns (lines 3606-3614)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include step-by-step project creation workflow
- Include complete project structure diagram
```

#### Prompt: tauri-impl-testing

```
## Task: Create the tauri-impl-testing skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-impl\tauri-impl-testing\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (mockIPC, mockWindows, mockConvertFileSrc, clearMocks signatures)
3. references/examples.md (unit test patterns, mock patterns, cleanup)
4. references/anti-patterns.md (testing mistakes)

### YAML Frontmatter
---
name: tauri-impl-testing
description: "Guides Tauri 2 testing strategies including frontend IPC mocking with mockIPC, window mocking with mockWindows, mock cleanup patterns, Rust command unit testing, and end-to-end testing approaches. Activates when writing tests for Tauri applications, mocking IPC calls, or setting up test infrastructure."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x with TypeScript."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- Frontend mocking: mockIPC (command + event mocking), mockWindows, mockConvertFileSrc
- clearMocks: always in afterEach
- Rust command unit testing patterns
- E2E testing overview
- Test file organization

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 3.9: Testing and Mocking (lines 1831-1897)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include complete mockIPC example with command routing
- Include React/generic cleanup pattern
```

---

### Batch 6

#### Prompt: tauri-impl-plugin-development

```
## Task: Create the tauri-impl-plugin-development skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-impl\tauri-impl-plugin-development\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (plugin::Builder, lifecycle hooks, configuration, mobile hooks, build.rs)
3. references/examples.md (basic plugin, configured plugin, plugin with commands and state)
4. references/anti-patterns.md (plugin development mistakes)

### YAML Frontmatter
---
name: tauri-impl-plugin-development
description: "Guides custom Tauri 2 plugin development including plugin::Builder, plugin commands and state, lifecycle hooks (setup, on_navigation, on_webview_ready, on_event, on_drop), plugin configuration, mobile hooks, permission generation in build.rs, and plugin command invocation from JavaScript. Activates when creating custom plugins, extending Tauri functionality, or building reusable plugin packages."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x with Rust and TypeScript."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- plugin::Builder API: new, invoke_handler, setup, on_navigation, on_webview_ready, on_event, on_drop, build
- Plugin with configuration (Deserialize config struct)
- Plugin commands: Runtime generic, JS invocation pattern (plugin:<name>|<cmd>)
- Plugin state management
- Plugin permissions: build.rs with tauri_plugin::Builder, auto-generated allow/deny
- CommandScope and GlobalScope for scoped commands
- Lifecycle hook details

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 2.5: Plugin System Rust Side (lines 674-800)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include complete plugin creation workflow
- Include build.rs permission generation pattern
- Include JS invocation pattern for plugin commands
```

#### Prompt: tauri-impl-multi-window

```
## Task: Create the tauri-impl-multi-window skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-impl\tauri-impl-multi-window\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (window creation, communication, visibility patterns)
3. references/examples.md (settings window, splashscreen, inter-window messaging)
4. references/anti-patterns.md (multi-window mistakes)

### YAML Frontmatter
---
name: tauri-impl-multi-window
description: "Guides Tauri 2 multi-window patterns including creating windows from Rust and JavaScript, inter-window communication via events, show/hide window workflows, splashscreen patterns, and window lifecycle management. Activates when implementing multiple windows, inter-window communication, or window management workflows."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x with Rust and TypeScript."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- Creating windows: Rust WebviewWindowBuilder, JS Window constructor
- Window labels as unique identifiers
- Inter-window communication via emit_to / emitTo
- Show/hide patterns from commands
- Splashscreen → main window pattern
- get_webview_window() to find windows by label
- config-defined windows vs programmatic windows (create: false)

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 2.6: Multi-Window Pattern (lines 897-927)
- Section 3.3: Window creation from JS (lines 1497-1511)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include complete splashscreen workflow
- Include inter-window communication pattern
```

#### Prompt: tauri-impl-mobile

```
## Task: Create the tauri-impl-mobile skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-impl\tauri-impl-mobile\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (mobile CLI commands, cfg attributes, Cargo.toml config)
3. references/examples.md (Android setup, iOS setup, platform-specific code)
4. references/anti-patterns.md (mobile development mistakes)

### YAML Frontmatter
---
name: tauri-impl-mobile
description: "Guides Tauri 2 mobile development including Android and iOS target setup, prerequisites, tauri android/ios init/dev/build commands, platform-specific Rust code with cfg attributes, lib.rs restructuring for mobile, Cargo.toml crate-type configuration, and mobile bundle settings. Activates when targeting Android or iOS, setting up mobile development environment, or writing platform-specific code."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x with Rust."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- Prerequisites: Android (Studio, SDK, NDK, env vars, Rust targets) and iOS (Xcode, Cocoapods, Rust targets)
- CLI commands: tauri android/ios init, dev, build
- Cargo.toml: crate-type = ["staticlib", "cdylib", "rlib"]
- Entry point: lib.rs with #[cfg_attr(mobile, tauri::mobile_entry_point)]
- Platform-specific code: #[cfg(mobile)], #[cfg(desktop)], #[cfg(target_os = "android")], #[cfg(target_os = "ios")]
- Bundle config: iOS minimumSystemVersion, Android minSdkVersion/versionCode

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 7: Mobile Development (lines 2878-2968)
- Section 10.7: Migration anti-pattern #37 (mobile restructuring)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include complete prerequisite checklist per platform
- Include cfg attribute decision table
```

---

### Batch 7

#### Prompt: tauri-impl-build-deploy

```
## Task: Create the tauri-impl-build-deploy skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-impl\tauri-impl-build-deploy\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (build commands, bundler options, signing config, updater config, CI/CD config)
3. references/examples.md (GitHub Actions workflow, signing setup, updater server response)
4. references/anti-patterns.md (build/deploy mistakes)

### YAML Frontmatter
---
name: tauri-impl-build-deploy
description: "Guides Tauri 2 build and deployment including tauri build command, platform-specific bundlers (NSIS/MSI/DMG/AppImage), resource bundling, sidecar binaries, code signing for Windows and macOS, auto-updater setup with key generation and server response format, and CI/CD with GitHub Actions using tauri-action. Activates when building for production, configuring installers, setting up code signing, implementing auto-updates, or creating CI/CD pipelines."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- Build commands: tauri build, --no-bundle, --bundles, --config, android/ios build
- Platform bundlers: Windows (NSIS, WiX), macOS (App, DMG), Linux (AppImage, deb, rpm), Android (APK, AAB), iOS (IPA)
- Resource bundling: bundle.resources, externalBin (sidecars)
- Code signing: Windows (Authenticode, certificateThumbprint, signCommand), macOS (Developer ID, notarization, ad-hoc), Linux (GPG)
- Auto-updater: plugin setup, key generation, config (pubkey, endpoints, install modes), server response format (static + dynamic), JS/Rust update check, generated artifacts
- CI/CD: GitHub Actions with tauri-action, cross-platform matrix, code signing in CI

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 6: Build & Distribution (lines 2479-2875)
- Section 10.6: Build anti-patterns #30-#33 (lines 3624-3633)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include platform bundler table
- Include complete GitHub Actions workflow
- Include updater server response JSON format
- Include signing checklist per platform
```

#### Prompt: tauri-impl-security

```
## Task: Create the tauri-impl-security skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-impl\tauri-impl-security\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (CSP directives, security config options, protocol reference)
3. references/examples.md (CSP configurations, security hardening checklist)
4. references/anti-patterns.md (security mistakes — all from research)

### YAML Frontmatter
---
name: tauri-impl-security
description: "Guides Tauri 2 security implementation including Content Security Policy configuration, Tauri-specific protocols (tauri://, asset://, ipc://), asset protocol scope, freezePrototype, isolation patterns, dangerous permissions audit, and Windows URL scheme changes. Activates when configuring CSP, hardening security, auditing permissions, or debugging protocol-related issues."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- CSP configuration in tauri.conf.json
- Tauri-specific protocols: tauri:, asset:/https://asset.localhost, ipc:/http://ipc.localhost
- Common CSP directives for Tauri apps
- Security settings: freezePrototype, dangerousDisableAssetCspModification, assetProtocol (enable + scope)
- Isolation pattern (brownfield)
- Windows URL scheme change (http vs https in v2)
- Security audit checklist

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 5.3: Content Security Policy (lines 2404-2476)
- Section 10.3: Security anti-patterns #18-#22 (lines 3594-3604)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include CSP directive table with Tauri-specific values
- Include security hardening checklist
- All dangerous* options MUST have Critical Warnings
```

#### Prompt: tauri-impl-migration

```
## Task: Create the tauri-impl-migration skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-impl\tauri-impl-migration\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (config mapping table, Rust API renames, JS API renames, plugin migration table)
3. references/examples.md (migration CLI, before/after code, capability file conversion)
4. references/anti-patterns.md (migration mistakes)

### YAML Frontmatter
---
name: tauri-impl-migration
description: "Guides Tauri 1 to Tauri 2 migration including configuration restructuring, allowlist to permissions conversion, Rust API renames, JavaScript API renames, event system changes, menu system overhaul, feature flag changes, plugin migration table, and the migration CLI tool. Activates when migrating from Tauri v1, encountering deprecated APIs, or converting allowlist configuration to capabilities."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- Configuration restructuring: v1 key → v2 key mapping table
- Allowlist → permissions/capabilities migration
- Migration CLI: npx @tauri-apps/cli migrate
- Rust API changes: type renames, removed modules → plugins, removed manager methods
- JS API changes: module renames, per-plugin packages
- Event system changes: emit broadcast, emit_to, listen_any
- Menu system overhaul: all type renames
- Feature flag changes: removed and added flags
- Plugin migration table (Cargo crate + npm package)
- Updater UI removal (must implement manually)

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 8: Migration v1 to v2 (lines 2972-3075)
- Section 10.7: Migration anti-patterns #34-#37 (lines 3634-3643)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include complete config mapping table (v1 → v2)
- Include Rust type rename table
- Include JS module rename table
- Include plugin migration table
- Include migration validation checklist
```

---

### Batch 8

#### Prompt: tauri-impl-database

```
## Task: Create the tauri-impl-database skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-impl\tauri-impl-database\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (store API: load, LazyStore, get, set, delete, etc.; sql plugin overview)
3. references/examples.md (store patterns, SQLite integration, custom DB via commands)
4. references/anti-patterns.md (database mistakes)

### YAML Frontmatter
---
name: tauri-impl-database
description: "Guides Tauri 2 database and storage integration including tauri-plugin-store for persistent key-value storage, tauri-plugin-sql for SQLite, and custom database integration via commands. Activates when implementing persistent storage, using SQLite, or integrating custom databases like sqlx or diesel."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x with Rust and TypeScript."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- tauri-plugin-store: load(), LazyStore, get/set/delete/clear/save/reload/reset, autoSave, keys/values/entries/length
- tauri-plugin-sql: setup overview, SQLite integration
- Custom DB patterns: wrapping sqlx/diesel/rusqlite in commands with State<Mutex<Connection>>
- Choosing between store, sql plugin, and custom integration

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 9.3: Store plugin (lines 3388-3416)
- Section 9.1: SQL plugin entry (line 3108)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include decision tree: store vs sql plugin vs custom
- Include complete store usage example
```

#### Prompt: tauri-errors-ipc

```
## Task: Create the tauri-errors-ipc skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-errors\tauri-errors-ipc\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (error types, InvokeError, serde error patterns)
3. references/examples.md (error scenarios with solutions)
4. references/anti-patterns.md (error handling mistakes)

### YAML Frontmatter
---
name: tauri-errors-ipc
description: "Diagnoses and resolves Tauri 2 IPC errors including serialization failures, command not found, type mismatches between Rust and JavaScript, permission denied errors, async command panics, missing Serialize on error types, and incorrect argument naming. Activates when encountering invoke errors, serialization failures, or IPC communication problems."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- Serialization errors: missing Serialize, type mismatches, serde failures
- Command not found: unregistered commands, multiple invoke_handler calls
- Argument mismatches: snake_case vs camelCase, missing required args, Option<T> for optional
- Error type issues: Result<T, String> vs thiserror, missing Serialize on error enum
- Permission denied: command not in capability
- Async issues: &str in async commands, blocking main thread
- Frontend: missing await, unhandled promise rejection, snake_case keys

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 2.1: Error Handling (lines 215-268)
- Section 10.1: Rust anti-patterns #1-#9 (lines 3493-3511)
- Section 10.2: Frontend anti-patterns #10-#17 (lines 3513-3592)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Format as diagnostic table: Symptom → Cause → Fix
- Include code examples for each error scenario
```

#### Prompt: tauri-errors-permissions

```
## Task: Create the tauri-errors-permissions skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-errors\tauri-errors-permissions\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (permission error types, debugging tools)
3. references/examples.md (common permission errors with fixes)
4. references/anti-patterns.md (permission configuration mistakes)

### YAML Frontmatter
---
name: tauri-errors-permissions
description: "Diagnoses and resolves Tauri 2 permission and capability errors including command not allowed, missing plugin permissions, scope violations, CSP violations, TOML vs JSON format confusion, missing core: prefix, and wildcard misuse. Activates when encountering permission denied errors, capability configuration problems, or security-related runtime errors."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- "Command not allowed" errors: missing permission in capability
- Plugin permission not working: forgot to add to capability after npm/cargo install
- Scope violations: path traversal rejected, URL not in scope
- CSP violations: blocked resource loading, inline script errors
- Format confusion: TOML for permissions, JSON/TOML for capabilities
- Missing core: prefix for built-in features
- Wildcard misuse: windows: ["*"] with broad permissions
- Debugging workflow: check capability file → check permission name → check scope

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 5: Security & Permissions (lines 2173-2476)
- Section 10.3: Security anti-patterns #18-#22 (lines 3594-3604)
- Section 10.5: Permission anti-patterns #27-#29 (lines 3616-3623)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Format as diagnostic table: Symptom → Cause → Fix
- Include debugging flowchart
```

---

### Batch 9

#### Prompt: tauri-errors-build

```
## Task: Create the tauri-errors-build skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-errors\tauri-errors-build\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (system dependency list, build environment requirements)
3. references/examples.md (common build errors with solutions)
4. references/anti-patterns.md (build configuration mistakes)

### YAML Frontmatter
---
name: tauri-errors-build
description: "Diagnoses and resolves Tauri 2 build and bundling errors including Cargo compilation failures, missing Linux system dependencies, bundler errors, code signing failures, updater artifact issues, mobile build problems, and CI/CD pipeline errors. Activates when encountering build failures, bundling problems, signing errors, or CI/CD issues."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- Cargo compilation errors in Tauri context
- Missing system dependencies: Linux (libwebkit2gtk-4.1-dev, libappindicator3-dev, librsvg2-dev)
- Bundler errors per platform
- Code signing failures: Windows (missing cert/thumbprint), macOS (identity, notarization)
- Updater artifacts: missing TAURI_SIGNING_PRIVATE_KEY, v1Compatible confusion
- Mobile build: missing SDK/NDK, wrong Rust targets
- CI/CD: GitHub token permissions, runner dependencies

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 6: Build & Distribution (lines 2479-2875)
- Section 7.1: Mobile Prerequisites (lines 2880-2891)
- Section 10.6: Build anti-patterns #30-#33 (lines 3624-3633)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Format as diagnostic table: Error Message → Cause → Fix
- Include Linux dependency install command
- Include CI/CD troubleshooting section
```

#### Prompt: tauri-errors-runtime

```
## Task: Create the tauri-errors-runtime skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-errors\tauri-errors-runtime\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (runtime error types, panic sources)
3. references/examples.md (common runtime errors with solutions)
4. references/anti-patterns.md (runtime mistakes)

### YAML Frontmatter
---
name: tauri-errors-runtime
description: "Diagnoses and resolves Tauri 2 runtime errors including window not found, state not managed panic, plugin not initialized, asset resolution failures, event name validation panics, and configuration errors. Activates when encountering runtime panics, missing state errors, or unexpected runtime behavior."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- State not managed panic: type mismatch between manage() and State<T>
- Window not found: incorrect label, window not yet created
- Plugin not initialized: forgot .plugin() call
- Event name validation: invalid characters cause panic
- Asset resolution: resource not found, incorrect path
- Config errors: missing identifier, invalid beforeDevCommand
- Deadlock: nested Mutex locks

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 2.2: State Pitfalls (lines 431-456)
- Section 2.3: Event name validation (line 535)
- Section 10.1: Rust anti-patterns (lines 3493-3511)
- Section 10.4: Config anti-patterns (lines 3606-3614)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Format as diagnostic table: Symptom → Cause → Fix
- State type mismatch MUST be prominently featured (it panics at runtime, not compile time)
```

---

### Batch 10

#### Prompt: tauri-agents-review

```
## Task: Create the tauri-agents-review skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-agents\tauri-agents-review\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (complete validation checklist)
3. references/examples.md (review scenarios: good code, bad code, fixes)
4. references/anti-patterns.md (all anti-patterns consolidated from other skills)

### YAML Frontmatter
---
name: tauri-agents-review
description: "Validates generated Tauri 2 code for correctness by checking command signatures, permission coverage, state management patterns, error handling, IPC bridge completeness, and known anti-patterns. Activates when reviewing Tauri code, validating a Tauri project, or checking for common mistakes before deployment."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- Command validation: correct macro usage, async for I/O, proper error types with Serialize
- IPC bridge completeness: every Rust command has matching JS invoke call
- Permission coverage: every command has permission in capability file
- State management: correct type matching, no redundant Arc, proper Mutex usage
- Event handling: cleanup in components, Serialize+Clone payloads
- Configuration: identifier set, beforeBuildCommand set, Cargo.lock committed
- Security: CSP defined, freezePrototype enabled, no wildcard permissions with broad access
- Anti-pattern detection: all 37 anti-patterns from vooronderzoek §10

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 10: Common Mistakes & Anti-Patterns (lines 3491-3643) — ALL subsections

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Structure as a checklist that can be run against any Tauri project
- Group checks by area (commands, permissions, state, events, config, security, build)
- Each check item: what to verify, expected correct state, common failure mode
```

#### Prompt: tauri-agents-project-scaffolder

```
## Task: Create the tauri-agents-project-scaffolder skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-agents\tauri-agents-project-scaffolder\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (scaffolding templates, file generation patterns)
3. references/examples.md (complete project scaffold output)
4. references/anti-patterns.md (scaffolding mistakes)

### YAML Frontmatter
---
name: tauri-agents-project-scaffolder
description: "Generates complete Tauri 2 project structure including Rust backend with commands, TypeScript frontend with invoke calls, tauri.conf.json, capability files, permission definitions, Cargo.toml, and package.json. Activates when generating a new Tauri project from scratch, adding Tauri to an existing web app, or scaffolding a feature-complete Tauri application."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x with Rust and TypeScript."
metadata:
  author: Impertio
  version: "1.0"
---

### Scope (EXACT)
- Project structure generation: all required files and directories
- Rust scaffolding: lib.rs, main.rs, commands module, error module, state module
- Frontend scaffolding: invoke helpers, event listeners, type definitions
- Configuration: tauri.conf.json with sensible defaults
- Capabilities: default.json with core permissions + custom command permissions
- Permissions: TOML files for custom commands
- Cargo.toml: tauri dependency, plugin dependencies, crate-type for mobile
- package.json: @tauri-apps/api, @tauri-apps/cli, plugin packages
- Decision tree: which plugins to include based on project requirements

### Research Sections to Read
From vooronderzoek-tauri.md:
- Section 1: Project Structure (lines 39-90)
- Section 4.1: Complete Config (lines 1926-2031)
- Section 9.6: Custom Command Full Flow (lines 3447-3487)
- Section 11: API Surface Summary (lines 3646-3714) — for quick reference

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include complete file listing with content templates
- Include plugin selection decision tree
- Generated code MUST follow all patterns from syntax skills
- Generated config MUST include proper permissions
```

---

## Appendix: Skill Directory Structure

```
skills/source/
├── tauri-core/
│   ├── tauri-core-architecture/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── methods.md
│   │       ├── examples.md
│   │       └── anti-patterns.md
│   ├── tauri-core-config/
│   │   ├── SKILL.md
│   │   └── references/
│   ├── tauri-core-runtime/
│   │   ├── SKILL.md
│   │   └── references/
├── tauri-syntax/
│   ├── tauri-syntax-commands/
│   ├── tauri-syntax-events/
│   ├── tauri-syntax-state/
│   ├── tauri-syntax-window/
│   ├── tauri-syntax-webview/
│   ├── tauri-syntax-menu/
│   ├── tauri-syntax-permissions/
│   ├── tauri-syntax-plugins-api/
├── tauri-impl/
│   ├── tauri-impl-project-setup/
│   ├── tauri-impl-plugin-development/
│   ├── tauri-impl-multi-window/
│   ├── tauri-impl-mobile/
│   ├── tauri-impl-build-deploy/
│   ├── tauri-impl-testing/
│   ├── tauri-impl-migration/
│   ├── tauri-impl-security/
│   ├── tauri-impl-database/
├── tauri-errors/
│   ├── tauri-errors-ipc/
│   ├── tauri-errors-build/
│   ├── tauri-errors-permissions/
│   ├── tauri-errors-runtime/
├── tauri-agents/
│   ├── tauri-agents-review/
│   ├── tauri-agents-project-scaffolder/
```
