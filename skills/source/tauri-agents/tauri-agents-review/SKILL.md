---
name: tauri-agents-review
description: "Provides a comprehensive validation checklist for reviewing Tauri 2 code including command signature correctness, permission coverage verification, state management patterns, error handling completeness, IPC bridge completeness (Rust+TypeScript pairing), security audit, and anti-pattern detection across all Tauri domains. Activates when reviewing Tauri code, auditing permissions, or validating a Tauri project before deployment."
license: MIT
compatibility: "Designed for Claude Code. Requires Tauri 2.x."
metadata:
  author: Impertio
  version: "1.0"
---

# tauri-agents-review

## Review Workflow

Execute these checklists in order. Each section is independent -- report all findings, do not stop at the first issue.

```
Review Order:
1. Commands & IPC Bridge    --> Rust signatures + JS invoke pairing
2. Permissions & Capabilities --> ACL coverage
3. State Management         --> Thread safety + correctness
4. Error Handling           --> Result types + frontend catching
5. Security                 --> CSP + scopes + dangerous settings
6. Build Configuration      --> Config completeness
7. Anti-Pattern Scan        --> Known mistakes
```

---

## Checklist 1: Commands & IPC Bridge

### 1A: Rust Command Signatures

For every `#[tauri::command]` function, verify:

```
[ ] Function is NOT pub if defined directly in lib.rs
[ ] Function IS pub if defined in a separate module
[ ] Async commands use owned types (String, not &str) for parameters
    Exception: &str allowed if return type is Result<T, E>
[ ] Return type is Result<T, E> for any fallible operation
[ ] E implements serde::Serialize
[ ] All custom types in parameters implement serde::Deserialize
[ ] All custom types in return values implement serde::Serialize
[ ] Event payload types implement both Serialize AND Clone
```

### 1B: Command Registration

```
[ ] All commands listed in a SINGLE generate_handler![] call
    FAIL if multiple .invoke_handler() calls found on Builder
[ ] Every #[tauri::command] function is registered
    Cross-reference: grep for #[tauri::command] and compare with generate_handler![]
[ ] Module paths are correct (e.g., commands::greet, not just greet)
```

### 1C: IPC Bridge Completeness

For every Rust command, verify a matching frontend invoke exists:

```
[ ] invoke('command_name', ...) call exists in TypeScript/JavaScript
[ ] Argument keys use camelCase (NOT snake_case)
[ ] Argument types match Rust parameter types:
    - Rust String <-> JS string
    - Rust i32/u32/u64/f64 <-> JS number
    - Rust bool <-> JS boolean
    - Rust Vec<T> <-> JS array
    - Rust Option<T> <-> JS null/undefined
[ ] Return type generic matches: invoke<ExpectedType>('...')
[ ] Error case handled with try/catch
```

### 1D: Channel Usage (if applicable)

```
[ ] Channel<T> parameter in Rust has matching new Channel<T>() in JS
[ ] Channel.onmessage is set before invoke() call
[ ] Channel type parameter matches Rust send() type
```

---

## Checklist 2: Permissions & Capabilities

### 2A: Plugin Permissions

For every plugin in Cargo.toml dependencies:

```
[ ] Plugin initialized in Builder: .plugin(tauri_plugin_X::init())
[ ] Corresponding npm package installed: @tauri-apps/plugin-X
[ ] Permission entry exists in capabilities file(s)
[ ] Default permission or specific allow-* permissions granted
```

### 2B: Custom Command Permissions

For every `#[tauri::command]`:

```
[ ] Permission defined in src-tauri/permissions/*.toml
    Format: commands.allow = ["command_name"]
[ ] Permission referenced in capabilities file
[ ] No custom command is callable without explicit permission
```

### 2C: Capability File Validation

For every file in `src-tauri/capabilities/`:

```
[ ] Has valid $schema reference
[ ] Has unique identifier
[ ] windows[] array targets specific labels (NOT ["*"] without justification)
[ ] permissions[] array contains all needed permissions
[ ] Platform-specific capabilities use correct platforms[] values
[ ] No duplicate permission entries across capability files
```

### 2D: Scope Verification

```
[ ] fs plugin scopes restrict to needed directories only
[ ] http plugin scopes restrict to needed URLs only
[ ] shell plugin scopes whitelist specific commands only
[ ] Asset protocol scope is minimal ($APPDATA/**, $RESOURCE/**)
[ ] Deny rules used to exclude sensitive paths/URLs
```

---

## Checklist 3: State Management

### 3A: Registration

```
[ ] All State<'_, T> types used in commands have matching manage() calls
[ ] manage() called BEFORE any command that uses the state can execute
    (i.e., in Builder chain or early in setup())
[ ] No duplicate manage() calls for the same type (returns false)
[ ] State types are Send + Sync + 'static
```

### 3B: Thread Safety

```
[ ] Mutable state wrapped in Mutex, RwLock, or atomic types
[ ] std::sync::Mutex used by default (NOT tokio::sync::Mutex)
[ ] tokio::sync::Mutex used ONLY when lock held across .await
[ ] No Arc wrapping (Tauri manages Arc internally)
[ ] No nested locking of the same Mutex (deadlock risk)
```

### 3C: Lock Discipline

```
[ ] Mutex guards are dropped before calling other functions that may lock
[ ] Lock scopes are minimal (lock, operate, drop)
[ ] RwLock used for read-heavy state (multiple concurrent readers)
[ ] No lock held during I/O or network operations (unless tokio Mutex)
```

---

## Checklist 4: Error Handling

### 4A: Rust Side

