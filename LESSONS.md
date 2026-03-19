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

---

## L-004: Batch Execution Velocity with 3-Agent Batches

- **Date**: 2026-03-18
- **Context**: Phase 5 skill creation used Claude Code Agent tool with 3 agents per batch.
- **Finding**: Batch execution with 3 agents per batch proved effective for skill creation velocity — 27 skills across 10 batches completed in a single session.

---

## L-005: 500-Line Limit Requires Aggressive Reference Usage

- **Date**: 2026-03-18
- **Context**: Several skills (design-patterns, project-scaffolder) approached 498 lines in SKILL.md.
- **Finding**: The 498-line SKILL.md files (design-patterns, project-scaffolder) show the 500-line limit is tight for complex topics — aggressive use of references/ directory is essential.

---

## L-006: Deep Vooronderzoek Can Replace Topic-Specific Research

- **Date**: 2026-03-18
- **Context**: Phase 4 topic-specific research was planned but the vooronderzoek was 3713 lines.
- **Finding**: Consolidating Phase 4 topic research into a deep vooronderzoek (3713 lines) eliminated redundant work without quality loss — viable when the technology scope is focused (single tech, Tauri 2 only).

---

## L-007: YAML Frontmatter Description Format

- **Date**: 2026-03-18
- **Context**: YAML frontmatter description fields in skill metadata need consistent formatting.
- **Finding**: YAML frontmatter description field requires careful attention — quoted strings work but folded block scalar (`>`) is the methodology standard for readability and YAML safety.
