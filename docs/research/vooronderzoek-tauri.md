# Vooronderzoek: Tauri 2

Research document for the Tauri 2 Claude Skill Package.
Compiled from three research fragments covering Rust API, Frontend API, and Configuration/Security/Build.

**Date**: 2026-03-18
**Sources**: v2.tauri.app, docs.rs/tauri, GitHub tauri-apps
**Status**: Phase 2 complete

---

## Table of Contents

1. [Tauri 2 Architecture Overview](#1-tauri-2-architecture-overview)
2. [Rust Backend API](#2-rust-backend-api)
3. [Frontend / TypeScript API](#3-frontend--typescript-api)
4. [Configuration](#4-configuration)
5. [Security & Permissions](#5-security--permissions)
6. [Build & Distribution](#6-build--distribution)
7. [Mobile Development](#7-mobile-development)
8. [Migration v1 to v2](#8-migration-v1-to-v2)
9. [Official Plugins Reference](#9-official-plugins-reference)
10. [Common Mistakes & Anti-Patterns](#10-common-mistakes--anti-patterns)
11. [API Surface Summary](#11-api-surface-summary)

---

## 1. Tauri 2 Architecture Overview

Tauri 2 is a framework for building cross-platform desktop and mobile applications using a Rust backend with a web-based frontend (HTML/CSS/JS). The architecture consists of:

- **Rust Backend** (`src-tauri/`): Application logic, system access, state management. Commands defined with `#[tauri::command]` are exposed as IPC endpoints.
- **Frontend** (`src/`): Web UI rendered in a platform-native webview (WKWebView on macOS/iOS, WebView2 on Windows, WebKitGTK on Linux, Android WebView on Android).
- **IPC Bridge**: The `invoke()` function on the frontend calls Rust commands via JSON-serialized messages. Events provide bidirectional pub/sub messaging. Channels enable streaming data from Rust to JS.
- **Plugin System**: First-party and third-party plugins extend capabilities (filesystem, dialogs, HTTP, notifications, etc.). Each plugin has both a Rust crate and an npm package.
- **Permission System** (new in v2): Fine-grained access control via capabilities and permissions, replacing the v1 allowlist.
- **Runtime**: Tokio async runtime under the hood. Async commands are spawned on the tokio runtime automatically.

### Project Structure

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
│   ├── permissions/            # Custom command permissions (optional)
│   ├── icons/                  # Application icons (all sizes)
│   ├── gen/                    # Generated files (schemas, etc.)
│   │   └── schemas/
│   │       ├── desktop-schema.json
│   │       ├── mobile-schema.json
│   │       └── remote-schema.json
│   ├── Cargo.toml              # Rust dependencies
│   ├── Cargo.lock              # Deterministic build lock (commit this!)
│   ├── tauri.conf.json         # Main configuration
│   ├── build.rs                # Cargo build script
│   └── .taurignore             # Exclude files from watch
├── package.json                # Frontend dependencies
├── tsconfig.json               # TypeScript config (if applicable)
└── vite.config.ts              # Bundler config (if Vite)
```

### Key Dependencies

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

### Source Control

- **Commit**: `src-tauri/Cargo.lock` (deterministic builds)
- **Ignore**: `src-tauri/target/` (build artifacts)

---

## 2. Rust Backend API

> Based on Tauri 2 documentation (v2.10.2). Sources: v2.tauri.app/develop/*, docs.rs/tauri/2.10.2/*

### 2.1 Commands System

The command system is the primary mechanism for frontend-to-backend communication. The `#[tauri::command]` macro transforms regular Rust functions into IPC endpoints callable from JavaScript.

#### The `#[tauri::command]` Macro

Basic usage:

```rust
#[tauri::command]
fn greet(name: String) -> String {
    format!("Hello, {}!", name)
}
```

**Critical rule**: Commands defined in `lib.rs` cannot be marked `pub` due to glue code generation constraints. Commands in separate modules should be `pub`.

**Macro attributes:**
- `#[tauri::command(rename_all = "snake_case")]` — changes the argument naming convention for the JS side (default is camelCase)
- `#[tauri::command(async)]` — forces a synchronous function to run on the async runtime instead of the main thread

#### Sync vs Async Commands

**Synchronous commands** run on the main thread by default:

```rust
#[tauri::command]
fn compute(x: i32, y: i32) -> i32 {
    x + y
}
```

**Async commands** are spawned on the tokio runtime via `async_runtime::spawn`:

```rust
#[tauri::command]
async fn fetch_data(url: String) -> Result<String, String> {
    let response = reqwest::get(&url).await.map_err(|e| e.to_string())?;
    response.text().await.map_err(|e| e.to_string())
}
```

**When to use async**: Any command involving I/O, network, file system, or long-running computation should be async to avoid blocking the main thread and freezing the UI.

**Borrowed type limitation**: Async commands cannot use borrowed references (`&str`) directly. Use owned types (`String`) or wrap the return in `Result`:

```rust
// Option 1: use owned types
#[tauri::command]
async fn process(value: String) -> String {
    value.to_uppercase()
}

// Option 2: wrap return in Result to allow &str
#[tauri::command]
async fn process_ref(value: &str) -> Result<String, ()> {
    Ok(value.to_uppercase())
}
```

#### Argument Types

Commands accept any type that implements `serde::Deserialize`:

| Type | Example | Notes |
|------|---------|-------|
| Primitives | `i32`, `u64`, `f64`, `bool` | Direct mapping from JS numbers/booleans |
| `String` | `name: String` | JS string to Rust String |
| `Vec<T>` | `items: Vec<String>` | JS array to Rust Vec |
| `Option<T>` | `limit: Option<u32>` | JS null/undefined to None |
| Custom structs | `data: MyStruct` | Must `#[derive(Deserialize)]` |
| `PathBuf` | `path: std::path::PathBuf` | JS string to file path |

Arguments are passed from JavaScript as a JSON object with **camelCase** keys:

```javascript
invoke('my_command', { userName: 'Alice', itemCount: 42 });
```

Maps to Rust:

```rust
#[tauri::command]
fn my_command(user_name: String, item_count: u32) { }
```

#### Return Types

Commands can return:

- **`()`** — no return value (void)
- **`T`** — any type implementing `serde::Serialize`
- **`Result<T, E>`** — where `E` must implement `Serialize` (or `Into<InvokeError>`)

```rust
#[derive(serde::Serialize)]
struct User {
    name: String,
    age: u32,
}

#[tauri::command]
fn get_user() -> User {
    User { name: "Alice".into(), age: 30 }
}
```

**Binary data optimization** — bypass JSON serialization for large payloads:

```rust
use tauri::ipc::Response;

#[tauri::command]
fn read_file(path: String) -> Response {
    let data = std::fs::read(path).unwrap();
    tauri::ipc::Response::new(data)
}
```

#### Error Handling

The idiomatic pattern uses `thiserror` with a manual `Serialize` implementation:

```rust
#[derive(Debug, thiserror::Error)]
enum Error {
    #[error(transparent)]
    Io(#[from] std::io::Error),
    #[error("database error: {0}")]
    Database(String),
    #[error("not found")]
    NotFound,
}

// Required: Serialize for sending error to frontend
impl serde::Serialize for Error {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where S: serde::ser::Serializer {
        serializer.serialize_str(self.to_string().as_ref())
    }
}

#[tauri::command]
fn open_file(path: String) -> Result<String, Error> {
    let content = std::fs::read_to_string(path)?; // auto-converts via #[from]
    Ok(content)
}
```

For structured errors visible to the frontend, use tagged enums:

```rust
#[derive(serde::Serialize)]
#[serde(tag = "kind", content = "message")]
#[serde(rename_all = "camelCase")]
enum AppError {
    Io(String),
    NotFound(String),
    Unauthorized(String),
}
```

The frontend then receives typed error objects:

```typescript
try {
  await invoke('load_data');
} catch (err: any) {
  // err = { kind: 'notFound', message: 'File missing' }
  if (err.kind === 'notFound') {
    showNotFoundUI(err.message);
  }
}
```

#### Special Parameters (Injected Automatically)

These parameters are provided by Tauri at runtime — the frontend does NOT pass them:

```rust
#[tauri::command]
async fn complex_command(
    app: tauri::AppHandle,           // access to the application
    window: tauri::WebviewWindow,    // the calling window
    state: tauri::State<'_, MyState>, // managed state
    request: tauri::ipc::Request,    // raw IPC request (headers, body)
) -> Result<String, Error> {
    println!("Called from window: {}", window.label());
    let data = state.inner();
    Ok("done".into())
}
```

The `tauri::ipc::Request` gives access to raw headers and body:

```rust
#[tauri::command]
fn upload(request: tauri::ipc::Request) -> Result<(), Error> {
    let tauri::ipc::InvokeBody::Raw(data) = request.body() else {
        return Err(Error::InvalidBody);
    };
    let auth = request.headers().get("Authorization");
    Ok(())
}
```

#### Command Registration

All commands must be registered in a single `generate_handler![]` call:

```rust
// src-tauri/src/lib.rs
mod commands;

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![
            commands::greet,
            commands::fetch_data,
            commands::open_file,
        ])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

**Important**: Only one `.invoke_handler()` call is processed. Multiple calls do NOT merge — only the last one takes effect.

#### Runtime Generics

For commands used in plugins or requiring test mocking:

```rust
use tauri::{AppHandle, Runtime, WebviewWindow};

#[tauri::command]
async fn my_command<R: Runtime>(
    app: AppHandle<R>,
    window: WebviewWindow<R>,
) -> Result<(), String> {
    Ok(())
}
```

With the default `wry` feature, the runtime resolves to `Wry`.

### 2.2 State Management

#### Registering State

State is registered via `manage()` on the Builder or App:

```rust
struct AppConfig {
    api_url: String,
    max_retries: u32,
}

// Option 1: on Builder
tauri::Builder::default()
    .manage(AppConfig {
        api_url: "https://api.example.com".into(),
        max_retries: 3,
    })

// Option 2: in setup() hook (for state that depends on app initialization)
tauri::Builder::default()
    .setup(|app| {
        app.manage(AppConfig {
            api_url: "https://api.example.com".into(),
            max_retries: 3,
        });
        Ok(())
    })
```

#### Accessing State in Commands

```rust
#[tauri::command]
fn get_config(state: tauri::State<'_, AppConfig>) -> String {
    state.api_url.clone()
}
```

#### Accessing State Outside Commands

Via the `Manager` trait (implemented by `App`, `AppHandle`, `Window`, `WebviewWindow`):

```rust
let config = app_handle.state::<AppConfig>();
// or with option:
let maybe_config = app_handle.try_state::<AppConfig>();
```

#### Mutable State with Mutex

Tauri wraps managed state in `Arc` internally — you do NOT need `Arc`. But for mutation, you need interior mutability:

```rust
use std::sync::Mutex;

#[derive(Default)]
struct Counter {
    value: u32,
}

// Register
app.manage(Mutex::new(Counter::default()));

// Use in command
#[tauri::command]
fn increment(state: tauri::State<'_, Mutex<Counter>>) -> u32 {
    let mut counter = state.lock().unwrap();
    counter.value += 1;
    counter.value
}
```

**std::sync::Mutex vs tokio::sync::Mutex**: Use `std::sync::Mutex` for most cases. Only use `tokio::sync::Mutex` when you need to hold the guard across `.await` points:

```rust
// tokio::sync::Mutex needed here because of .await while holding lock
#[tauri::command]
async fn save_and_increment(
    state: tauri::State<'_, tokio::sync::Mutex<Counter>>,
) -> Result<u32, ()> {
    let mut counter = state.lock().await;
    // ... do async work while holding lock ...
    counter.value += 1;
    Ok(counter.value)
}
```

#### State Pitfalls

**Runtime panic on type mismatch**: Using `State<'_, AppConfig>` when you registered `Mutex<AppConfig>` causes a runtime panic, not a compile error. Always match the exact type:

```rust
// WRONG: panics at runtime
app.manage(Mutex::new(Counter::default()));
fn bad(state: State<'_, Counter>) { } // panic!

// CORRECT
fn good(state: State<'_, Mutex<Counter>>) { }
```

**Deadlock with nested locks**: Never lock the same Mutex twice in the same scope. Use `RwLock` when you need concurrent reads:

```rust
use std::sync::RwLock;

app.manage(RwLock::new(AppData::default()));

#[tauri::command]
fn read_data(state: State<'_, RwLock<AppData>>) -> String {
    let data = state.read().unwrap(); // multiple readers allowed
    data.name.clone()
}
```

### 2.3 Event System (Rust Side)

The event system provides bidirectional communication. For the frontend perspective, see [Section 3.2](#32-event-system-frontend-side).

#### Emitter Trait

The `Emitter` trait is implemented by `App`, `AppHandle`, `Webview`, `WebviewWindow`, and `Window`.

```rust
// Broadcast to all targets
fn emit<S: Serialize + Clone>(&self, event: &str, payload: S) -> Result<()>

// Emit with pre-serialized JSON payload
fn emit_str(&self, event: &str, payload: String) -> Result<()>

// Emit to a specific target
fn emit_to<I: Into<EventTarget>, S: Serialize + Clone>(
    &self, target: I, event: &str, payload: S
) -> Result<()>

// Emit to targets matching a filter
fn emit_filter<S: Serialize + Clone, F: Fn(&EventTarget) -> bool>(
    &self, event: &str, payload: S, filter: F
) -> Result<()>
```

```rust
use tauri::Emitter;

// Broadcast to all windows
app_handle.emit("data-updated", serde_json::json!({ "count": 42 }))?;

// Emit to a specific window
app_handle.emit_to("main", "notification", "Hello!")?;

// Emit with filter
app_handle.emit_filter("sync", payload, |target| {
    matches!(target, EventTarget::WebviewWindow { label } if label != "settings")
})?;
```

#### Listener Trait

The `Listener` trait is implemented by the same types as `Emitter`.

```rust
// Listen for events (returns EventId for unlisten)
fn listen<F>(&self, event: impl Into<String>, handler: F) -> EventId
where F: Fn(Event) + Send + 'static

// Listen once (auto-removes after first event)
fn once<F>(&self, event: impl Into<String>, handler: F) -> EventId
where F: FnOnce(Event) + Send + 'static

// Remove a listener
fn unlisten(&self, id: EventId)

// Listen for events from any source
fn listen_any<F>(&self, event: impl Into<String>, handler: F) -> EventId
where F: Fn(Event) + Send + 'static

// Listen once from any source
fn once_any<F>(&self, event: impl Into<String>, handler: F) -> EventId
where F: FnOnce(Event) + Send + 'static
```

```rust
use tauri::Listener;

let id = app_handle.listen("user-action", |event| {
    println!("Received event with payload: {:?}", event.payload());
});

// Later: remove the listener
app_handle.unlisten(id);
```

**Event name validation**: Event names may only contain alphanumeric characters, `-`, `/`, `:`, and `_`. Other characters cause a panic.

#### Event Payloads

Event payloads are serialized via serde. Any type implementing `Serialize + Clone` works:

```rust
#[derive(Clone, serde::Serialize)]
struct ProgressUpdate {
    task_id: String,
    percent: f64,
    message: String,
}

app_handle.emit("progress", ProgressUpdate {
    task_id: "download-1".into(),
    percent: 45.5,
    message: "Downloading file...".into(),
})?;
```

### 2.4 App Lifecycle & Runtime

#### Builder — The Entry Point

`tauri::Builder` provides the full application configuration API:

```rust
tauri::Builder::default()
    .manage(MyState::default())           // register state
    .plugin(my_plugin::init())            // register plugin
    .invoke_handler(tauri::generate_handler![cmd1, cmd2]) // register commands
    .setup(|app| {                        // initialization hook
        // app is &mut App — full access
        Ok(())
    })
    .on_window_event(|window, event| {    // window event handler
        // called for ALL window events
    })
    .on_menu_event(|app, event| {         // menu event handler
        println!("Menu item clicked: {:?}", event.id());
    })
    .menu(|app| {                         // build app menu
        MenuBuilder::new(app).build()
    })
    .run(tauri::generate_context!())
    .expect("error running app");
```

##### Complete Builder Method List (v2.10.2)

| Method | Purpose |
|--------|---------|
| `new()` | Create a new builder |
| `any_thread(self)` | Allow running on any thread (not just main) |
| `setup(F)` | App initialization hook |
| `invoke_handler(F)` | Register command handlers |
| `manage(T)` | Register managed state |
| `plugin(P)` | Register a plugin |
| `plugin_boxed(Box<dyn Plugin>)` | Register a boxed plugin |
| `menu(F)` | Set the application menu |
| `on_menu_event(F)` | Menu event handler |
| `on_tray_icon_event(F)` | Tray icon event handler |
| `on_window_event(F)` | Window event handler |
| `on_webview_event(F)` | Webview event handler |
| `on_page_load(F)` | Page load handler |
| `enable_macos_default_menu(bool)` | macOS default menu toggle |
| `register_uri_scheme_protocol(N, H)` | Custom URI scheme |
| `register_asynchronous_uri_scheme_protocol(N, H)` | Async URI scheme |
| `device_event_filter(DeviceEventFilter)` | Filter device events |
| `build(Context)` | Build without running (returns `App<R>`) |
| `run(Context)` | Build and run the app |

#### The `setup()` Hook

The setup hook runs after the app is initialized but before windows are created. It receives `&mut App<R>`:

```rust
.setup(|app| {
    // Access AppHandle for use in threads
    let handle = app.handle().clone();

    // Register state that depends on app paths
    let db_path = app.path().app_data_dir()?.join("data.db");
    app.manage(Database::new(&db_path)?);

    // Spawn background tasks
    std::thread::spawn(move || {
        loop {
            handle.emit("heartbeat", ()).unwrap();
            std::thread::sleep(std::time::Duration::from_secs(30));
        }
    });

    Ok(())
})
```

**Return type**: `Result<(), Box<dyn Error>>` — any error aborts app startup.

#### AppHandle

`AppHandle` is the primary handle for interacting with the application from any context. It is `Clone + Send + Sync`, making it safe to pass to threads:

```rust
let handle = app.handle().clone();
tokio::spawn(async move {
    let state = handle.state::<MyState>();
    handle.emit("event", "data").unwrap();
    let window = handle.get_webview_window("main");
});
```

#### The Manager Trait

The `Manager` trait is the unifying interface implemented by `App`, `AppHandle`, `Webview`, `WebviewWindow`, and `Window`. Key methods:

```rust
fn app_handle(&self) -> &AppHandle<R>
fn config(&self) -> &Config
fn package_info(&self) -> &PackageInfo
fn get_window(&self, label: &str) -> Option<Window<R>>
fn get_focused_window(&self) -> Option<Window<R>>
fn windows(&self) -> HashMap<String, Window<R>>
fn get_webview_window(&self, label: &str) -> Option<WebviewWindow<R>>
fn webview_windows(&self) -> HashMap<String, WebviewWindow<R>>
fn manage<T: Send + Sync + 'static>(&self, state: T) -> bool
fn unmanage<T: Send + Sync + 'static>(&self) -> Option<T>
fn state<T: Send + Sync + 'static>(&self) -> State<'_, T>
fn try_state<T: Send + Sync + 'static>(&self) -> Option<State<'_, T>>
fn path(&self) -> &PathResolver<R>
fn add_capability(&self, capability: impl RuntimeCapability) -> Result<()>
fn resources_table(&self) -> MutexGuard<'_, ResourceTable>  // only required method
```

#### Async Runtime

Tauri 2 uses **tokio** under the hood. Async commands are automatically spawned on the tokio runtime — you do NOT need to configure tokio yourself. The `tauri::async_runtime` module provides access if needed.

### 2.5 Plugin System (Rust Side)

#### Plugin Builder

```rust
use tauri::plugin::{Builder, TauriPlugin};
use tauri::Runtime;

pub fn init<R: Runtime>() -> TauriPlugin<R> {
    Builder::new("my-plugin")
        .invoke_handler(tauri::generate_handler![commands::do_something])
        .setup(|app, _api| {
            app.manage(PluginState::default());
            Ok(())
        })
        .build()
}
```

#### Plugin with Configuration

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct PluginConfig {
    timeout: u64,
    api_key: String,
}

pub fn init<R: Runtime>() -> TauriPlugin<R, PluginConfig> {
    Builder::<R, PluginConfig>::new("my-plugin")
        .setup(|app, api| {
            let config = api.config();
            println!("Timeout: {}", config.timeout);
            Ok(())
        })
        .build()
}
```

#### Plugin Lifecycle Hooks

```rust
Builder::new("my-plugin")
    // Called when plugin initializes
    .setup(|app, api| {
        Ok(())
    })
    // Called on webview navigation
    .on_navigation(|window, url| {
        url.scheme() != "forbidden" // return false to cancel
    })
    // Called when a new webview is ready
    .on_webview_ready(|window| {
        window.listen("content-loaded", |_| {
            println!("content loaded");
        });
    })
    // Called on app events (exit, window events, etc.)
    .on_event(|app, event| {
        match event {
            RunEvent::ExitRequested { api, .. } => {
                api.prevent_exit();
            }
            RunEvent::Exit => {
                // cleanup
            }
            _ => {}
        }
    })
    // Called when plugin is destroyed
    .on_drop(|app| {
        // cleanup resources
    })
    .build()
```

#### Plugin Commands

Plugin commands are invoked from JS with the prefix `plugin:<name>|<command>`:

```typescript
// JavaScript side
await invoke('plugin:my-plugin|do_something', { arg: 'value' });
```

```rust
// Rust side — plugin commands often use Runtime generic
#[tauri::command]
async fn do_something<R: Runtime>(
    app: AppHandle<R>,
    window: Window<R>,
    arg: String,
) -> Result<String, String> {
    Ok(format!("Done: {}", arg))
}
```

#### Plugin Permissions

Permissions are auto-generated from command names in `build.rs`:

```rust
// build.rs
const COMMANDS: &[&str] = &["do_something", "get_status"];

fn main() {
    tauri_plugin::Builder::new(COMMANDS).build();
}
```

This generates `allow-do-something` and `deny-do-something` permissions automatically. Scopes can be accessed in commands:

```rust
use tauri::ipc::{CommandScope, GlobalScope};

#[tauri::command]
async fn scoped_command<R: Runtime>(
    command_scope: CommandScope<'_, ScopeEntry>,
    global_scope: GlobalScope<'_, ScopeEntry>,
) -> Result<(), String> {
    let allowed = command_scope.allows();
    let denied = command_scope.denies();
    Ok(())
}
```

### 2.6 Window Management (Rust Side)

For the frontend perspective on window management, see [Section 3.3](#33-window-api).

#### Creating Windows

```rust
use tauri::webview::WebviewWindowBuilder;
use tauri::WebviewUrl;

// In setup() or any context with Manager access
let window = WebviewWindowBuilder::new(
    app,
    "settings",                          // window label (unique ID)
    WebviewUrl::App("settings.html".into()), // URL to load
)
    .title("Settings")
    .inner_size(800.0, 600.0)
    .center()
    .resizable(true)
    .build()?;
```

##### Key WebviewWindowBuilder Methods

```rust
// Window geometry
.inner_size(width: f64, height: f64)
.min_inner_size(w: f64, h: f64)
.max_inner_size(w: f64, h: f64)
.position(x: f64, y: f64)
.center()

// Window behavior
.title(title: impl Into<String>)
.visible(bool)
.focused(bool)
.maximized(bool)
.fullscreen(bool)
.resizable(bool)
.closable(bool)
.minimizable(bool)
.maximizable(bool)
.focusable(bool)
.transparent(bool)

// Webview configuration
.initialization_script(script: impl Into<String>)
.user_agent(ua: &str)
.devtools(enabled: bool)
.disable_javascript()
.zoom_hotkeys_enabled(bool)

// Event handlers
.on_page_load(F)
.on_navigation(F)
.on_web_resource_request(F)
.on_download(F)
.on_document_title_changed(F)
.on_new_window(F)
```

#### Window Events

The `WindowEvent` enum variants:

| Variant | Fields | Description |
|---------|--------|-------------|
| `Resized` | `PhysicalSize<u32>` | Window client area resized |
| `Moved` | `PhysicalPosition<i32>` | Window position changed |
| `CloseRequested` | `api: CloseRequestApi` | Close requested; call `api.prevent_close()` to cancel |
| `Destroyed` | -- | Window has been destroyed |
| `Focused` | `bool` | `true` = gained focus, `false` = lost focus |
| `ScaleFactorChanged` | `scale_factor: f64, new_inner_size: PhysicalSize<u32>` | DPI/display scale changed |
| `DragDrop` | `DragDropEvent` | File drag-and-drop event |
| `ThemeChanged` | `Theme` | System theme changed (not supported on Linux) |

```rust
.on_window_event(|window, event| {
    match event {
        WindowEvent::CloseRequested { api, .. } => {
            // Prevent close and hide instead
            api.prevent_close();
            window.hide().unwrap();
        }
        WindowEvent::Focused(focused) => {
            if *focused {
                println!("{} gained focus", window.label());
            }
        }
        _ => {}
    }
})
```

#### Multi-Window Pattern

```rust
.setup(|app| {
    // Main window is typically created from tauri.conf.json
    // Additional windows created programmatically:
    let _settings = WebviewWindowBuilder::new(
        app,
        "settings",
        WebviewUrl::App("settings.html".into()),
    )
    .title("Settings")
    .inner_size(600.0, 400.0)
    .visible(false) // hidden by default
    .build()?;

    Ok(())
})
```

Show/hide from commands:

```rust
#[tauri::command]
fn show_settings(app: tauri::AppHandle) {
    if let Some(window) = app.get_webview_window("settings") {
        window.show().unwrap();
        window.set_focus().unwrap();
    }
}
```

### 2.7 Menu & Tray (Rust Side)

For the frontend perspective on menus and tray, see [Section 3.6](#36-menu-api) and [Section 3.7](#37-tray-icon-api).

#### Application Menu

```rust
use tauri::menu::{MenuBuilder, SubmenuBuilder, PredefinedMenuItem};

.menu(|app| {
    let file_menu = SubmenuBuilder::new(app, "File")
        .text("new", "New")
        .text("open", "Open")
        .separator()
        .quit()
        .build()?;

    let edit_menu = SubmenuBuilder::new(app, "Edit")
        .undo()
        .redo()
        .separator()
        .cut()
        .copy()
        .paste()
        .select_all()
        .build()?;

    MenuBuilder::new(app)
        .item(&file_menu)
        .item(&edit_menu)
        .build()
})
```

#### Menu Event Handling

```rust
.on_menu_event(|app, event| {
    match event.id().as_ref() {
        "new" => {
            println!("New file requested");
        }
        "open" => {
            println!("Open file requested");
        }
        _ => {}
    }
})
```

#### Menu Item Types

| Type | Purpose |
|------|---------|
| `MenuItem` / `MenuItemBuilder` | Basic text menu item |
| `CheckMenuItem` / `CheckMenuItemBuilder` | Toggleable checkbox item |
| `IconMenuItem` / `IconMenuItemBuilder` | Item with icon |
| `PredefinedMenuItem` | OS-native items: `quit()`, `about()`, `copy()`, etc. |
| `Submenu` / `SubmenuBuilder` | Nested menu container |

#### System Tray

```rust
use tauri::tray::TrayIconBuilder;
use tauri::menu::MenuBuilder;
use tauri::image::Image;

.setup(|app| {
    let menu = MenuBuilder::new(app)
        .text("show", "Show Window")
        .text("hide", "Hide Window")
        .separator()
        .text("quit", "Quit")
        .build()?;

    let _tray = TrayIconBuilder::new()
        .icon(Image::from_path("icons/tray.png")?)
        .tooltip("My Tauri App")
        .menu(&menu)
        .show_menu_on_left_click(true)
        .on_menu_event(|app, event| {
            match event.id().as_ref() {
                "show" => {
                    if let Some(w) = app.get_webview_window("main") {
                        w.show().unwrap();
                    }
                }
                "quit" => app.exit(0),
                _ => {}
            }
        })
        .on_tray_icon_event(|tray, event| {
            println!("Tray event: {:?}", event);
        })
        .build(app)?;

    Ok(())
})
```

##### TrayIconBuilder Methods

```rust
TrayIconBuilder::new()              // create builder
    .with_id("my-tray")            // set custom ID
    .icon(Image)                    // set icon
    .tooltip("text")                // set tooltip
    .title("title")                 // set title
    .menu(&menu)                    // attach context menu
    .show_menu_on_left_click(bool)  // default: true
    .icon_as_template(bool)         // macOS only
    .on_menu_event(F)               // menu click handler
    .on_tray_icon_event(F)          // tray icon click handler
    .build(manager)?                // build (requires Manager)
```

### 2.8 IPC Channels (Advanced)

Channels enable **streaming** data from Rust to JavaScript. Unlike commands that return a single response, channels allow sending multiple messages over time.

For the frontend perspective, see [Section 3.1](#31-core-invoke-api) (Channels subsection).

#### The Channel Type

```rust
pub struct Channel<TSend = InvokeResponseBody> { /* ... */ }

impl<TSend: IpcResponse> Channel<TSend> {
    pub fn new<F>(on_message: F) -> Self
    where F: Fn(InvokeResponseBody) -> Result<()> + Send + Sync + 'static;

    pub fn id(&self) -> u32;
    pub fn send(&self, data: TSend) -> Result<()>;
}
```

Channel is `Clone + Send + Sync`, making it safe to use across threads.

#### Streaming Pattern

```rust
use tauri::ipc::Channel;

#[derive(Clone, serde::Serialize)]
struct DownloadProgress {
    bytes_read: u64,
    total: u64,
}

#[tauri::command]
async fn download_file(
    url: String,
    on_progress: Channel<DownloadProgress>,
) -> Result<Vec<u8>, String> {
    let response = reqwest::get(&url).await.map_err(|e| e.to_string())?;
    let total = response.content_length().unwrap_or(0);
    let mut bytes = Vec::new();

    let mut bytes_read = 0u64;
    // ... read chunks and send progress
    on_progress.send(DownloadProgress { bytes_read, total }).unwrap();

    Ok(bytes)
}
```

JavaScript side:

```typescript
import { invoke, Channel } from '@tauri-apps/api/core';

const onProgress = new Channel<{ bytes_read: number; total: number }>();
onProgress.onmessage = (progress) => {
    console.log(`${progress.bytes_read}/${progress.total}`);
};

const data = await invoke('download_file', {
    url: 'https://example.com/file.zip',
    onProgress,
});
```

#### Binary Data Streaming

For large binary payloads, use `Channel<&[u8]>` to stream chunks:

```rust
#[tauri::command]
async fn load_image(
    path: std::path::PathBuf,
    reader: Channel<&[u8]>,
) {
    let mut file = tokio::fs::File::open(path).await.unwrap();
    let mut chunk = vec![0u8; 4096];
    loop {
        let len = file.read(&mut chunk).await.unwrap();
        if len == 0 { break; }
        reader.send(&chunk[..len]).unwrap();
    }
}
```

#### Channel vs Events

| Feature | Channel | Events |
|---------|---------|--------|
| Direction | Rust to JS (response stream) | Bidirectional |
| Lifetime | Tied to command invocation | App lifetime |
| Targeting | Specific caller | Broadcast or filtered |
| Use case | Progress, streaming data | App-wide notifications |

---

## 3. Frontend / TypeScript API

> Based on `@tauri-apps/api` v2.x. Sources: official Tauri v2 documentation, npm registry, GitHub source.

### 3.1 Core invoke() API

**Import:** `import { invoke } from '@tauri-apps/api/core'`

The `invoke()` function is the primary mechanism for calling Rust backend commands from the frontend.

#### Function Signature

```typescript
invoke<T>(cmd: string, args?: InvokeArgs, options?: InvokeOptions): Promise<T>
```

**Type definitions:**

```typescript
type InvokeArgs = Record<string, unknown> | number[] | ArrayBuffer | Uint8Array;

interface InvokeOptions {
  headers?: HeadersInit;
}
```

#### Basic Usage

```typescript
import { invoke } from '@tauri-apps/api/core';

// Simple call, no arguments
await invoke('my_custom_command');

// With arguments — keys are camelCase, matching Rust snake_case params
const result = await invoke<string>('greet', { name: 'World' });
console.log(result); // "Hello, World!"
```

**Critical rule:** Argument keys must be **camelCase** in JavaScript. Tauri automatically converts them to **snake_case** for the Rust side. If the Rust command uses `#[tauri::command(rename_all = "snake_case")]`, you pass snake_case keys from JS instead.

#### Argument Passing

```typescript
// Frontend side
const msg = await invoke<string>('greet', { name: 'Alice', age: 30 });
```

For raw binary data, pass `ArrayBuffer` or `Uint8Array` directly:

```typescript
const data = new Uint8Array([1, 2, 3]);
await invoke('upload', data, {
  headers: { Authorization: 'Bearer token123' },
});
```

#### Return Values and TypeScript Typing

The generic parameter `<T>` types the resolved Promise value:

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

const user = await invoke<User>('get_user', { userId: 42 });
```

#### Error Handling

When a Rust command returns `Result<T, E>`, the `Err` variant causes the Promise to reject:

```typescript
try {
  const token = await invoke<string>('login', {
    user: 'admin',
    password: 'secret',
  });
  console.log('Logged in:', token);
} catch (error) {
  // error is the serialized Err value (string or structured object)
  console.error('Login failed:', error);
}
```

#### Channels (Streaming Data)

The `Channel` class enables streaming data from Rust to the frontend:

```typescript
import { invoke, Channel } from '@tauri-apps/api/core';

const onChunk = new Channel<Uint8Array>();
onChunk.onmessage = (chunk) => {
  console.log('Received chunk:', chunk.length, 'bytes');
};

await invoke('load_image', { path: '/photo.jpg', reader: onChunk });
```

#### Other Core Exports

| Export | Signature | Description |
|--------|-----------|-------------|
| `Channel<T>` | `new Channel<T>(onmessage?)` | Streaming data channel |
| `Resource` | `new Resource(rid)` | Base class for native resources; call `.close()` to release |
| `PluginListener` | class | Manages plugin event subscriptions; call `.unregister()` |
| `transformCallback<T>` | `(cb?, once?) => number` | Low-level callback registration (internal use) |
| `convertFileSrc` | `(filePath, protocol?) => string` | Converts native paths to asset URLs |
| `addPluginListener<T>` | `(plugin, event, cb) => Promise<PluginListener>` | Subscribe to plugin events |
| `checkPermissions<T>` | `(plugin) => Promise<T>` | Query plugin permission state |
| `requestPermissions<T>` | `(plugin) => Promise<T>` | Request plugin permissions |
| `isTauri` | `() => boolean` | Returns `true` when running inside a Tauri webview |

### 3.2 Event System (Frontend Side)

**Import:** `import { listen, emit, emitTo, once } from '@tauri-apps/api/event'`

The event system provides a loosely-coupled messaging layer using JSON payloads. It complements `invoke()` for broadcast communication or Rust-initiated messages. For the Rust perspective, see [Section 2.3](#23-event-system-rust-side).

#### listen

```typescript
listen<T>(event: EventName, handler: EventCallback<T>, options?: Options): Promise<UnlistenFn>
```

```typescript
import { listen } from '@tauri-apps/api/event';

interface DownloadProgress {
  url: string;
  bytesReceived: number;
  totalBytes: number;
}

const unlisten = await listen<DownloadProgress>('download-progress', (event) => {
  console.log(`Progress: ${event.payload.bytesReceived}/${event.payload.totalBytes}`);
});

// Later, when done:
unlisten();
```

#### once

```typescript
once<T>(event: EventName, handler: EventCallback<T>, options?: Options): Promise<UnlistenFn>
```

```typescript
import { once } from '@tauri-apps/api/event';

await once<string>('app-ready', (event) => {
  console.log('App initialized:', event.payload);
});
```

#### emit

```typescript
emit(event: EventName, payload?: unknown): Promise<void>
```

Broadcasts an event to all listeners (frontend and backend):

```typescript
import { emit } from '@tauri-apps/api/event';

await emit('file-selected', { path: '/documents/report.pdf' });
```

#### emitTo

```typescript
emitTo(target: string | EventTarget, event: EventName, payload?: unknown): Promise<void>
```

Sends an event to a specific target:

```typescript
import { emitTo } from '@tauri-apps/api/event';

await emitTo('settings', 'settings-update-requested', {
  key: 'notification',
  value: 'all',
});
```

#### Types

```typescript
type EventName = string; // alphanumeric, hyphens, slashes, colons, underscores
type EventCallback<T> = (event: Event<T>) => void;
type UnlistenFn = () => void;

interface Event<T> {
  event: EventName;
  id: number;
  payload: T;
}

interface Options {
  target?: string | EventTarget; // defaults to 'Any'
}

type EventTarget =
  | { kind: 'Any' }
  | { kind: 'AnyLabel'; label: string }
  | { kind: 'Window'; label: string }
  | { kind: 'Webview'; label: string }
  | { kind: 'WebviewWindow'; label: string };
```

#### TauriEvent Enum (Built-in Events)

```typescript
import { TauriEvent } from '@tauri-apps/api/event';
```

| Member | Value |
|--------|-------|
| `WINDOW_RESIZED` | `"tauri://resize"` |
| `WINDOW_MOVED` | `"tauri://move"` |
| `WINDOW_CLOSE_REQUESTED` | `"tauri://close-requested"` |
| `WINDOW_DESTROYED` | `"tauri://destroyed"` |
| `WINDOW_FOCUS` | `"tauri://focus"` |
| `WINDOW_BLUR` | `"tauri://blur"` |
| `WINDOW_SCALE_FACTOR_CHANGED` | `"tauri://scale-change"` |
| `WINDOW_THEME_CHANGED` | `"tauri://theme-changed"` |
| `WINDOW_CREATED` | `"tauri://window-created"` |
| `WEBVIEW_CREATED` | `"tauri://webview-created"` |
| `DRAG_ENTER` | `"tauri://drag-enter"` |
| `DRAG_OVER` | `"tauri://drag-over"` |
| `DRAG_DROP` | `"tauri://drag-drop"` |
| `DRAG_LEAVE` | `"tauri://drag-leave"` |

#### Event Cleanup Pattern

Always clean up listeners, especially in component-based frameworks:

```typescript
// React example
useEffect(() => {
  let unlisten: (() => void) | undefined;

  listen<string>('update', (e) => {
    setState(e.payload);
  }).then((fn) => { unlisten = fn; });

  return () => {
    if (unlisten) unlisten();
  };
}, []);
```

### 3.3 Window API

**Import:** `import { Window, getCurrentWindow, getAllWindows } from '@tauri-apps/api/window'`

#### Getting Window References

```typescript
import { getCurrentWindow, Window } from '@tauri-apps/api/window';

// Current window
const win = getCurrentWindow();

// By label
const settings = await Window.getByLabel('settings');

// All windows
const allWindows = await getAllWindows();

// Focused window
const focused = await Window.getFocusedWindow();
```

#### Window Manipulation Methods

All methods return `Promise<void>` unless noted.

**Position and Size:**

```typescript
const win = getCurrentWindow();
await win.center();
await win.setPosition(new LogicalPosition(100, 100));
await win.setSize(new LogicalSize(800, 600));
const innerSize = await win.innerSize();   // PhysicalSize
const outerSize = await win.outerSize();   // PhysicalSize
const innerPos = await win.innerPosition(); // PhysicalPosition
const outerPos = await win.outerPosition(); // PhysicalPosition
```

**Visibility and State:**

```typescript
await win.show();
await win.hide();
await win.maximize();
await win.minimize();
await win.unmaximize();
await win.unminimize();
await win.toggleMaximize();
await win.setFullscreen(true);
const isMax = await win.isMaximized();    // boolean
const isMin = await win.isMinimized();    // boolean
const isVis = await win.isVisible();      // boolean
const isFull = await win.isFullscreen();  // boolean
```

**Properties:**

```typescript
await win.setTitle('My App - Document.txt');
const title = await win.title();
await win.setIcon('/path/to/icon.png');
await win.setDecorations(false);
await win.setAlwaysOnTop(true);
await win.setResizable(false);
await win.setClosable(false);
```

**Lifecycle:**

```typescript
await win.close();    // Triggers close-requested event first
await win.destroy();  // Force-closes without events
```

#### Window Events

```typescript
const win = getCurrentWindow();

// Close requested (can be prevented)
await win.onCloseRequested(async (event) => {
  const confirmed = await confirm('Are you sure?');
  if (!confirmed) {
    event.preventDefault();
  }
});

// Other window events
await win.onResized(({ payload: size }) => console.log('New size:', size));
await win.onMoved(({ payload: pos }) => console.log('New position:', pos));
await win.onFocusChanged(({ payload: focused }) => console.log('Focused:', focused));
await win.onThemeChanged(({ payload: theme }) => console.log('Theme:', theme));
await win.onScaleChanged(({ payload }) => console.log('Scale:', payload.scaleFactor));
await win.onDragDropEvent((event) => console.log('Drag/drop:', event));
```

#### Creating New Windows

```typescript
import { Window } from '@tauri-apps/api/window';

const webview = new Window('settings-window', {
  url: '/settings',
  title: 'Settings',
  width: 600,
  height: 400,
  center: true,
  decorations: true,
  resizable: true,
});
```

#### Key Enums

```typescript
type CursorIcon = 'default' | 'crosshair' | 'hand' | 'move' | 'text' | 'wait' | ...;
type Theme = 'light' | 'dark';
type TitleBarStyle = 'visible' | 'transparent' | 'overlay';
enum UserAttentionType { Critical = 1, Informational = 2 }
enum ProgressBarStatus { None, Normal, Indeterminate, Paused, Error }
```

#### Monitor Information

```typescript
import { availableMonitors, currentMonitor, primaryMonitor, cursorPosition } from '@tauri-apps/api/window';

const monitors = await availableMonitors();
const current = await currentMonitor();
const primary = await primaryMonitor();
const cursor = await cursorPosition();
```

### 3.4 Webview API

**Import:** `import { Webview, getCurrentWebview, getAllWebviews } from '@tauri-apps/api/webview'`

The Webview API provides finer control than the Window API, allowing multiple webviews inside a single window.

```typescript
import { getCurrentWebview } from '@tauri-apps/api/webview';

const webview = getCurrentWebview();

// Content
await webview.setZoom(1.5);
await webview.clearAllBrowsingData();

// Sizing
const size = await webview.size();
const pos = await webview.position();
await webview.setSize(new LogicalSize(400, 300));
await webview.setPosition(new LogicalPosition(0, 0));
await webview.setAutoResize(true);

// Visibility
await webview.show();
await webview.hide();
await webview.setFocus();

// Reparenting
await webview.reparent('other-window');

// Events
await webview.listen<string>('custom-event', (event) => {
  console.log(event.payload);
});
await webview.onDragDropEvent((event) => {
  console.log('Drop event:', event.payload);
});

// Lifecycle
await webview.close();
```

#### Creating a Webview Inside a Window

```typescript
import { Webview } from '@tauri-apps/api/webview';
import { getCurrentWindow } from '@tauri-apps/api/window';

const webview = new Webview(getCurrentWindow(), 'my-webview', {
  url: 'https://example.com',
  x: 0,
  y: 0,
  width: 400,
  height: 300,
});
```

### 3.5 Path API

**Import:** `import { appDataDir, join, resolve, ... } from '@tauri-apps/api/path'`

All path functions are async and return `Promise<string>`.

#### Directory Functions

```typescript
import {
  appDataDir, appConfigDir, appLocalDataDir, appCacheDir, appLogDir,
  audioDir, cacheDir, configDir, dataDir, desktopDir, documentDir,
  downloadDir, executableDir, fontDir, homeDir, localDataDir,
  pictureDir, publicDir, resourceDir, runtimeDir, tempDir,
  templateDir, videoDir,
} from '@tauri-apps/api/path';

const dataPath = await appDataDir();     // e.g., "C:\Users\X\AppData\Roaming\com.app"
const configPath = await appConfigDir(); // e.g., "C:\Users\X\AppData\Roaming\com.app"
const homePath = await homeDir();        // e.g., "C:\Users\X"
```

#### Path Manipulation

```typescript
import { join, resolve, dirname, basename, extname, normalize, isAbsolute, sep, delimiter } from '@tauri-apps/api/path';

const full = await join(await appDataDir(), 'databases', 'app.db');
const abs = await resolve('relative', 'path', 'file.txt');
const dir = await dirname('/path/to/file.txt');   // "/path/to"
const base = await basename('/path/to/file.txt'); // "file.txt"
const ext = await extname('document.pdf');         // "pdf"
const norm = await normalize('/path/./to/../file');
const isAbs = await isAbsolute('/usr/bin');         // true
const separator = sep();   // "\\" on Windows, "/" on Unix
const delim = delimiter(); // ";" on Windows, ":" on Unix
```

#### Resource Resolution

```typescript
import { resolveResource } from '@tauri-apps/api/path';

const modelPath = await resolveResource('models/default.bin');
```

#### BaseDirectory Enum

```typescript
import { BaseDirectory } from '@tauri-apps/api/path';

// Key members:
BaseDirectory.AppData      // 14
BaseDirectory.AppConfig    // 13
BaseDirectory.AppLocalData // 15
BaseDirectory.AppCache     // 16
BaseDirectory.AppLog       // 17
BaseDirectory.Home         // 21
BaseDirectory.Desktop      // 18
BaseDirectory.Document     // 6
BaseDirectory.Download     // 7
BaseDirectory.Resource     // 11
BaseDirectory.Temp         // 12
```

### 3.6 Menu API

**Import:** `import { Menu, MenuItem, Submenu, CheckMenuItem, PredefinedMenuItem } from '@tauri-apps/api/menu'`

#### Creating Menus

```typescript
import { Menu, MenuItem, Submenu, PredefinedMenuItem, CheckMenuItem } from '@tauri-apps/api/menu';

const menu = await Menu.new({
  items: [
    await Submenu.new({
      text: 'File',
      items: [
        await MenuItem.new({
          text: 'Open',
          accelerator: 'CmdOrCtrl+O',
          action: () => { console.log('Open clicked'); },
        }),
        await MenuItem.new({
          text: 'Save',
          accelerator: 'CmdOrCtrl+S',
          action: () => { console.log('Save clicked'); },
        }),
        await PredefinedMenuItem.new({ item: 'Separator' }),
        await PredefinedMenuItem.new({ item: 'Quit' }),
      ],
    }),
    await Submenu.new({
      text: 'View',
      items: [
        await CheckMenuItem.new({
          text: 'Dark Mode',
          checked: false,
          action: (item) => { console.log('Toggled'); },
        }),
      ],
    }),
  ],
});

// Set as application menu
await menu.setAsAppMenu();

// Or set for a specific window
await menu.setAsWindowMenu();
```

#### Context Menus

```typescript
await menu.popup();
await menu.popup({ x: 100, y: 200 });
```

#### Menu Item Management

```typescript
await menu.append(await MenuItem.new({ text: 'New Item', action: () => {} }));
await menu.prepend(await MenuItem.new({ text: 'First Item', action: () => {} }));
await menu.insert(1, await MenuItem.new({ text: 'At Index 1', action: () => {} }));
await menu.remove('item-id');
await menu.removeAt(0);

const item = await menu.get('item-id');
const allItems = await menu.items();
```

#### PredefinedMenuItem Types

Platform-standard items: `About`, `Hide`, `HideOthers`, `ShowAll`, `CloseWindow`, `Quit`, `Copy`, `Cut`, `Paste`, `SelectAll`, `Undo`, `Redo`, `Minimize`, `Zoom`, `Separator`, `Fullscreen`, `Services`, `BringAllToFront`.

### 3.7 Tray Icon API

**Import:** `import { TrayIcon } from '@tauri-apps/api/tray'`

```typescript
import { TrayIcon } from '@tauri-apps/api/tray';
import { Menu, MenuItem } from '@tauri-apps/api/menu';

const menu = await Menu.new({
  items: [
    await MenuItem.new({ text: 'Show', action: () => showWindow() }),
    await MenuItem.new({ text: 'Quit', action: () => exit(0) }),
  ],
});

const tray = await TrayIcon.new({
  icon: 'icons/tray-icon.png',
  tooltip: 'My App',
  menu,
  action: (event) => {
    if (event.type === 'Click') {
      console.log('Tray clicked with', event.button);
    }
  },
});

// Update at runtime
await tray.setTooltip('Updated tooltip');
await tray.setIcon('icons/new-icon.png');
await tray.setVisible(false);
```

**Event types:** `Click`, `DoubleClick`, `Enter`, `Move`, `Leave`
**Mouse buttons:** `Left`, `Right`, `Middle`

### 3.8 Utilities

#### convertFileSrc

**Import:** `import { convertFileSrc } from '@tauri-apps/api/core'`

Converts a native file path to a URL that can be used in `<img>`, `<video>`, `<audio>`, or CSS:

```typescript
import { convertFileSrc } from '@tauri-apps/api/core';

const assetUrl = convertFileSrc('/path/to/image.png');
// Returns: "https://asset.localhost/path/to/image.png" (on most platforms)
```

Usage in HTML:

```typescript
const imgSrc = convertFileSrc(await join(await appDataDir(), 'photos', 'avatar.jpg'));
document.getElementById('avatar').src = imgSrc;
```

Custom protocol:

```typescript
const url = convertFileSrc('/path/to/file', 'my-protocol');
// Returns: "my-protocol://localhost/path/to/file"
```

#### isTauri

```typescript
import { isTauri } from '@tauri-apps/api/core';

if (isTauri()) {
  const data = await invoke('get_data');
} else {
  const data = await fetch('/api/data').then(r => r.json());
}
```

#### Asset Protocol

Tauri serves local files through the `asset://` protocol. Configured in `tauri.conf.json`:

```json
{
  "app": {
    "security": {
      "assetProtocol": {
        "enable": true,
        "scope": ["$APPDATA/**", "$RESOURCE/**"]
      }
    }
  }
}
```

#### Global Tauri Object

When `app.withGlobalTauri` is set to `true` in `tauri.conf.json`, all APIs are available on `window.__TAURI__`:

```typescript
const { invoke } = window.__TAURI__.core;
const { listen, emit } = window.__TAURI__.event;
const { getCurrentWindow } = window.__TAURI__.window;
```

### 3.9 Testing and Mocking

**Import:** `import { mockIPC, mockWindows, clearMocks, mockConvertFileSrc } from '@tauri-apps/api/mocks'`

#### mockIPC

```typescript
import { mockIPC, clearMocks } from '@tauri-apps/api/mocks';
import { invoke } from '@tauri-apps/api/core';

mockIPC((cmd, args) => {
  if (cmd === 'greet') {
    return `Hello, ${args.name}!`;
  }
  if (cmd === 'get_count') {
    return 42;
  }
  throw new Error(`Unknown command: ${cmd}`);
});

const result = await invoke<string>('greet', { name: 'Test' });
expect(result).toBe('Hello, Test!');

clearMocks();
```

With event mocking:

```typescript
mockIPC(
  (cmd, args) => { /* handle commands */ },
  { shouldMockEvents: true }
);
```

#### mockWindows

```typescript
import { mockWindows, clearMocks } from '@tauri-apps/api/mocks';
import { getCurrentWindow } from '@tauri-apps/api/window';

mockWindows('main', 'settings', 'about');

const win = getCurrentWindow();
expect(win.label).toBe('main');

clearMocks();
```

#### mockConvertFileSrc

```typescript
import { mockConvertFileSrc, clearMocks } from '@tauri-apps/api/mocks';

mockConvertFileSrc('linux');
clearMocks();
```

#### clearMocks

Always call in `afterEach` or equivalent teardown:

```typescript
afterEach(() => {
  clearMocks();
});
```

### 3.10 Package Information

- **Package:** `@tauri-apps/api`
- **Current major version:** 2.x
- **Installation:** `npm install @tauri-apps/api`
- **Submodule exports:**
  - `@tauri-apps/api/core` — invoke, Channel, Resource, convertFileSrc, isTauri
  - `@tauri-apps/api/event` — listen, emit, emitTo, once, TauriEvent
  - `@tauri-apps/api/window` — Window, getCurrentWindow, getAllWindows
  - `@tauri-apps/api/webview` — Webview, getCurrentWebview, getAllWebviews
  - `@tauri-apps/api/path` — Directory functions, path manipulation, BaseDirectory
  - `@tauri-apps/api/menu` — Menu, MenuItem, Submenu, CheckMenuItem, PredefinedMenuItem
  - `@tauri-apps/api/tray` — TrayIcon
  - `@tauri-apps/api/image` — Image handling
  - `@tauri-apps/api/dpi` — LogicalSize, PhysicalSize, LogicalPosition, PhysicalPosition
  - `@tauri-apps/api/mocks` — mockIPC, mockWindows, clearMocks, mockConvertFileSrc

---

## 4. Configuration

### 4.1 tauri.conf.json Structure

The main configuration file lives at `src-tauri/tauri.conf.json`. Tauri generates JSON schemas in `gen/schemas/` for IDE autocompletion.

#### Complete Example

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

#### Top-Level Keys

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

#### Build Section

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

#### App Section

| Property | Type | Description |
|----------|------|-------------|
| `windows` | `WindowConfig[]` | Windows created at startup. Default: `[]`. |
| `security` | `object` | Security settings (CSP, capabilities, asset protocol). |
| `trayIcon` | `TrayIconConfig \| null` | System tray icon configuration. |
| `withGlobalTauri` | `boolean` | Inject Tauri API on `window.__TAURI__`. |
| `enableGTKAppId` | `boolean` | Set identifier as GTK app ID. |
| `macOSPrivateApi` | `boolean` | Enable transparent background and fullscreen on macOS. |

#### Window Configuration (`app.windows[]`)

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `label` | `string` | `"main"` | Unique window identifier |
| `title` | `string` | -- | Window title bar text |
| `url` | `string` | `"/"` | URL or path to load |
| `width` | `number` | -- | Width in logical pixels |
| `height` | `number` | -- | Height in logical pixels |
| `x` / `y` | `number` | -- | Window position |
| `minWidth` / `minHeight` | `number` | -- | Minimum dimensions |
| `maxWidth` / `maxHeight` | `number` | -- | Maximum dimensions |
| `resizable` | `boolean` | `true` | Allow resizing |
| `fullscreen` | `boolean` | `false` | Start fullscreen |
| `focus` | `boolean` | `true` | Focus on creation |
| `create` | `boolean` | `true` | Create at startup (set `false` for programmatic creation) |
| `transparent` | `boolean` | `false` | Enable transparency |
| `decorations` | `boolean` | `true` | Show title bar / borders |
| `alwaysOnTop` | `boolean` | `false` | Stay above other windows |

#### Bundle Section

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

#### Platform-Specific Bundle Configuration

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

#### Minimal Viable tauri.conf.json

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

---

## 5. Security & Permissions

### 5.1 Permissions System (New in v2)

The permissions system replaces the Tauri v1 allowlist. It provides fine-grained control over which IPC commands the frontend can invoke.

#### Core Concepts

- **Permissions** define which commands are allowed or denied
- **Scopes** restrict *what data* those commands can access (e.g., file paths, URLs)
- **Capabilities** group permissions and assign them to specific windows/webviews

#### Permission Format

Permissions follow the naming convention:
- `<plugin-name>:default` — Default permission set for a plugin
- `<plugin-name>:allow-<command>` — Allow a specific command
- `<plugin-name>:deny-<command>` — Deny a specific command
- `<plugin-name>:<custom-set>` — Custom permission set

The system auto-prepends `tauri-plugin-` to plugin identifiers at compile time. Identifiers are limited to lowercase ASCII `[a-z]`, max 116 characters.

#### Defining Permissions (TOML format)

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

#### Permission Sets

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

#### Custom Command Permissions

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

#### File Structure

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

### 5.2 Capabilities System

Capabilities tie permissions to specific windows and webviews, forming the access control bridge.

#### Capability File Structure

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

#### Key Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `identifier` | `string` | Yes | Unique capability name |
| `description` | `string` | No | Purpose description |
| `windows` | `string[]` | Yes | Window labels to apply to (supports `"*"` wildcard) |
| `permissions` | `string[]` | Yes | Array of permission identifiers |
| `platforms` | `string[]` | No | Target OS: `"linux"`, `"macOS"`, `"windows"`, `"iOS"`, `"android"` |
| `remote` | `object` | No | Remote URL access configuration |

#### Platform-Specific Capabilities

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

#### Remote API Access

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

#### Inline Capabilities in tauri.conf.json

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

#### Window Merging Behavior

Windows that appear in multiple capabilities merge all permissions from all involved capabilities. This is additive — there is no override or conflict resolution.

#### Schema Generation

Tauri generates platform-specific JSON schemas in `gen/schemas/`:
- `desktop-schema.json`
- `mobile-schema.json`
- `remote-schema.json`

Reference them via `$schema` for IDE autocompletion.

#### Minimal Viable Capability File

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

### 5.3 Content Security Policy (CSP)

Tauri applies a Content Security Policy to restrict resource loading in the webview.

#### Configuration

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

#### Tauri-Specific Protocols

| Protocol | Purpose | CSP Usage |
|----------|---------|-----------|
| `tauri:` | Internal Tauri protocol for serving frontend assets | Used in `default-src` |
| `asset:` / `https://asset.localhost` | Access bundled resources and filesystem assets | Used in `img-src`, `media-src` |
| `ipc:` / `http://ipc.localhost` | IPC communication channel between frontend and Rust | Used in `connect-src` |

#### Security Settings

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

#### Common CSP Directives for Tauri

```
default-src 'self'
script-src 'self'
style-src 'self' 'unsafe-inline'
img-src 'self' asset: https://asset.localhost data:
font-src 'self' data:
connect-src ipc: http://ipc.localhost https://api.example.com
media-src 'self' asset: https://asset.localhost
```

#### Windows URL Change (v2)

On Windows, production frontend files now load from `http://tauri.localhost` instead of `https://tauri.localhost`. This change resets IndexedDB, LocalStorage, and Cookies unless:
- `dangerousUseHttpScheme` was enabled in v1
- `app.windows[].useHttpsScheme` is set to `true` in v2

---

## 6. Build & Distribution

### 6.1 Build Commands

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

### 6.2 Platform-Specific Bundlers

| Platform | Formats | Notes |
|----------|---------|-------|
| **Windows** | NSIS installer, MSI (WiX), portable exe | WebView2 installation mode configurable |
| **macOS** | App bundle (.app), DMG | Code signing + notarization required for distribution |
| **Linux** | AppImage, .deb, .rpm, Snap, Flatpak, AUR | AppImage is the most universal |
| **Android** | APK, AAB | Play Store requires AAB |
| **iOS** | IPA | App Store or TestFlight distribution |

### 6.3 Resource Bundling

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

### 6.4 Code Signing

#### Windows (Authenticode)

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
- `WINDOWS_CERTIFICATE` -- Base64-encoded `.pfx` file
- `WINDOWS_CERTIFICATE_PASSWORD` -- Export password

#### macOS (Apple Developer ID + Notarization)

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
- `APPLE_CERTIFICATE` -- Base64-encoded `.p12`
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

#### Linux

GPG signatures for packages. Platform-specific package managers handle verification.

### 6.5 Auto-Updater

#### Setup

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

#### Key Generation

```bash
npm run tauri signer generate -- -w ~/.tauri/myapp.key
```

This creates a public/private keypair. **Never share the private key.** Losing it prevents all future updates to existing installations.

Set build environment variables:

```bash
export TAURI_SIGNING_PRIVATE_KEY="path or content"
export TAURI_SIGNING_PRIVATE_KEY_PASSWORD=""
```

#### Configuration

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
- `{{current_version}}` -- Current app version
- `{{target}}` -- OS (`linux`, `windows`, `darwin`)
- `{{arch}}` -- Architecture (`x86_64`, `i686`, `aarch64`, `armv7`)

**Windows install modes:**
- `"passive"` -- No user interaction (default)
- `"basicUi"` -- Requires user interaction
- `"quiet"` -- No feedback at all

#### Server Response Format

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

#### JavaScript Update Check

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

#### Rust Update Check

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

#### Generated Artifacts

| Platform | Update Artifact | Signature |
|----------|----------------|-----------|
| Linux | `.AppImage.tar.gz` | `.AppImage.tar.gz.sig` |
| macOS | `.app.tar.gz` | `.app.tar.gz.sig` |
| Windows | `.exe` or `.msi` | `.exe.sig` or `.msi.sig` |

### 6.6 CI/CD Patterns

#### GitHub Actions with `tauri-action`

The official `tauri-apps/tauri-action` automates cross-platform builds and releases.

#### Cross-Platform Build Matrix

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

#### Key Configuration Notes

- `__VERSION__` is auto-replaced with the app version from `tauri.conf.json`
- `releaseDraft: true` creates draft releases for review before publishing
- Linux dependencies (`libwebkit2gtk-4.1-dev`, `libappindicator3-dev`, `librsvg2-dev`) must be installed explicitly
- GitHub token permissions must be set to "Read and write" in repository settings
- ARM Linux builds: use `ubuntu-22.04-arm` or `ubuntu-24.04-arm` runners (GA since August 2025)

#### Windows Code Signing in CI

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

#### macOS Code Signing in CI

Required secrets:
- `APPLE_CERTIFICATE` (base64 `.p12`)
- `APPLE_CERTIFICATE_PASSWORD`
- `APPLE_SIGNING_IDENTITY`
- `KEYCHAIN_PASSWORD`
- `APPLE_API_ISSUER`, `APPLE_API_KEY`, `APPLE_API_KEY_PATH` (for notarization)

---

## 7. Mobile Development

### 7.1 Prerequisites

**Android:**
- Android Studio with SDK Platform, Platform-Tools, NDK, Build-Tools, Command-line Tools
- Environment variables: `JAVA_HOME`, `ANDROID_HOME`, `NDK_HOME`
- Rust targets: `aarch64-linux-android`, `armv7-linux-androideabi`, `i686-linux-android`, `x86_64-linux-android`

**iOS (macOS only):**
- Full Xcode installation (not just Command Line Tools)
- Cocoapods (via Homebrew)
- Rust targets: `aarch64-apple-ios`, `x86_64-apple-ios`, `aarch64-apple-ios-sim`

### 7.2 Commands

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

### 7.3 Cargo.toml for Mobile Support

```toml
[lib]
name = "app_lib"
crate-type = ["staticlib", "cdylib", "rlib"]
```

### 7.4 Entry Point for Mobile

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

### 7.5 Platform-Specific Rust Code

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

### 7.6 Bundle Configuration

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

## 8. Migration v1 to v2

### 8.1 Configuration Restructuring

| Tauri v1 | Tauri v2 |
|----------|----------|
| `package.productName` | `productName` (top-level) |
| `package.version` | `version` (top-level) |
| `tauri` | `app` |
| `tauri.allowlist` | Removed -- replaced by capabilities/permissions |
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

### 8.2 Allowlist to Permissions Migration

The v1 allowlist is completely replaced by the capability-based permission system. Use the migration CLI:

```bash
npx @tauri-apps/cli migrate
```

This auto-converts v1 allowlist entries to capability files in `src-tauri/capabilities/`.

### 8.3 Rust API Changes

**Type renames:**
- `Window` -> `WebviewWindow`
- `WindowBuilder` -> `WebviewWindowBuilder`
- `WindowUrl` -> `WebviewUrl`
- `Manager::get_window()` -> `Manager::get_webview_window()`

**Removed modules** (now plugins):
- `api::dialog` -> `tauri-plugin-dialog`
- `api::http` -> `tauri-plugin-http`
- `api::path` -> `Manager::path()` methods
- `api::process` -> `tauri-plugin-process`
- `api::shell` -> `tauri-plugin-shell`

**Removed manager methods:**
- `App::clipboard_manager()` -> `tauri-plugin-clipboard-manager`
- `App::get_cli_matches()` -> `tauri-plugin-cli`
- `App::global_shortcut_manager()` -> `tauri-plugin-global-shortcut`

### 8.4 JavaScript API Changes

**Module renames:**
- `@tauri-apps/api/tauri` -> `@tauri-apps/api/core`
- `@tauri-apps/api/window` -> `@tauri-apps/api/webviewWindow`

**All non-core modules removed** from the base `@tauri-apps/api` package. Install per-plugin packages:
- `@tauri-apps/plugin-dialog`
- `@tauri-apps/plugin-fs`
- `@tauri-apps/plugin-http`
- `@tauri-apps/plugin-shell`
- etc.

### 8.5 Event System Changes

- `emit()` now broadcasts to all listeners
- New `emit_to()` for targeted events
- `listen_global()` renamed to `listen_any()`
- `WebviewWindow.listen()` only receives events for that specific window

### 8.6 Menu System Overhaul

- `Menu` -> `MenuBuilder`
- `CustomMenuItem` -> `MenuItemBuilder`
- `Submenu` -> `SubmenuBuilder`
- `MenuItem` -> `PredefinedMenuItem`
- `SystemTray` -> `TrayIconBuilder`
- Event handling: `Builder::on_menu_event()` removed; use `App::on_menu_event()` or `AppHandle::on_menu_event()`

### 8.7 Feature Flag Changes

**Removed**: `reqwest-client`, `reqwest-native-tls-vendored`, `process-command-api`, `shell-open-api`, `windows7-compat`, `updater`, `linux-protocol-headers`, `system-tray`

**Added**: `linux-protocol-body`

### 8.8 Plugin Migration Table

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

## 9. Official Plugins Reference

### 9.1 Complete Plugin List

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

### 9.2 Core Permissions (built-in, no plugin needed)

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

### 9.3 Plugin Frontend API Details

#### File System (`@tauri-apps/plugin-fs`)

**Requires permissions:** `fs:allow-read-file`, `fs:allow-write-file`, `fs:allow-read-dir`, `fs:allow-mkdir`, `fs:allow-remove`, `fs:allow-exists`, `fs:allow-stat`, etc. Use `fs:default` for app-specific directory read access.

```typescript
import {
  readTextFile, writeTextFile, readFile, writeFile,
  readDir, mkdir, remove, rename, copyFile,
  exists, stat, lstat, truncate,
  create, open,
  readTextFileLines,
  watch, watchImmediate,
  BaseDirectory,
} from '@tauri-apps/plugin-fs';

// Read text file
const content = await readTextFile('config.toml', {
  baseDir: BaseDirectory.AppConfig,
});

// Write text file
await writeTextFile('config.json', JSON.stringify({ theme: 'dark' }), {
  baseDir: BaseDirectory.AppConfig,
});

// Read binary file
const bytes = await readFile('icon.png', { baseDir: BaseDirectory.Resource });

// File handle API
const file = await open('data.bin', {
  read: true, write: true, create: true,
  baseDir: BaseDirectory.AppData,
});
const stat_result = await file.stat();
await file.close();

// Stream large files line by line
const lines = await readTextFileLines('app.log', { baseDir: BaseDirectory.AppLog });
for await (const line of lines) {
  console.log(line);
}

// Directory operations
await mkdir('cache/images', { baseDir: BaseDirectory.AppCache });
const entries = await readDir('data', { baseDir: BaseDirectory.AppData });

// Check existence
if (await exists('token.json', { baseDir: BaseDirectory.AppLocalData })) {
  // ...
}

// Watch for changes (requires "watch" feature in Cargo.toml)
const stopWatching = await watch('app.log', (event) => {
  console.log('File changed:', event);
}, { baseDir: BaseDirectory.AppLog, delayMs: 500 });
```

**Security:** Paths must be relative to a `BaseDirectory` or created via the path API. Traversal patterns like `../` are rejected. Deny rules take precedence over allow rules.

#### Dialog (`@tauri-apps/plugin-dialog`)

**Requires permissions:** `dialog:default` or specific `dialog:allow-open`, `dialog:allow-save`, `dialog:allow-ask`, `dialog:allow-confirm`, `dialog:allow-message`.

```typescript
import { open, save, ask, confirm, message } from '@tauri-apps/plugin-dialog';

// File picker
const filePath = await open({
  multiple: false,
  directory: false,
  filters: [{ name: 'Images', extensions: ['png', 'jpg', 'gif'] }],
});

// Directory picker
const dirPath = await open({ directory: true });

// Multiple files
const files = await open({ multiple: true });

// Save dialog
const savePath = await save({
  filters: [{ name: 'Documents', extensions: ['pdf', 'docx'] }],
});

// Message dialogs
await message('Operation completed!', { title: 'Success', kind: 'info' });
const yes = await ask('Delete this file?', { title: 'Confirm', kind: 'warning' });
const ok = await confirm('Proceed?', { title: 'Action' });
```

**Platform note:** Linux/Windows/macOS return filesystem paths; iOS returns `file://` URIs; Android returns content URIs.

#### HTTP Client (`@tauri-apps/plugin-http`)

**Requires permissions:** URL scoping is required. Configure allowed URLs in capabilities.

```typescript
import { fetch } from '@tauri-apps/plugin-http';

const response = await fetch('https://api.example.com/data', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ key: 'value' }),
});

console.log(response.status);
const data = await response.json();
```

Permission configuration:

```json
{
  "identifier": "http:default",
  "allow": [{ "url": "https://api.example.com/*" }],
  "deny": [{ "url": "https://api.example.com/admin/*" }]
}
```

**Note:** Forbidden headers (like `Origin`) are silently dropped unless the `unsafe-headers` Cargo feature is enabled.

#### Notification (`@tauri-apps/plugin-notification`)

**Requires permissions:** `notification:default` or `notification:allow-notify`, `notification:allow-request-permission`, `notification:allow-is-permission-granted`.

```typescript
import {
  isPermissionGranted, requestPermission, sendNotification,
  createChannel, Importance, Visibility,
} from '@tauri-apps/plugin-notification';

let granted = await isPermissionGranted();
if (!granted) {
  const permission = await requestPermission();
  granted = permission === 'granted';
}

if (granted) {
  sendNotification({
    title: 'Download Complete',
    body: 'Your file has been downloaded successfully.',
  });
}

// Android: create notification channel
await createChannel({
  id: 'updates',
  name: 'App Updates',
  importance: Importance.High,
  visibility: Visibility.Public,
});
```

#### Shell (`@tauri-apps/plugin-shell`)

**Requires permissions:** `shell:allow-execute`, `shell:allow-spawn`, `shell:allow-kill`, `shell:allow-stdin-write`. Commands must be pre-defined in the permission scope.

```typescript
import { Command } from '@tauri-apps/plugin-shell';

const result = await Command.create('exec-sh', ['-c', 'echo "Hello"']).execute();
console.log(result.stdout);
console.log(result.status.code);

if (result.status.success) {
  const output = new TextDecoder().decode(result.stdout);
  console.log(output);
}
```

Permission configuration must whitelist specific commands:

```json
{
  "identifier": "shell:allow-execute",
  "allow": [{
    "name": "exec-sh",
    "cmd": "sh",
    "args": ["-c", { "validator": "\\S+" }],
    "sidecar": false
  }]
}
```

**Note:** `shell.open` has moved to the separate **Opener plugin** (`@tauri-apps/plugin-opener`).

#### Clipboard (`@tauri-apps/plugin-clipboard-manager`)

**Requires permissions:** Must explicitly grant `clipboard-manager:allow-read-text`, `clipboard-manager:allow-write-text`, etc.

```typescript
import { writeText, readText } from '@tauri-apps/plugin-clipboard-manager';

await writeText('Hello, clipboard!');
const content = await readText();
```

#### OS Information (`@tauri-apps/plugin-os`)

**Requires permissions:** `os:default`.

```typescript
import { platform, arch, type, version, family, hostname, locale, eol } from '@tauri-apps/plugin-os';

const os = await platform();  // "windows", "linux", "macos", "android", "ios"
const cpu = await arch();     // "x86_64", "aarch64", etc.
const osType = await type();  // "windows_nt", "linux", "darwin"
const ver = await version();  // e.g., "10.0.22621"
const fam = await family();   // "unix" or "windows"
const host = await hostname();
const loc = await locale();   // e.g., "en-US"
const lineEnd = eol();        // "\r\n" on Windows, "\n" on Unix
```

#### Process (`@tauri-apps/plugin-process`)

**Requires permissions:** `process:default`.

```typescript
import { exit, relaunch } from '@tauri-apps/plugin-process';

await exit(0);
await relaunch();
```

#### Updater (`@tauri-apps/plugin-updater`)

**Requires permissions:** `updater:default`. Individual: `updater:allow-check`, `updater:allow-download`, `updater:allow-install`, `updater:allow-download-and-install`.

```typescript
import { check } from '@tauri-apps/plugin-updater';
import { relaunch } from '@tauri-apps/plugin-process';

const update = await check();
if (update) {
  console.log(`Update available: ${update.version}`);
  await update.downloadAndInstall((event) => {
    switch (event.event) {
      case 'Started':
        console.log(`Downloading ${event.data.contentLength} bytes`);
        break;
      case 'Progress':
        console.log(`Chunk: ${event.data.chunkLength} bytes`);
        break;
      case 'Finished':
        console.log('Download complete');
        break;
    }
  });
  await relaunch();
}
```

**`check()` options:** `proxy?: string`, `timeout?: number`, `headers?: Record<string, string>`, `target?: string`.

#### Store (`@tauri-apps/plugin-store`)

**Requires permissions:** `store:default`.

```typescript
import { load, LazyStore } from '@tauri-apps/plugin-store';

// Eager loading
const store = await load('settings.json', { autoSave: true });

await store.set('theme', 'dark');
await store.set('user', { name: 'Alice', prefs: { lang: 'en' } });

const theme = await store.get<string>('theme');
const hasKey = await store.has('theme');
const allKeys = await store.keys();
const allValues = await store.values();
const allEntries = await store.entries();
const count = await store.length();

await store.delete('theme');
await store.clear();
await store.save();    // Manual save (if autoSave is false)
await store.reload();  // Reload from disk
await store.reset();   // Reset to defaults

// Lazy loading (defers initialization until first use)
const lazyStore = new LazyStore('cache.json');
```

### 9.4 Permission Usage Pattern

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

### 9.5 Typical Permission Suffixes

For each plugin, auto-generated permissions typically follow this pattern:
- `<plugin>:default` -- Safe default permissions
- `<plugin>:allow-<command>` -- Allow a specific command
- `<plugin>:deny-<command>` -- Deny a specific command

### 9.6 Custom Command -- Full Flow

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

---

## 10. Common Mistakes & Anti-Patterns

### 10.1 Rust Backend

1. **Blocking the main thread**: Using sync commands for I/O operations freezes the UI. Always use `async` for I/O.

2. **`Result<T, String>` everywhere**: Loses error type information. Use `thiserror` with custom error enums.

3. **Missing `Serialize` on error types**: The error type in `Result<T, E>` must implement `Serialize` to be sent to the frontend.

4. **Wrapping state in `Arc`**: Tauri already wraps managed state in `Arc`. Adding another `Arc` is redundant: `app.manage(Arc::new(data))` -- just use `app.manage(data)`.

5. **Nested Mutex locks**: Locking the same `Mutex` twice in a call chain causes deadlocks. Restructure to lock once, or use `RwLock` for read-heavy access.

6. **Multiple `invoke_handler()` calls**: Only the last call takes effect. Put all commands in a single `generate_handler![]`.

7. **Using `&str` in async commands**: Borrowed references don't work with async command spawning. Use `String` instead.

8. **Forgetting `Clone` on event payloads**: `emit()` requires `Serialize + Clone` on payloads.

9. **Pub functions in lib.rs**: Command functions defined directly in `src-tauri/src/lib.rs` must **not** be `pub`, as the glue code generation prevents it. Move commands to a separate module if they need to be public.

### 10.2 Frontend / TypeScript

10. **Forgetting to await unlisten functions**:
```typescript
// WRONG: listen returns a Promise<UnlistenFn>, not UnlistenFn
const unlisten = listen('event', handler);
unlisten(); // TypeError: unlisten is not a function

// CORRECT
const unlisten = await listen('event', handler);
unlisten();
```

11. **Not cleaning up event listeners**:
```typescript
// WRONG: Memory leak in React component
useEffect(() => {
  listen('update', handler); // Never cleaned up!
}, []);

// CORRECT
useEffect(() => {
  const promise = listen('update', handler);
  return () => { promise.then(fn => fn()); };
}, []);
```

12. **Using snake_case argument keys in invoke**:
```typescript
// WRONG: Rust expects camelCase->snake_case auto-conversion
await invoke('save_file', { file_path: '/doc.txt' }); // won't match

// CORRECT
await invoke('save_file', { filePath: '/doc.txt' });
```

13. **Using Tauri APIs in a regular browser**:
```typescript
// WRONG: Crashes in browser
const data = await invoke('get_data');

// CORRECT: Guard with isTauri()
import { isTauri } from '@tauri-apps/api/core';
if (isTauri()) {
  const data = await invoke('get_data');
}
```

14. **Not configuring permissions for plugins**: All plugin APIs will throw runtime errors if the corresponding permissions are not configured in `src-tauri/capabilities/default.json`. This is a common source of "command not allowed" errors.

15. **Assuming synchronous path operations**:
```typescript
// WRONG: Path functions are async in Tauri (unlike Node.js)
const dir = appDataDir(); // Returns Promise, not string!

// CORRECT
const dir = await appDataDir();
```

16. **Using relative paths without BaseDirectory**:
```typescript
// WRONG: No base directory, path will fail security checks
await readTextFile('config.json');

// CORRECT
await readTextFile('config.json', { baseDir: BaseDirectory.AppConfig });
```

17. **Not handling invoke errors**:
```typescript
// WRONG: Unhandled promise rejection if Rust returns Err
const data = await invoke('risky_operation');

// CORRECT
try {
  const data = await invoke('risky_operation');
} catch (error) {
  console.error('Operation failed:', error);
}
```

### 10.3 Security

18. **Setting CSP to `null` in production** -- Disables all content security restrictions. Always define a restrictive CSP.

19. **Using `"windows": ["*"]` with broad permissions** -- Grants all windows the same permissions. Use specific window labels.

20. **Using `dangerousDisableAssetCspModification: true`** -- Removes automatic CSP nonce injection. Only use if you fully understand the implications.

21. **Not enabling `freezePrototype`** -- Leaves your app vulnerable to prototype pollution attacks.

22. **Granting `shell:default` without scope restrictions** -- Allows arbitrary command execution. Always define scopes.

### 10.4 Configuration

23. **Not setting `identifier` to a unique reverse-domain** -- Causes conflicts with other apps on the system.

24. **Forgetting `beforeBuildCommand`** -- Frontend assets won't be built before bundling.

25. **Using `build.devUrl` without `beforeDevCommand`** -- The dev server won't start automatically.

26. **Not committing `Cargo.lock`** -- Leads to non-deterministic builds across environments.

### 10.5 Permissions

27. **Not adding capabilities for plugins** -- Installing a plugin does not automatically grant permissions. You must add permissions to a capability file.

28. **Mixing up TOML vs JSON** -- Application permissions must be TOML. Capabilities can be JSON or TOML.

29. **Forgetting `core:` prefixed permissions** -- Core features (window management, events, paths) also require explicit permissions in v2.

### 10.6 Build & Distribution

30. **Not signing macOS builds** -- Unsigned apps are quarantined by Gatekeeper. At minimum use ad-hoc signing (`"-"`).

31. **Losing the updater private key** -- Makes it impossible to push updates to existing installations. Back it up securely.

32. **Setting `createUpdaterArtifacts: true` without signing keys** -- Build will fail. Set `TAURI_SIGNING_PRIVATE_KEY` environment variable.

33. **Using `"v1Compatible"` for new projects** -- Only needed when migrating from v1 updater. New projects should use `true`.

### 10.7 Migration (v1 to v2)

34. **Not running `npx @tauri-apps/cli migrate`** -- Manual migration is error-prone; the CLI handles most conversions.

35. **Forgetting the updater UI** -- v2 removed the automatic update dialog. You must implement update checking explicitly or users will never receive updates.

36. **Using old import paths** -- `@tauri-apps/api/tauri` is now `@tauri-apps/api/core`; `@tauri-apps/api/window` is now `@tauri-apps/api/webviewWindow`.

37. **Not restructuring for mobile** -- If targeting mobile, you must split `main.rs` into `lib.rs` + `main.rs` and add the `#[cfg_attr(mobile, tauri::mobile_entry_point)]` macro.

---

## 11. API Surface Summary

### Rust API Quick Reference

| API Area | Key Types / Functions | Import |
|----------|----------------------|--------|
| Commands | `#[tauri::command]`, `generate_handler![]` | `tauri` |
| State | `State<'_, T>`, `manage()`, `try_state()` | `tauri::State` |
| Events (emit) | `emit()`, `emit_to()`, `emit_filter()`, `emit_str()` | `tauri::Emitter` |
| Events (listen) | `listen()`, `once()`, `unlisten()`, `listen_any()` | `tauri::Listener` |
| App lifecycle | `Builder`, `AppHandle`, `setup()` | `tauri::Builder` |
| Manager trait | `state()`, `get_webview_window()`, `path()`, `config()` | `tauri::Manager` |
| Windows | `WebviewWindowBuilder`, `WebviewUrl` | `tauri::webview` |
| Menus | `MenuBuilder`, `SubmenuBuilder`, `PredefinedMenuItem` | `tauri::menu` |
| Tray | `TrayIconBuilder` | `tauri::tray` |
| IPC Channels | `Channel<T>`, `Response` | `tauri::ipc` |
| Plugins | `plugin::Builder`, `TauriPlugin` | `tauri::plugin` |
| Errors | `thiserror::Error` + manual `Serialize` impl | `thiserror`, `serde` |

### Frontend API Quick Reference

| API Area | Key Functions / Types | Import Path |
|----------|----------------------|-------------|
| Commands | `invoke<T>()`, `Channel<T>` | `@tauri-apps/api/core` |
| Events | `listen()`, `emit()`, `emitTo()`, `once()` | `@tauri-apps/api/event` |
| Windows | `getCurrentWindow()`, `Window`, `getAllWindows()` | `@tauri-apps/api/window` |
| Webviews | `getCurrentWebview()`, `Webview` | `@tauri-apps/api/webview` |
| Paths | `appDataDir()`, `join()`, `resolve()`, `BaseDirectory` | `@tauri-apps/api/path` |
| Menus | `Menu.new()`, `MenuItem.new()`, `Submenu.new()` | `@tauri-apps/api/menu` |
| Tray | `TrayIcon.new()` | `@tauri-apps/api/tray` |
| Utilities | `convertFileSrc()`, `isTauri()` | `@tauri-apps/api/core` |
| Testing | `mockIPC()`, `mockWindows()`, `clearMocks()` | `@tauri-apps/api/mocks` |

### Plugin Quick Reference

| Plugin | Cargo Crate | npm Package | Default Permission |
|--------|-------------|-------------|-------------------|
| File System | `tauri-plugin-fs` | `@tauri-apps/plugin-fs` | `fs:default` |
| Dialog | `tauri-plugin-dialog` | `@tauri-apps/plugin-dialog` | `dialog:default` |
| HTTP | `tauri-plugin-http` | `@tauri-apps/plugin-http` | `http:default` |
| Shell | `tauri-plugin-shell` | `@tauri-apps/plugin-shell` | `shell:default` |
| Notification | `tauri-plugin-notification` | `@tauri-apps/plugin-notification` | `notification:default` |
| Clipboard | `tauri-plugin-clipboard-manager` | `@tauri-apps/plugin-clipboard-manager` | (none, explicit) |
| OS | `tauri-plugin-os` | `@tauri-apps/plugin-os` | `os:default` |
| Process | `tauri-plugin-process` | `@tauri-apps/plugin-process` | `process:default` |
| Updater | `tauri-plugin-updater` | `@tauri-apps/plugin-updater` | `updater:default` |
| Store | `tauri-plugin-store` | `@tauri-apps/plugin-store` | `store:default` |
| Global Shortcut | `tauri-plugin-global-shortcut` | `@tauri-apps/plugin-global-shortcut` | `global-shortcut:default` |
| Opener | `tauri-plugin-opener` | `@tauri-apps/plugin-opener` | `opener:default` |

### Configuration Quick Reference

| File | Location | Format | Purpose |
|------|----------|--------|---------|
| `tauri.conf.json` | `src-tauri/` | JSON | Main app configuration |
| `default.json` | `src-tauri/capabilities/` | JSON/TOML | Default capability (permissions for windows) |
| `*.toml` | `src-tauri/permissions/` | TOML | Custom command permissions |
| `Cargo.toml` | `src-tauri/` | TOML | Rust dependencies |
| `build.rs` | `src-tauri/` | Rust | Build script (plugin permission generation) |

---

## Version Notes

- **Tauri 2.0**: Initial stable release with the command system, state management, event system, and plugin architecture.
- **Tauri 2.10.2**: Current latest stable (as of March 2026). Added `channel_interceptor`, `append_invoke_initialization_script`, and `invoke_system` on Builder. `emit_str` / `emit_str_to` / `emit_str_filter` methods added to Emitter for pre-serialized payloads.
- **Runtime**: The `R: Runtime` generic defaults to `Wry` when the `wry` feature is enabled (default). Custom runtimes are possible but rarely needed outside testing.
- **Mobile support**: `#[cfg_attr(mobile, tauri::mobile_entry_point)]` on the `run()` function. Plugin mobile support via `--android`/`--ios` CLI flags during plugin creation.
