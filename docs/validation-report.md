# Validatierapport Tauri 2 Claude Skill Package

**Datum**: 2026-03-19
**Totaal skills gevonden**: 27 (taak specificeerde 28 -- geen ontbrekende skill geidentificeerd in de verwachte categorien)

## Legenda

- **Lines**: Aantal regels in SKILL.md (max 500)
- **Frontmatter**: Geldige YAML met name, description, license, compatibility, metadata
- **References**: methods.md, examples.md, anti-patterns.md aanwezig
- **Language**: Alleen Engels (geen Nederlands of andere talen)
- **Deterministic**: Gebruikt ALWAYS/NEVER, niet "you should" / "consider"
- **CritWarn**: Critical Warnings sectie aanwezig
- **DualLang**: IPC-gerelateerde skills tonen zowel Rust als TypeScript voorbeelden
- **Status**: PASS / WARN / FAIL

## Validatietabel

### tauri-core (3 skills)

| Skill | Lines | Frontmatter | References | Language | Deterministic | CritWarn | DualLang | Status |
|-------|-------|-------------|------------|----------|---------------|----------|----------|--------|
| tauri-core-architecture | 298 | PASS | PASS | PASS | PASS | PASS | PASS (refs) | PASS |
| tauri-core-config | 334 | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| tauri-core-runtime | 302 | PASS | PASS | PASS | PASS | PASS | PASS | PASS |

### tauri-syntax (8 skills)

| Skill | Lines | Frontmatter | References | Language | Deterministic | CritWarn | DualLang | Status |
|-------|-------|-------------|------------|----------|---------------|----------|----------|--------|
| tauri-syntax-commands | 400 | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| tauri-syntax-events | 265 | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| tauri-syntax-state | 249 | PASS | PASS | PASS | PASS | PASS | WARN | WARN |
| tauri-syntax-window | 334 | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| tauri-syntax-webview | 326 | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| tauri-syntax-menu | 350 | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| tauri-syntax-permissions | 335 | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| tauri-syntax-plugins-api | 391 | PASS | PASS | PASS | PASS | PASS | PASS | PASS |

### tauri-impl (10 skills)

| Skill | Lines | Frontmatter | References | Language | Deterministic | CritWarn | DualLang | Status |
|-------|-------|-------------|------------|----------|---------------|----------|----------|--------|
| tauri-impl-project-setup | 410 | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| tauri-impl-plugin-development | 299 | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| tauri-impl-multi-window | 342 | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| tauri-impl-mobile | 346 | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| tauri-impl-build-deploy | 395 | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| tauri-impl-testing | 443 | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| tauri-impl-migration | 251 | PASS | PASS | PASS | PASS | PASS | PASS (refs) | PASS |
| tauri-impl-security | 346 | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| tauri-impl-database | 435 | PASS | PASS | PASS | PASS | WARN | PASS | WARN |
| tauri-impl-design-patterns | 498 | PASS | PASS | PASS | PASS | WARN | PASS | WARN |

### tauri-errors (4 skills)

| Skill | Lines | Frontmatter | References | Language | Deterministic | CritWarn | DualLang | Status |
|-------|-------|-------------|------------|----------|---------------|----------|----------|--------|
| tauri-errors-ipc | 422 | PASS | PASS | PASS | PASS | WARN | PASS | WARN |
| tauri-errors-build | 400 | PASS | PASS | PASS | PASS | WARN | PASS | WARN |
| tauri-errors-permissions | 413 | PASS | PASS | PASS | PASS | WARN | PASS | WARN |
| tauri-errors-runtime | 466 | PASS | PASS | PASS | PASS | WARN | PASS | WARN |

### tauri-agents (2 skills)

| Skill | Lines | Frontmatter | References | Language | Deterministic | CritWarn | DualLang | Status |
|-------|-------|-------------|------------|----------|---------------|----------|----------|--------|
| tauri-agents-review | 385 | PASS | PASS | PASS | PASS | WARN | PASS (refs) | WARN |
| tauri-agents-project-scaffolder | 450 | PASS | PASS | PASS | PASS | WARN | PASS | WARN |

## Samenvatting

### Volledig PASS: 19 van 27 skills

### Warnings (niet-blokkerend): 8 skills

**CritWarn missend (8 skills)**: De volgende skills missen een expliciete `## Critical Warnings` sectie op top-level. Sommige hebben wel `### Critical Rules` als subsecties, maar geen consistente top-level heading:

1. `tauri-agents-project-scaffolder` -- geen critical warnings sectie
2. `tauri-agents-review` -- heeft "Critical Issues" in review template, maar geen eigen CW sectie
3. `tauri-errors-build` -- heeft regels verspreid in tekst maar geen CW sectie
4. `tauri-errors-ipc` -- heeft `### Critical Rules` subsectie maar geen `## Critical Warnings`
5. `tauri-errors-permissions` -- heeft `### Critical Rules` subsecties maar geen `## Critical Warnings`
6. `tauri-errors-runtime` -- geen critical warnings sectie
7. `tauri-impl-database` -- heeft `### Critical Rules` subsecties maar geen `## Critical Warnings`
8. `tauri-impl-design-patterns` -- geen critical warnings sectie

**DualLang warning (1 skill)**:

1. `tauri-syntax-state` -- Heeft 8 Rust codeblokken maar 0 TypeScript codeblokken in SKILL.md. Slechts 1 TS voorbeeld in references/examples.md. Dit is een IPC-gerelateerde skill die zou moeten laten zien hoe state vanuit de frontend wordt benaderd via invoke().

### Blockers: GEEN

Alle skills voldoen aan de harde eisen:
- Alle SKILL.md bestanden zijn onder 500 regels (max: 498 bij design-patterns)
- Alle YAML frontmatter is geldig met vereiste velden
- Alle references/ directories bevatten de 3 vereiste bestanden
- Geen Nederlands of andere talen gevonden in skill content
- Geen non-deterministisch taalgebruik gevonden ("you should", "consider", etc.)

### Aanbevelingen

1. **Hoge prioriteit**: Voeg TypeScript voorbeelden toe aan `tauri-syntax-state/SKILL.md` die laten zien hoe managed state wordt benaderd vanuit frontend invoke() calls
2. **Medium prioriteit**: Overweeg een consistent `## Critical Warnings` top-level sectie toe te voegen aan de 8 skills die dit missen, zodat het formaat uniform is over alle skills
3. **Let op**: `tauri-impl-design-patterns` staat op 498 regels -- zeer dicht bij de 500-regel limiet
