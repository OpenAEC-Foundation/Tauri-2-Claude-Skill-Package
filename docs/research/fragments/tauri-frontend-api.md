# Tauri 2 Frontend / JavaScript / TypeScript API Reference

> Research document for the Claude Skill Package project.
> Sources: official Tauri v2 documentation, npm registry, GitHub source.

---

## Table of Contents

1. [Core invoke() API](#1-core-invoke-api)
2. [Event System](#2-event-system)
3. [Window API](#3-window-api)
4. [Webview API](#4-webview-api)
5. [Path API](#5-path-api)
6. [Menu API](#6-menu-api)
7. [Tray Icon API](#7-tray-icon-api)
8. [Official Plugin Frontend APIs](#8-official-plugin-frontend-apis)
9. [Utilities](#9-utilities)
10. [Testing and Mocking](#10-testing-and-mocking)
11. [Common Mistakes and Anti-Patterns](#11-common-mistakes-and-anti-patterns)

---

## 1. Core invoke() API

**Import:** `import { invoke } from '@tauri-apps/api/core'`

The `invoke()` function is the primary mechanism for calling Rust backend commands from the frontend. It sends an IPC message to a registered `#[tauri::command]` handler and returns a typed Promise.

### Function Signature

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

### Basic Usage

```typescript
import { invoke } from '@tauri-apps/api/core';

// Simple call, no arguments
await invoke('my_custom_command');

// With arguments — keys are camelCase, matching Rust snake_case params
const result = await invoke<string>('greet', { name: 'World' });
console.log(result); // "Hello, World!"
```

**Critical rule:** Argument keys must be **camelCase** in JavaScript. Tauri automatically converts them to **snake_case** for the Rust side. If the Rust command uses `#[tauri::command(rename_all = "snake_case")]`, you pass snake_case keys from JS instead.

### Argument Passing

Arguments are passed as a plain object whose keys correspond to the Rust function parameters:

```rust
// Rust side
#[tauri::command]
fn greet(name: String, age: u32) -> String {
    format!("Hello {}, you are {} years old", name, age)
}
```

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

### Return Values and TypeScript Typing

The generic parameter `<T>` types the resolved Promise value. The Rust return type must implement `serde::Serialize`:

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

const user = await invoke<User>('get_user', { userId: 42 });
// user is typed as User
```

For large binary responses, Rust can return `tauri::ipc::Response` to avoid JSON serialization overhead:

```rust
use tauri::ipc::Response;

#[tauri::command]
fn read_file() -> Response {
    let data = std::fs::read("/path/to/file").unwrap();
    Response::new(data)
}
```

### Error Handling

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

For structured errors, define a tagged enum in Rust:

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

### Channels (Streaming Data)

The `Channel` class enables streaming data from Rust to the frontend, recommended for large data transfers like file downloads or real-time logs.

**Import:** `import { Channel } from '@tauri-apps/api/core'`

```typescript
import { invoke, Channel } from '@tauri-apps/api/core';

const onChunk = new Channel<Uint8Array>();
onChunk.onmessage = (chunk) => {
  console.log('Received chunk:', chunk.length, 'bytes');
};

await invoke('load_image', { path: '/photo.jpg', reader: onChunk });
```

Rust side:

```rust
#[tauri::command]
async fn load_image(path: std::path::PathBuf, reader: tauri::ipc::Channel<&[u8]>) {
    let mut file = tokio::fs::File::open(path).await.unwrap();
    let mut chunk = vec![0; 4096];
    loop {
        let len = file.read(&mut chunk).await.unwrap();
        if len == 0 { break; }
        reader.send(&chunk[..len]).unwrap();
    }
}
```

### Other Core Exports

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

---

## 2. Event System

**Import:** `import { listen, emit, emitTo, once } from '@tauri-apps/api/event'`

The event system provides a loosely-coupled, non-type-safe messaging layer using JSON payloads. It complements `invoke()` for scenarios where you need broadcast communication or Rust-initiated messages.

### Core Functions

#### listen

```typescript
listen<T>(event: EventName, handler: EventCallback<T>, options?: Options): Promise<UnlistenFn>
```

Subscribes to an event. Returns an unlisten function that must be called for cleanup.

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

Subscribes to an event and automatically unsubscribes after the first emission:

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

Sends an event to a specific target (window/webview label or EventTarget object):

```typescript
import { emitTo } from '@tauri-apps/api/event';

await emitTo('settings', 'settings-update-requested', {
  key: 'notification',
  value: 'all',
});
```

### Types

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

### TauriEvent Enum (Built-in Events)

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

### Event Cleanup Pattern (Important)

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

---

## 3. Window API

**Import:** `import { Window, getCurrentWindow, getAllWindows } from '@tauri-apps/api/window'`

### Getting Window References

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

### Window Manipulation Methods

All methods return `Promise<void>` unless noted:

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

### Window Events

```typescript
const win = getCurrentWindow();

// Close requested (can be prevented)
await win.onCloseRequested(async (event) => {
  const confirmed = await confirm('Are you sure?');
  if (!confirmed) {
    event.preventDefault(); // Prevents window from closing
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

### Creating New Windows

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

### Key Enums

```typescript
// Cursor icons
type CursorIcon = 'default' | 'crosshair' | 'hand' | 'move' | 'text' | 'wait' | ...;

// Theme
type Theme = 'light' | 'dark';

// Title bar style
type TitleBarStyle = 'visible' | 'transparent' | 'overlay';

// User attention
enum UserAttentionType { Critical = 1, Informational = 2 }

// Progress bar status
enum ProgressBarStatus { None, Normal, Indeterminate, Paused, Error }
```

### Monitor Information

```typescript
import { availableMonitors, currentMonitor, primaryMonitor, cursorPosition } from '@tauri-apps/api/window';

const monitors = await availableMonitors();
const current = await currentMonitor();
const primary = await primaryMonitor();
const cursor = await cursorPosition();
```

---

## 4. Webview API

**Import:** `import { Webview, getCurrentWebview, getAllWebviews } from '@tauri-apps/api/webview'`

The Webview API provides finer control than the Window API, allowing multiple webviews inside a single window.

### Key Methods

```typescript
import { getCurrentWebview } from '@tauri-apps/api/webview';

const webview = getCurrentWebview();

// Content
await webview.setZoom(1.5);
await webview.clearAllBrowsingData();

// Sizing
const size = await webview.size();       // PhysicalSize
const pos = await webview.position();    // PhysicalPosition
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

### Creating a Webview Inside a Window

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

---

## 5. Path API

**Import:** `import { appDataDir, join, resolve, ... } from '@tauri-apps/api/path'`

All path functions are async and return `Promise<string>`.

### Directory Functions

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

### Path Manipulation

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

### Resource Resolution

```typescript
import { resolveResource } from '@tauri-apps/api/path';

// Resolves a path relative to the app's resource directory
const modelPath = await resolveResource('models/default.bin');
```

### BaseDirectory Enum

```typescript
import { BaseDirectory } from '@tauri-apps/api/path';

// Used by plugin-fs and other plugins to specify path bases
// Key members:
BaseDirectory.AppData     // 14
BaseDirectory.AppConfig   // 13
BaseDirectory.AppLocalData // 15
BaseDirectory.AppCache    // 16
BaseDirectory.AppLog      // 17
BaseDirectory.Home        // 21
BaseDirectory.Desktop     // 18
BaseDirectory.Document    // 6
BaseDirectory.Download    // 7
BaseDirectory.Resource    // 11
BaseDirectory.Temp        // 12
```

---

## 6. Menu API

**Import:** `import { Menu, MenuItem, Submenu, CheckMenuItem, PredefinedMenuItem } from '@tauri-apps/api/menu'`

### Creating Menus

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

### Context Menus

```typescript
// Show a context menu at the cursor position
await menu.popup();

// Show at specific coordinates
await menu.popup({ x: 100, y: 200 });
```

### Menu Item Management

```typescript
// Dynamically modify menus
await menu.append(await MenuItem.new({ text: 'New Item', action: () => {} }));
await menu.prepend(await MenuItem.new({ text: 'First Item', action: () => {} }));
await menu.insert(1, await MenuItem.new({ text: 'At Index 1', action: () => {} }));
await menu.remove('item-id');
await menu.removeAt(0);

const item = await menu.get('item-id');
const allItems = await menu.items();
```

### PredefinedMenuItem Types

Platform-standard items: `About`, `Hide`, `HideOthers`, `ShowAll`, `CloseWindow`, `Quit`, `Copy`, `Cut`, `Paste`, `SelectAll`, `Undo`, `Redo`, `Minimize`, `Zoom`, `Separator`, `Fullscreen`, `Services`, `BringAllToFront`.

---

## 7. Tray Icon API

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

---

## 8. Official Plugin Frontend APIs

All plugins require both npm installation and Cargo dependency. Permissions must be configured in `src-tauri/capabilities/default.json`.

### 8.1 File System (`@tauri-apps/plugin-fs`)

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

### 8.2 Dialog (`@tauri-apps/plugin-dialog`)

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

### 8.3 HTTP Client (`@tauri-apps/plugin-http`)

**Requires permissions:** URL scoping is required. Configure allowed URLs in capabilities.

```typescript
import { fetch } from '@tauri-apps/plugin-http';

const response = await fetch('https://api.example.com/data', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ key: 'value' }),
});

console.log(response.status);     // 200
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

### 8.4 Notification (`@tauri-apps/plugin-notification`)

**Requires permissions:** `notification:default` or `notification:allow-notify`, `notification:allow-request-permission`, `notification:allow-is-permission-granted`.

```typescript
import {
  isPermissionGranted, requestPermission, sendNotification,
  createChannel, Importance, Visibility,
} from '@tauri-apps/plugin-notification';

// Check and request permission
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

### 8.5 Shell (`@tauri-apps/plugin-shell`)

**Requires permissions:** `shell:allow-execute`, `shell:allow-spawn`, `shell:allow-kill`, `shell:allow-stdin-write`. Commands must be pre-defined in the permission scope with allowed arguments.

```typescript
import { Command } from '@tauri-apps/plugin-shell';

const result = await Command.create('exec-sh', ['-c', 'echo "Hello"']).execute();
console.log(result.stdout); // Uint8Array
console.log(result.status.code); // 0

// Check success
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

### 8.6 Clipboard (`@tauri-apps/plugin-clipboard-manager`)

**Requires permissions:** No default permissions. Must explicitly grant `clipboard-manager:allow-read-text`, `clipboard-manager:allow-write-text`, `clipboard-manager:allow-read-image`, `clipboard-manager:allow-write-image`, `clipboard-manager:allow-write-html`, `clipboard-manager:allow-clear`.

```typescript
import { writeText, readText } from '@tauri-apps/plugin-clipboard-manager';

await writeText('Hello, clipboard!');
const content = await readText();
console.log(content); // "Hello, clipboard!"
```

### 8.7 OS Information (`@tauri-apps/plugin-os`)

**Requires permissions:** `os:default`.

```typescript
import { platform, arch, type, version, family, hostname, locale, eol } from '@tauri-apps/plugin-os';

const os = await platform();  // "windows", "linux", "macos", "android", "ios"
const cpu = await arch();     // "x86_64", "aarch64", etc.
const osType = await type();  // "windows_nt", "linux", "darwin"
const ver = await version();  // e.g., "10.0.22621"
const fam = await family();   // "unix" or "windows"
const host = await hostname(); // machine hostname
const loc = await locale();   // e.g., "en-US"
const lineEnd = eol();        // "\r\n" on Windows, "\n" on Unix
```

### 8.8 Process (`@tauri-apps/plugin-process`)

**Requires permissions:** `process:default` (includes `process:allow-exit` and `process:allow-restart`).

```typescript
import { exit, relaunch } from '@tauri-apps/plugin-process';

await exit(0);     // Terminate with status code
await relaunch();  // Restart the application
```

### 8.9 Updater (`@tauri-apps/plugin-updater`)

**Requires permissions:** `updater:default`.

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

### 8.10 Store (`@tauri-apps/plugin-store`)

**Requires permissions:** `store:default`.

```typescript
import { load, LazyStore } from '@tauri-apps/plugin-store';

// Eager loading
const store = await load('settings.json', { autoSave: true });

await store.set('theme', 'dark');
await store.set('user', { name: 'Alice', prefs: { lang: 'en' } });

const theme = await store.get<string>('theme');         // "dark"
const hasKey = await store.has('theme');                 // true
const allKeys = await store.keys();                     // ["theme", "user"]
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
// Operations are identical but initialization happens on first call
```

---

## 9. Utilities

### convertFileSrc

**Import:** `import { convertFileSrc } from '@tauri-apps/api/core'`

Converts a native file path to a URL that can be used in `<img>`, `<video>`, `<audio>`, or CSS:

```typescript
import { convertFileSrc } from '@tauri-apps/api/core';

const assetUrl = convertFileSrc('/path/to/image.png');
// Returns: "https://asset.localhost/path/to/image.png" (on most platforms)
// Or: "asset://localhost/path/to/image.png"
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

### isTauri

```typescript
import { isTauri } from '@tauri-apps/api/core';

if (isTauri()) {
  // Running inside Tauri webview — safe to use Tauri APIs
  const data = await invoke('get_data');
} else {
  // Running in a regular browser — use web APIs
  const data = await fetch('/api/data').then(r => r.json());
}
```

### Asset Protocol

Tauri serves local files through the `asset://` protocol. This is configured in `tauri.conf.json`:

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

### Global Tauri Object

When `app.withGlobalTauri` is set to `true` in `tauri.conf.json`, all APIs are available on `window.__TAURI__`:

```typescript
const { invoke } = window.__TAURI__.core;
const { listen, emit } = window.__TAURI__.event;
const { getCurrentWindow } = window.__TAURI__.window;
```

---

## 10. Testing and Mocking

**Import:** `import { mockIPC, mockWindows, clearMocks, mockConvertFileSrc } from '@tauri-apps/api/mocks'`

### mockIPC

Intercepts all `invoke()` calls with a handler function:

```typescript
import { mockIPC, clearMocks } from '@tauri-apps/api/mocks';
import { invoke } from '@tauri-apps/api/core';

// Setup
mockIPC((cmd, args) => {
  if (cmd === 'greet') {
    return `Hello, ${args.name}!`;
  }
  if (cmd === 'get_count') {
    return 42;
  }
  throw new Error(`Unknown command: ${cmd}`);
});

// Test
const result = await invoke<string>('greet', { name: 'Test' });
expect(result).toBe('Hello, Test!');

// Cleanup
clearMocks();
```

With event mocking:

```typescript
mockIPC(
  (cmd, args) => { /* handle commands */ },
  { shouldMockEvents: true }
);
```

### mockWindows

Creates mock window labels for testing window-dependent code:

```typescript
import { mockWindows, clearMocks } from '@tauri-apps/api/mocks';
import { getCurrentWindow } from '@tauri-apps/api/window';

mockWindows('main', 'settings', 'about');

const win = getCurrentWindow();
expect(win.label).toBe('main');

clearMocks();
```

### mockConvertFileSrc

Mocks the `convertFileSrc` function for a specific OS:

```typescript
import { mockConvertFileSrc, clearMocks } from '@tauri-apps/api/mocks';

mockConvertFileSrc('linux');
// convertFileSrc will now behave as if running on Linux

clearMocks();
```

### clearMocks

Resets all mock state. Always call in `afterEach` or equivalent teardown:

```typescript
afterEach(() => {
  clearMocks();
});
```

---

## 11. Common Mistakes and Anti-Patterns

### 1. Forgetting to await unlisten functions

```typescript
// WRONG: listen returns a Promise<UnlistenFn>, not UnlistenFn
const unlisten = listen('event', handler);
unlisten(); // TypeError: unlisten is not a function

// CORRECT
const unlisten = await listen('event', handler);
unlisten();
```

### 2. Not cleaning up event listeners

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

### 3. Using snake_case argument keys in invoke

```typescript
// WRONG: Rust expects camelCase→snake_case auto-conversion
await invoke('save_file', { file_path: '/doc.txt' }); // won't match

// CORRECT
await invoke('save_file', { filePath: '/doc.txt' });

// ALSO CORRECT (if Rust uses rename_all = "snake_case")
await invoke('save_file', { file_path: '/doc.txt' });
```

### 4. Using Tauri APIs in a regular browser

```typescript
// WRONG: Crashes in browser
const data = await invoke('get_data');

// CORRECT: Guard with isTauri()
import { isTauri } from '@tauri-apps/api/core';
if (isTauri()) {
  const data = await invoke('get_data');
}
```

### 5. Not configuring permissions for plugins

All plugin APIs will throw runtime errors if the corresponding permissions are not configured in `src-tauri/capabilities/default.json`. This is a common source of "command not allowed" errors.

### 6. Assuming synchronous path operations

```typescript
// WRONG: Path functions are async in Tauri (unlike Node.js)
const dir = appDataDir(); // Returns Promise, not string!

// CORRECT
const dir = await appDataDir();
```

### 7. Using relative paths without BaseDirectory

```typescript
// WRONG: No base directory, path will fail security checks
await readTextFile('config.json');

// CORRECT
await readTextFile('config.json', { baseDir: BaseDirectory.AppConfig });
```

### 8. Not handling invoke errors

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

### 9. Blocking the main thread with synchronous commands

Rust commands that perform I/O or heavy computation should be marked `async`. Non-async commands block the main thread, causing UI freezes.

### 10. Pub functions in lib.rs

Command functions defined directly in `src-tauri/src/lib.rs` must **not** be `pub`, as the glue code generation prevents it. Move commands to a separate module (`src/commands.rs`) if they need to be public.

---

## Appendix: Package Information

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