```
[ ] All commands returning data use Result<T, E>
[ ] Error type uses thiserror (not manual Display impl)
[ ] Error type implements Serialize (manual impl, not derive)
[ ] No unwrap() or expect() in command handlers
[ ] I/O errors converted via #[from] or manual From impl
[ ] Error messages are user-friendly (no raw debug output)
```

### 4B: Frontend Side

```
[ ] Every invoke() call wrapped in try/catch
[ ] Error catch blocks handle the error (log, display, recover)
[ ] Structured errors parsed correctly (if using tagged enum pattern)
[ ] No unhandled Promise rejections from invoke()
```

---

## Checklist 5: Security Audit

### 5A: Content Security Policy

```
[ ] CSP is NOT null in production
[ ] default-src set to 'self'
[ ] script-src does NOT include 'unsafe-eval'
[ ] connect-src includes ipc: and http://ipc.localhost
[ ] img-src includes asset: and https://asset.localhost if needed
[ ] No wildcard (*) domains without justification
```

### 5B: Dangerous Settings

```
[ ] freezePrototype set to true (prevents prototype pollution)
[ ] dangerousDisableAssetCspModification is false or absent
[ ] withGlobalTauri is false in production
[ ] No shell:default without scope restrictions
[ ] No fs permissions without scope restrictions
[ ] No http permissions without URL scope
```

### 5C: Capability Scope

```
[ ] No capability uses windows: ["*"] with broad permissions
[ ] Platform-specific capabilities correctly scoped
[ ] Remote capabilities (if any) have minimal permissions
[ ] Core permissions are explicit (core:default, core:window:default)
```

---

## Checklist 6: Build Configuration

### 6A: tauri.conf.json

```
[ ] identifier is unique reverse-domain format
[ ] build.frontendDist points to correct output directory
[ ] build.beforeBuildCommand builds frontend assets
[ ] build.beforeDevCommand starts dev server
[ ] build.devUrl matches dev server port
[ ] app.windows[] has at least one window configured
[ ] bundle.icon includes all required formats (.ico, .icns, .png)
```

### 6B: Cargo.toml

```
[ ] tauri dependency version is 2.x
[ ] tauri-build in build-dependencies
[ ] build.rs exists and calls tauri_build::build()
[ ] crate-type includes ["staticlib", "cdylib", "rlib"] if targeting mobile
[ ] All plugin crates versioned at "2"
```

### 6C: Package.json

```
[ ] @tauri-apps/cli in devDependencies
[ ] @tauri-apps/api in dependencies
[ ] All plugin npm packages installed (@tauri-apps/plugin-*)
[ ] Versions aligned (all ^2)
```

### 6D: Source Control

```
[ ] Cargo.lock committed (NOT in .gitignore)
[ ] src-tauri/target/ in .gitignore
[ ] No secrets in committed files (.env, API keys, signing keys)
```

---

## Checklist 7: Anti-Pattern Scan

Scan the codebase for these known issues:

```
[ ] No multiple .invoke_handler() calls
[ ] No pub commands in lib.rs
[ ] No Arc wrapping of managed state
[ ] No &str in async command parameters (without Result return)
[ ] No .unwrap() in command handlers
[ ] No snake_case keys in frontend invoke() calls
[ ] No event listeners without cleanup
[ ] No event names with dots, spaces, or special characters
[ ] No CSP set to null
[ ] No missing Serialize on error types
[ ] No missing Clone on event payloads
[ ] No sync I/O operations in command handlers (use async)
[ ] No missing permissions for installed plugins
[ ] No relative paths without BaseDirectory in fs calls
```

---

## Review Report Template

After completing all checklists, produce a report:

```
## Tauri 2 Code Review Report

### Summary
- Total issues found: X
- Critical (blocks deployment): X
- Warning (should fix): X
- Info (improvement suggestion): X

### Critical Issues
1. [CRIT-001] Description -- Location -- Fix

### Warnings
1. [WARN-001] Description -- Location -- Fix

### Passed Checks
- Commands & IPC Bridge: PASS/FAIL (X/Y checks passed)
- Permissions: PASS/FAIL
- State Management: PASS/FAIL
- Error Handling: PASS/FAIL
- Security: PASS/FAIL
- Build Config: PASS/FAIL
- Anti-Patterns: PASS/FAIL
```

---

## Decision Trees

### Is a command correctly wired?

```
Does #[tauri::command] exist on the function?
+-- No --> Add the macro
+-- Yes
    |
    Is it in generate_handler![]?
    +-- No --> Add it
    +-- Yes
        |
        Does a permission exist in src-tauri/permissions/?
        +-- No --> Create permission TOML
        +-- Yes
            |
            Is it referenced in a capability file?
            +-- No --> Add to capabilities
            +-- Yes
                |
                Does a matching invoke() call exist in JS?
                +-- No --> Add frontend invoke
                +-- Yes --> PASS
```

### Is state correctly managed?

```
Is State<'_, T> used in a command?
+-- No --> Skip
+-- Yes
    |
    Does manage(T) exist on Builder or in setup()?
    +-- No --> CRITICAL: Add manage() call
    +-- Yes
        |
        Does the generic type match exactly?
        +-- No --> CRITICAL: Fix type mismatch
        +-- Yes
            |
            Is T mutable?
            +-- No --> PASS
            +-- Yes
                |
                Is T wrapped in Mutex/RwLock?
                +-- No --> CRITICAL: Add interior mutability
                +-- Yes --> PASS (verify no deadlocks)
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- Review commands, grep patterns, validation rules
- [references/examples.md](references/examples.md) -- Example review findings and reports
- [references/anti-patterns.md](references/anti-patterns.md) -- Complete anti-pattern catalog with detection patterns

### Official Sources

- https://v2.tauri.app/security/
- https://v2.tauri.app/develop/calling-rust/
- https://v2.tauri.app/security/capabilities/
