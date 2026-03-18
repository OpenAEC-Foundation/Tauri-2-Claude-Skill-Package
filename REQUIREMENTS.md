# REQUIREMENTS

## What This Skill Package Must Achieve

### Primary Goal
Enable Claude to write correct, version-aware Tauri 2 code for both Rust backend and TypeScript/JavaScript frontend — without hallucinating APIs.

### What Claude Should Do After Loading Skills
1. Recognize Tauri 2 context from user requests
2. Select the correct skill(s) automatically based on the request
3. Write correct Rust AND TypeScript code for the specified operation
4. Avoid known anti-patterns and common AI mistakes
5. Follow best practices documented in the skill references

### Quality Guarantees
| Guarantee | Description |
|-----------|-------------|
| Version-correct | Code MUST specify Tauri 2.x target |
| API-accurate | All method signatures verified against official docs |
| Dual-language | IPC-related skills show both Rust and TypeScript |
| Anti-pattern-free | Known mistakes are explicitly documented and avoided |
| Deterministic | Skills use ALWAYS/NEVER language, not suggestions |
| Self-contained | Each skill works independently without requiring other skills |

---

## Per-Area Requirements

### 1. Commands
| Requirement | Detail |
|-------------|--------|
| Core pattern | `#[tauri::command]` macro usage, sync and async variants |
| Parameters | `State<T>`, `Window`, `AppHandle` injection patterns |
| Return types | Correct Result<T, E> serialization, error handling |
| IPC bridge | Both Rust handler and TypeScript `invoke()` call |
| Critical | Async command patterns with proper runtime handling |
| Critical | Error serialization across the IPC boundary |

### 2. Events
| Requirement | Detail |
|-------------|--------|
| Patterns | `emit`, `listen`, `once` for both global and window-scoped |
| Payload | Serde serialization/deserialization of event payloads |
| Lifecycle | Unlisten cleanup, memory leak prevention |
| Critical | Global vs window-scoped event routing |

### 3. Plugins
| Requirement | Detail |
|-------------|--------|
| Official plugins | tauri-plugin-fs, tauri-plugin-shell, tauri-plugin-dialog, etc. |
| Architecture | Plugin trait, Builder pattern, permissions, capabilities |
| ACL | Capability files, permission definitions, scope configuration |
| Custom | Full custom plugin development lifecycle |
| Critical | Permission system and capability file JSON format |

### 4. Window Management
| Requirement | Detail |
|-------------|--------|
| Creation | `WebviewWindowBuilder` API, multi-window patterns |
| Configuration | Window properties, decorations, transparency |
| Webview | Webview configuration, custom protocols |
| Critical | Multi-window state sharing and communication |

### 5. State Management
| Requirement | Detail |
|-------------|--------|
| Core | `app.manage()` registration, `State<T>` extraction |
| Concurrency | `Mutex<T>`, `RwLock<T>` for shared mutable state |
| Patterns | State initialization, lazy state, async state access |
| Critical | Deadlock prevention in concurrent state access |

### 6. Mobile
| Requirement | Detail |
|-------------|--------|
| Targets | Android and iOS build targets |
| Platform code | `#[cfg(mobile)]`, platform-specific APIs |
| Permissions | Mobile permission requests (camera, filesystem, etc.) |
| Critical | Platform-specific plugin capabilities |

### 7. Build & Distribution
| Requirement | Detail |
|-------------|--------|
| Configuration | `tauri.conf.json` structure and options |
| Bundler | Platform-specific bundler settings (NSIS, DMG, AppImage, etc.) |
| Signing | Code signing for Windows, macOS, Linux |
| Auto-updater | Update server configuration, update flow |
| CI/CD | GitHub Actions, cross-platform build pipelines |
| Critical | Correct `tauri.conf.json` for each target platform |

### 8. Security
| Requirement | Detail |
|-------------|--------|
| CSP | Content Security Policy configuration |
| Permissions | Permission system, default-deny model |
| Capabilities | Capability file format, scope definitions |
| IPC safety | Command access control, argument validation |
| Critical | Capability files MUST include correct JSON format with examples |

---

## Critical Requirements (apply to ALL skills)

- All Rust code MUST compile with latest stable Rust + tauri 2.x crate
- All TypeScript MUST work with `@tauri-apps/api` v2
- Permission examples MUST include capability file JSON
- IPC skills MUST show both Rust and TypeScript sides (D-008)
- Code examples MUST be verified against official documentation

---

## Structural Requirements

### Skill Format
- SKILL.md < 500 lines (heavy content in references/)
- YAML frontmatter with name and description (including trigger words)
- English-only content
- Deterministic language (ALWAYS/NEVER, imperative)

### Skill Categories
| Category | Purpose | Must Include |
|----------|---------|--------------|
| syntax/ | How to write it | Method signatures, code patterns, version notes |
| impl/ | How to build it | Decision trees, workflows, step-by-step |
| errors/ | How to handle failures | Error patterns, diagnostics, recovery |
| core/ | Cross-cutting | API overview, architecture, concepts |
| agents/ | Orchestration | Validation checklists, auto-detection |

---

## Research Requirements (before creating any skill)

1. Official documentation MUST be consulted and referenced
2. Source code MUST be checked for accuracy when docs are ambiguous
3. Anti-patterns MUST be identified from real issues (GitHub issues, Discord, forums)
4. Code examples MUST be verified (not hallucinated)
5. Version accuracy MUST be confirmed via WebFetch (D-012)

---

## Non-Requirements (explicitly out of scope)

- No Tauri 1.x coverage (except a migration reference — D-007)
- No Electron comparisons or migration guides from Electron
- No specific frontend framework deep-dives (React, Vue, Svelte internals)
- No GUI tutorials (skills cover code APIs, not UI walkthrough)
- No mobile-first tutorials (mobile is covered but desktop is primary)
