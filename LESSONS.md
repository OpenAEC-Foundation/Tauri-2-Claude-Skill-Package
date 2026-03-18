# Lessons Learned

Observations and findings captured during skill package development.

---

## L-001: Single Technology Simplifies Package Structure

- **Date**: 2026-03-18
- **Context**: Unlike the Blender package (4 technologies, 73 skills), Tauri 2 is one technology.
- **Finding**: No need for per-technology separation. `skills/source/tauri-{category}/` is sufficient. Simpler dependency graph. The entire package can use a flat category structure without the layered technology namespacing required by multi-technology packages.

---

## L-002: IPC Bridge is the Central Complexity

- **Date**: 2026-03-18
- **Context**: Tauri's core value proposition is the Rust-JS bridge.
- **Finding**: Every IPC-related skill MUST show both Rust and TypeScript sides. Serialization (serde) is the most common pain point. Skills must cover both directions: frontend invoking backend commands, and backend emitting events to frontend. Incomplete examples that show only one side lead to broken implementations.

---

## L-003: Permissions System is Entirely New in v2

- **Date**: 2026-03-18
- **Context**: Tauri 1 had an allowlist, Tauri 2 has a full permissions/capabilities system.
- **Finding**: No equivalent in v1. This needs dedicated deep research. Capability files, plugin permissions, scopes — all new concepts that Claude frequently gets wrong. Skills covering permissions MUST be verified against https://v2.tauri.app/security/ and cross-referenced with plugin source code for accurate permission identifiers.
