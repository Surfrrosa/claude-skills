# Customization guide

Most skills work out of the box. A few need a small one-time setup so the audit logic knows what your project's canonical sources are.

## The project registry (all skills)

Each skill has a `## Project Registry` section near the top with a placeholder row:

```markdown
| Alias | Path |
|-------|------|
| `myapp` | (add your projects here) |
```

Replace `myapp` and the path with your own projects. You can list as many as you want:

```markdown
| Alias | Path |
|-------|------|
| `web` | `~/code/marketing-site` |
| `api` | `~/code/api` |
| `mobile` | `~/code/mobile-app` |
```

Now you can run `/a11y web` instead of `/a11y ~/code/marketing-site`.

This is just convenience — you can always pass a full path instead.

## Per-skill setup

### `/drift` — canonical sources

`/drift` finds hardcoded values that should reference a canonical source. To know what "canonical" means in your project, it needs you to point at the files that hold your project's truth.

Edit `~/.claude/skills/drift/SKILL.md`, find the Project Registry table, and add a "Canonical sources" column for each project:

```markdown
| Alias | Path | Canonical sources |
|-------|------|-------------------|
| `web` | `~/code/web` | `src/config.ts` (URLs, model IDs, prices), `src/lib/enums.ts` (status enums), `docs/BRAND_VOICE.md` |
```

If you don't have a config module, that's worth its own task — `/drift` will surface "you have hardcoded values everywhere but no canonical source," which is itself a useful finding.

### `/cohesion` — canonical layer

`/cohesion` finds pages that drift from your design system. It needs you to point at your canonical layer:

```markdown
| Alias | Path | Canonical layer |
|-------|------|-----------------|
| `web` | `~/code/web` | `src/layouts/Base.astro`, `src/styles/global.css`, `src/components/ui/*`, `docs/DESIGN_SYSTEM.md` |
```

If you don't have a canonical layer yet, `/cohesion` will tell you that's the first thing to bootstrap.

### `/coupling` — architectural intent

`/coupling` flags modules that import across architectural boundaries. To avoid flagging your INTENTIONAL couplings (a config module that everything imports from, an ORM file that's deliberately central, etc.), document them in `CLAUDE.md`:

```markdown
## Architectural intent

- `src/config.ts` is canonical for all configuration; high fan-in is by design.
- `src/lib/db.ts` is the single database client; high fan-in is by design.
- `src/calculations/*` must NOT import from `src/api/*` (pure-calculation layer).
- Other cross-layer imports should be flagged.
```

Or add it to the project registry:

```markdown
| Alias | Path | Architectural intent |
|-------|------|----------------------|
| `web` | `~/code/web` | Layered (api/ → services/ → db/). Intentional: `src/config.ts` (canonical). Forbidden: `services/` → `api/`. |
```

`/coupling` reads this and skips anything documented as intentional.

### `/full-sweep` — sub-skill installation

`/full-sweep` orchestrates other audit skills. It expects them to be installed in `~/.claude/skills/`. Skills included in this repo: `/a11y`, `/perf`, `/privacy`, `/thatsweird`, `/drift`, `/cohesion`, `/walkthrough`, `/coupling`.

Skills NOT in this repo (you'll need to source separately or substitute):
- `/zombie` — dead-code audit
- `/security-review` — Anthropic publishes one
- `/vuln` — dependency vulnerability scan
- `/sentry` — production-signal audit (needs Sentry account)

If a sub-skill is missing, `/full-sweep` logs "skipped (not installed)" and continues. You'll still get a useful run with whatever's installed.

## Tips

- **Don't overengineer the registry early.** Add projects as you run skills against them. Start with one alias.
- **Update CLAUDE.md before bootstrapping audit skills.** A 10-line "architectural intent" section in CLAUDE.md saves hours of false-positive triage later.
- **Run `/onboard` on a new project first.** It reads your existing docs and surfaces what's missing — including the canonical-source / canonical-layer / architectural-intent sections the other skills need.
