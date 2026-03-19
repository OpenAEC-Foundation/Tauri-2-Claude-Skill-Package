# Tauri 2 Claude Skill Package

**Deterministic Claude AI skills for Tauri 2 desktop application development.**

Built on the [Agent Skills](https://agentskills.org) open standard.

**Status: COMPLETE — 27 skills across 5 categories**

## Skill Summary

| Category | Count | Purpose |
|----------|-------|---------|
| `tauri-core/` | 3 | Architecture, configuration, runtime lifecycle |
| `tauri-syntax/` | 8 | Commands, events, state, permissions, plugins, windows, webviews, menus |
| `tauri-impl/` | 10 | Project setup, build/deploy, mobile, testing, security, migration, database, design patterns, multi-window, plugin development |
| `tauri-errors/` | 4 | Build errors, IPC errors, permission errors, runtime errors |
| `tauri-agents/` | 2 | Project scaffolder, code review checklist |
| **Total** | **27** | |

See [INDEX.md](INDEX.md) for the complete skill catalog with descriptions and trigger scenarios.

## Tech Coverage

| Area | Skills | Topics |
|------|--------|--------|
| **Commands & IPC** | 3 | Tauri commands, invoke(), argument types, return types, error handling, Channel streaming |
| **Events** | 1 | Event system, emit/listen/once, global vs window-scoped, payloads |
| **Plugins** | 2 | Official plugin APIs (fs, dialog, http, etc.), custom plugin development |
| **Window & Webview** | 3 | Window management, multi-window, webview API, multi-webview |
| **State** | 1 | Managed state, Mutex/RwLock, AppHandle.state() |
| **Permissions & Security** | 3 | Capabilities, ACL, CSP, isolation pattern, scope control |
| **Mobile** | 1 | Android/iOS targets, platform-specific code, mobile entry points |
| **Build & Deploy** | 2 | Bundlers, code signing, auto-updater, CI/CD, sidecar binaries |
| **Architecture** | 2 | Project structure, design patterns, IPC decision trees |
| **Migration** | 1 | Tauri 1.x to 2.x migration guide |
| **Testing** | 1 | Rust unit tests, mockIPC, WebDriver E2E |
| **Database** | 1 | SQLite, tauri-plugin-store, sqlx/diesel integration |
| **Error Debugging** | 4 | Build, IPC, permissions, runtime error diagnosis |
| **Agents** | 2 | Project scaffolding, code review validation |

## Installatie

### Claude Code

```bash
# Optie 1: Clone het volledige pakket
git clone https://github.com/OpenAEC-Foundation/Tauri-2-Claude-Skill-Package.git
cp -r Tauri-2-Claude-Skill-Package/skills/source/ ~/.claude/skills/tauri/

# Optie 2: Voeg toe als git submodule
git submodule add https://github.com/OpenAEC-Foundation/Tauri-2-Claude-Skill-Package.git .claude/skills/tauri
```

### Claude.ai (Web)

Upload individuele SKILL.md bestanden als project knowledge.

## Quick Start

Na installatie worden skills automatisch geactiveerd op basis van context. Voorbeelden:

- **Nieuw project starten**: vraag Claude om een Tauri 2 app te maken — activeert `tauri-impl-project-setup` en `tauri-agents-project-scaffolder`
- **Command schrijven**: schrijf een `#[tauri::command]` — activeert `tauri-syntax-commands`
- **Build error debuggen**: plak een build error — activeert `tauri-errors-build`
- **Permissions instellen**: configureer capabilities — activeert `tauri-syntax-permissions`
- **Code review**: vraag om een Tauri project review — activeert `tauri-agents-review`

## Built With

Dit pakket is ontwikkeld met de **7-fase research-first methodologie**:

1. **Setup + Raw Masterplan** — Projectstructuur en governance bestanden
2. **Deep Research (Vooronderzoek)** — Uitgebreid bronnenonderzoek van Tauri 2 documentatie, source code, en community resources
3. **Masterplan Refinement** — Verfijning van skill inventaris op basis van onderzoeksresultaten
4. **Topic-Specific Research** — Diepgaand onderzoek per skill topic
5. **Skill Creation** — Deterministische skill bestanden schrijven volgens Agent Skills standaard
6. **Validation** — Controle op correctheid, completheid, en consistentie
7. **Publication** — GitHub release en documentatie

## Related Packages

- [SolidJS-Claude-Skill-Package](../SolidJS-Claude-Skill-Package/) — Frontend framework
- [Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package](../Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package/) — Proven methodology template

## License

MIT

---

Part of the [OpenAEC Foundation](https://github.com/OpenAEC-Foundation) ecosystem.
