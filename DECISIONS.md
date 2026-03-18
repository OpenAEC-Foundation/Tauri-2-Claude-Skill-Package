# DECISIONS

Architectural and process decisions with rationale. Each decision is numbered and immutable once recorded. New decisions may supersede old ones but old ones are never deleted.

---

## D-001: 7-Phase Research-First Methodology
**Date**: 2026-03-18
**Status**: ACTIVE
**Context**: Need a structured approach to build high-quality skills
**Decision**: Adopt the 7-phase methodology proven in the ERPNext Skill Package and Blender-Bonsai Skill Package
**Rationale**: ERPNext project successfully produced 28 domain skills, Blender-Bonsai produced 73 skills with this approach. Research-first prevents hallucinated content.
**Reference**: https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package/blob/main/WAY_OF_WORK.md

## D-002: Single Technology Package
**Date**: 2026-03-18
**Status**: ACTIVE
**Context**: Tauri 2 is one technology (unlike the Blender package which covers 4+ technologies)
**Decision**: No per-technology separation needed. All skills share the `tauri-` prefix under a single `skills/source/` tree.
**Rationale**: Tauri is a single framework. Splitting into sub-packages would add complexity without benefit. The Rust backend and TypeScript frontend are two sides of the same framework, not separate technologies.

## D-003: English-Only Skills
**Date**: 2026-03-18
**Status**: ACTIVE
**Context**: Team works primarily in Dutch, skills target international audience
**Decision**: ALL skill content in English only
**Rationale**: Skills are instructions for Claude, not end-user documentation. Claude reads English and responds in any language. Bilingual skills double maintenance with zero functional benefit. Proven in ERPNext and Blender projects.
**Reference**: ERPNext LESSONS_LEARNED.md, lesson on English-only skills

## D-004: Claude Code Agent Tool for Orchestration
**Date**: 2026-03-18
**Status**: ACTIVE
**Context**: Need to produce ~30 skills efficiently. Windows environment, no oa-cli available.
**Decision**: Use Claude Code Agent tool for parallel execution instead of oa-cli
**Rationale**: Windows environment does not support oa-cli (requires WSL/Linux). Claude Code Agent tool provides native parallelism within the Claude Code session. Simpler setup, no tmux/fcntl dependencies.

## D-005: MIT License
**Date**: 2026-03-18
**Status**: ACTIVE
**Context**: Need to choose open-source license
**Decision**: MIT License
**Rationale**: Most permissive, maximizes adoption. Consistent with OpenAEC Foundation philosophy.

## D-006: ROADMAP.md as Single Source of Truth
**Date**: 2026-03-18
**Status**: ACTIVE
**Context**: Need to track project status across multiple sessions and agents
**Decision**: ROADMAP.md is the ONLY place where project status is tracked
**Rationale**: Multiple status locations cause drift and confusion. Single source prevents "which is current?" questions. Proven in both ERPNext and Blender projects.

## D-007: Tauri 2 Only
**Date**: 2026-03-18
**Status**: ACTIVE
**Context**: Tauri has two major versions with significantly different APIs
**Decision**: No Tauri 1.x skills. All code targets Tauri v2.x exclusively. A migration guide may exist but all skill content is v2 only.
**Rationale**: Tauri 1.x is legacy. The v2 API is a complete rewrite with different patterns (plugins, permissions, capabilities). Supporting both would dilute quality and create confusion.

## D-008: Dual Coverage — Rust + TypeScript
**Date**: 2026-03-18
**Status**: ACTIVE
**Context**: Tauri apps have a Rust backend and TypeScript/JavaScript frontend communicating via IPC
**Decision**: Every IPC-related skill MUST show both Rust and TypeScript sides
**Rationale**: Tauri's core value proposition is the Rust-JS bridge. A skill that shows only the Rust command without the TypeScript invoke call (or vice versa) is incomplete and will lead to incorrect code.

## D-009: SKILL.md < 500 Lines
**Date**: 2026-03-18
**Status**: ACTIVE
**Context**: Skills need to be concise enough for Claude to load efficiently
**Decision**: SKILL.md must be under 500 lines. Heavy content goes in references/
**Rationale**: Anthropic convention. ERPNext skills ranged 180-427 lines, all well under 500. Keeps the main skill focused on decision trees and quick reference.

## D-010: Official Plugins = Syntax, Custom Plugins = Impl
**Date**: 2026-03-18
**Status**: ACTIVE
**Context**: Need to categorize plugin-related skills correctly
**Decision**: Using official Tauri plugins (tauri-plugin-fs, tauri-plugin-shell, etc.) is API consumption and belongs in syntax/. Building a custom plugin from scratch is a development workflow and belongs in impl/.
**Rationale**: Consumption vs creation are fundamentally different activities. A developer using `tauri-plugin-fs` needs method signatures (syntax). A developer building `tauri-plugin-myfeature` needs architecture guidance (impl).

## D-011: Existing Directory Structure Preserved
**Date**: 2026-03-18
**Status**: ACTIVE
**Context**: Repository already has `skills/source/tauri-{category}/` directories
**Decision**: Preserve the existing directory structure. Skills go into `skills/source/tauri-{category}/{skill-name}/`
**Rationale**: Structure was created intentionally and matches the naming convention. No reason to restructure.

## D-012: WebFetch for Research
**Date**: 2026-03-18
**Status**: ACTIVE
**Context**: Tauri 2 is relatively new and documentation evolves rapidly
**Decision**: Use WebFetch tool for all research to ensure latest documentation is consulted
**Rationale**: Claude's training data has a knowledge cutoff. Tauri 2 APIs may have changed since cutoff. WebFetch ensures research uses actual latest docs, not stale training data. This is critical for API accuracy guarantees.
