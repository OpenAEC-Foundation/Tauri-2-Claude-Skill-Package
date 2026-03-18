# Approved Sources Registry

All skills in this package MUST be verified against these approved sources. No other sources are authoritative.

## Official Documentation

| Source | URL | Purpose |
|--------|-----|---------|
| Tauri v2 Documentation | https://v2.tauri.app/ | Primary reference for all Tauri 2 content |
| Tauri v2 Guides | https://v2.tauri.app/start/ | Getting started, concepts, features |
| Tauri v2 JS API Reference | https://v2.tauri.app/reference/javascript/ | Complete frontend API reference |
| Tauri Rust Crate Docs | https://docs.rs/tauri/latest/tauri/ | Complete Rust crate docs |
| Tauri Config Reference | https://v2.tauri.app/reference/config/ | tauri.conf.json schema |
| Tauri Security/Permissions | https://v2.tauri.app/security/ | Permissions, capabilities, CSP |

## Source Code

| Source | URL | Purpose |
|--------|-----|---------|
| Tauri Core Monorepo | https://github.com/tauri-apps/tauri | Main repo (core, API, CLI, bundler) |
| Tauri Plugins Workspace | https://github.com/tauri-apps/plugins-workspace | Official plugin collection |
| create-tauri-app | https://github.com/tauri-apps/create-tauri-app | Project scaffolding |
| Tauri Examples | https://github.com/tauri-apps/tauri/tree/dev/examples | Official code examples |

## Crate/Package Docs

| Source | URL | Purpose |
|--------|-----|---------|
| tauri crate | https://docs.rs/tauri/latest/tauri/ | Rust API docs |
| tauri-build crate | https://docs.rs/tauri-build/latest/tauri_build/ | Build script |
| tauri-plugin crate | https://docs.rs/tauri-plugin/latest/tauri_plugin/ | Plugin framework |
| @tauri-apps/api (npm) | https://www.npmjs.com/package/@tauri-apps/api | Frontend TS/JS API |
| @tauri-apps/cli (npm) | https://www.npmjs.com/package/@tauri-apps/cli | CLI tool |

## Migration & Release Notes

| Source | URL | Purpose |
|--------|-----|---------|
| v1 to v2 Migration | https://v2.tauri.app/start/migrate/from-tauri-1/ | Migration reference |
| Tauri Blog | https://v2.tauri.app/blog/ | Release announcements |
| GitHub Releases | https://github.com/tauri-apps/tauri/releases | All version changelogs |

## Community (for Anti-Pattern Research)

| Source | URL | Purpose |
|--------|-----|---------|
| GitHub Issues | https://github.com/tauri-apps/tauri/issues | Real-world error patterns |
| GitHub Discussions | https://github.com/tauri-apps/tauri/discussions | Q&A, patterns |
| Tauri Discord | https://discord.com/invite/tauri | Community help |

## Claude / Anthropic (Skill Development Platform)

| Source | URL | Purpose |
|--------|-----|---------|
| Agent Skills Standard | https://agentskills.io | Open standard |
| Agent Skills Spec | https://github.com/agentskills/agentskills | Specification |
| Agent Skills Best Practices | https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices | Authoring guide |

## OpenAEC Foundation

| Source | URL | Purpose |
|--------|-----|---------|
| ERPNext Skill Package | https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package | Methodology template |
| Blender Skill Package | (reference only) | Proven 73-skill package |

## Source Verification Rules

1. **Primary sources ONLY** — Official docs, official repos, official crate/npm docs.
2. **NEVER trust random blog posts** — Even popular ones may be outdated or wrong for v2.
3. **Verify code against official docs** — Every code snippet in a skill MUST match current API.
4. **Note when source was last verified** — Track in the table below.
5. **Cross-reference if docs are sparse** — When official docs lack detail, verify against source code in the monorepo.

## Last Verified

| Technology | Date | Action | Notes |
|------------|------|--------|-------|
| Tauri | 2026-03-18 | Initial setup | All URLs pending verification |
