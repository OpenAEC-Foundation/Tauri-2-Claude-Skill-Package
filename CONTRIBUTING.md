# Contributing

Thank you for your interest in improving the Tauri 2 Claude Skill Package.

## How to Contribute

### Adding or Improving Skills

1. Read `WAY_OF_WORK.md` for methodology.
2. Read `REQUIREMENTS.md` for quality guarantees.
3. Check `SOURCES.md` for approved references.
4. Follow skill structure (`SKILL.md` + `references/`).
5. Validate before submitting.

### Skill Quality Standards

- `SKILL.md` MUST be under 500 lines.
- English only.
- YAML frontmatter with `name` and `description`.
- Deterministic language (ALWAYS/NEVER) — no vague recommendations.
- Both Rust and TypeScript examples for IPC skills.
- All code verified against official Tauri v2 docs.
- Anti-patterns documented with explanations.

### Commit Message Format

Use Conventional Commits: `feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:`

Phase commits use the format: `"Phase X.Y: [action] [subject]"`

### Pull Request Process

1. Fork the repository.
2. Create a feature branch.
3. Follow the skill creation process in `WAY_OF_WORK.md`.
4. Ensure all skills pass validation.
5. Update `ROADMAP.md` if adding new skills.
6. Submit PR with description of changes.

## Code of Conduct

Be respectful, constructive, and inclusive.

## License

By contributing, you agree that your contributions will be licensed under MIT.
