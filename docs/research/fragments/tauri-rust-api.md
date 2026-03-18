# Tauri 2 Rust Backend API Reference

> Research document generated from official Tauri 2 documentation (v2.10.2).
> Sources: v2.tauri.app/develop/*, docs.rs/tauri/2.10.2/*

---

## 1. Commands System

The command system is the primary mechanism for frontend-to-backend communication in Tauri 2. The `#[tauri::command]` macro transforms regular Rust functions into IPC endpoints callable from JavaScript.

### 1.1 The `#[tauri::command]` Macro

Basic usage:

```rust
#[tauri::command]
fn greet(name: String) -> String {
    format!("Hello, {}!", name)
}
```

**Critical rule**: Commands defined in `lib.rs` cannot be marked `pub` due to glue code generation constraints. Commands in separate modules should be `pub`.

#### Macro Attributes

- `#[tauri::command(rename_all = "snake_case")]` — changes the argument naming convention for the JS side (default is camelCase)
- `#[tauri::command(async)]` — forces a synchronous function to run on the async runtime instead of the main thread

### 1.2 Sync vs Async Commands

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

### 1.3 Argument Types

Commands accept any type that implements `serde::Deserialize`:

| Type | Example | Notes |
|------|---------|-------|
| Primitives | `i32`, `u64`, `f64`, `bool` | Direct mapping from JS numbers/booleans |
| `String` | `name: String` | JS string → Rust String |
| `Vec<T>` | `items: Vec<String>` | JS array → Rust Vec |
| `Option<T>` | `limit: Option<u32>` | JS null/undefined → None |
| Custom structs | `data: MyStruct` | Must `#[derive(Deserialize)]` |
| `PathBuf` | `path: std::path::PathBuf` | JS string → file path |

Arguments are passed from JavaScript as a JSON object with **camelCase** keys:

```javascript
invoke('my_command', { userName: 'Alice', itemCount: 42 });
```

Maps to Rust:

```rust
#[tauri::command]
fn my_command(user_name: String, item_count: u32) { }
```

### 1.4 Return Types

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

### 1.5 Error Handling

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

**Anti-pattern**: Using `Result<T, String>` everywhere. While quick, it loses type information and makes error handling on the JS side harder.

### 1.6 Special Parameters (Injected Automatically)

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

### 1.7 Command Registration

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

### 1.8 Runtime Generics

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

---

## 2. State Management

### 2.1 Registering State

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

### 2.2 Accessing State in Commands

```rust
#[tauri::command]
fn get_config(state: tauri::State<'_, AppConfig>) -> String {
    state.api_url.clone()
}
```

### 2.3 Accessing State Outside Commands

Via the `Manager` trait (implemented by `App`, `AppHandle`, `Window`, `WebviewWindow`):

```rust
let config = app_handle.state::<AppConfig>();
// or with option:
let maybe_config = app_handle.try_state::<AppConfig>();
```

### 2.4 Mutable State with Mutex

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

### 2.5 Common Pitfalls

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

---

## 3. Event System (Rust Side)

### 3.1 Emitter Trait

The `Emitter` trait is implemented by `App`, `AppHandle`, `Webview`, `WebviewWindow`, and `Window`.

#### Method Signatures

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

#### Usage Examples

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

### 3.2 Listener Trait

The `Listener` trait is implemented by the same types as `Emitter`.

#### Method Signatures

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

#### Usage Example

```rust
use tauri::Listener;

let id = app_handle.listen("user-action", |event| {
    println!("Received event with payload: {:?}", event.payload());
});

// Later: remove the listener
app_handle.unlisten(id);
```

**Event name validation**: Event names may only contain alphanumeric characters, `-`, `/`, `:`, and `_`. Other characters cause a panic.

### 3.3 Event Payloads

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

---

## 4. App Lifecycle & Runtime

### 4.1 Builder — The Entry Point

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
        // return Result<Menu<R>>
        MenuBuilder::new(app).build()
    })
    .run(tauri::generate_context!())
    .expect("error running app");
```

#### Complete Builder Method List (v2.10.2)

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

### 4.2 The `setup()` Hook

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

### 4.3 AppHandle

`AppHandle` is the primary handle for interacting with the application from any context. It is `Clone + Send + Sync`, making it safe to pass to threads:

```rust
let handle = app.handle().clone();
tokio::spawn(async move {
    let state = handle.state::<MyState>();
    handle.emit("event", "data").unwrap();
    let window = handle.get_webview_window("main");
});
```

### 4.4 The Manager Trait

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

### 4.5 Async Runtime

Tauri 2 uses **tokio** under the hood. Async commands are automatically spawned on the tokio runtime — you do NOT need to configure tokio yourself. The `tauri::async_runtime` module provides access if needed.

---

## 5. Plugin System (Rust Side)

### 5.1 Plugin Builder

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

### 5.2 Plugin with Configuration

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

### 5.3 Plugin Lifecycle Hooks

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

### 5.4 Plugin Commands

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

### 5.5 Plugin Permissions

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

---

## 6. Window Management (Rust Side)

### 6.1 Creating Windows

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

#### Key WebviewWindowBuilder Methods

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

### 6.2 Accessing Windows

```rust
// Get a specific window by label
if let Some(window) = app_handle.get_webview_window("main") {
    window.set_title("New Title").unwrap();
}

// Get all windows
let windows = app_handle.webview_windows(); // HashMap<String, WebviewWindow<R>>

// Get the currently focused window
if let Some(focused) = app_handle.get_focused_window() {
    println!("Focused: {}", focused.label());
}
```

### 6.3 Window Events

The `WindowEvent` enum variants:

| Variant | Fields | Description |
|---------|--------|-------------|
| `Resized` | `PhysicalSize<u32>` | Window client area resized |
| `Moved` | `PhysicalPosition<i32>` | Window position changed |
| `CloseRequested` | `api: CloseRequestApi` | Close requested; call `api.prevent_close()` to cancel |
| `Destroyed` | — | Window has been destroyed |
| `Focused` | `bool` | `true` = gained focus, `false` = lost focus |
| `ScaleFactorChanged` | `scale_factor: f64, new_inner_size: PhysicalSize<u32>` | DPI/display scale changed |
| `DragDrop` | `DragDropEvent` | File drag-and-drop event |
| `ThemeChanged` | `Theme` | System theme changed (not supported on Linux) |

#### Handling Window Events

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

### 6.4 Multi-Window Pattern

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

---

## 7. Menu & Tray (Rust Side)

### 7.1 Application Menu

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

### 7.2 Menu Event Handling

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

### 7.3 Menu Item Types

| Type | Purpose |
|------|---------|
| `MenuItem` / `MenuItemBuilder` | Basic text menu item |
| `CheckMenuItem` / `CheckMenuItemBuilder` | Toggleable checkbox item |
| `IconMenuItem` / `IconMenuItemBuilder` | Item with icon |
| `PredefinedMenuItem` | OS-native items: `quit()`, `about()`, `copy()`, etc. |
| `Submenu` / `SubmenuBuilder` | Nested menu container |

### 7.4 System Tray

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
            // Handle clicks on the tray icon itself
            println!("Tray event: {:?}", event);
        })
        .build(app)?;

    Ok(())
})
```

#### TrayIconBuilder Methods

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

---

## 8. IPC Channels (Advanced)

### 8.1 The Channel Type

`Channel<TSend>` enables **streaming** data from Rust to JavaScript. Unlike commands that return a single response, channels allow sending multiple messages over time.

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

### 8.2 Streaming Pattern

```rust
use tauri::ipc::Channel;
use tokio::io::AsyncReadExt;

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
    let mut stream = response.bytes_stream();

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

### 8.3 Binary Data Streaming

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

### 8.4 Channel vs Events

| Feature | Channel | Events |
|---------|---------|--------|
| Direction | Rust → JS (response stream) | Bidirectional |
| Lifetime | Tied to command invocation | App lifetime |
| Targeting | Specific caller | Broadcast or filtered |
| Use case | Progress, streaming data | App-wide notifications |

---

## Common Anti-Patterns

1. **Blocking the main thread**: Using sync commands for I/O operations freezes the UI. Always use `async` for I/O.

2. **`Result<T, String>` everywhere**: Loses error type information. Use `thiserror` with custom error enums.

3. **Missing `Serialize` on error types**: The error type in `Result<T, E>` must implement `Serialize` to be sent to the frontend.

4. **Wrapping state in `Arc`**: Tauri already wraps managed state in `Arc`. Adding another `Arc` is redundant: `app.manage(Arc::new(data))` — just use `app.manage(data)`.

5. **Nested Mutex locks**: Locking the same `Mutex` twice in a call chain causes deadlocks. Restructure to lock once, or use `RwLock` for read-heavy access.

6. **Multiple `invoke_handler()` calls**: Only the last call takes effect. Put all commands in a single `generate_handler![]`.

7. **Using `&str` in async commands**: Borrowed references don't work with async command spawning. Use `String` instead.

8. **Forgetting `Clone` on event payloads**: `emit()` requires `Serialize + Clone` on payloads.

---

## Version Notes

- **Tauri 2.0**: Initial stable release with the command system, state management, event system, and plugin architecture described above.
- **Tauri 2.10.2**: Current latest stable (as of March 2026). Added `channel_interceptor`, `append_invoke_initialization_script`, and `invoke_system` on Builder. `emit_str` / `emit_str_to` / `emit_str_filter` methods added to Emitter for pre-serialized payloads.
- **Runtime**: The `R: Runtime` generic defaults to `Wry` when the `wry` feature is enabled (default). Custom runtimes are possible but rarely needed outside testing.
- **Mobile support**: `#[cfg_attr(mobile, tauri::mobile_entry_point)]` on the `run()` function. Plugin mobile support via `--android`/`--ios` CLI flags during plugin creation.
