# Contributing

Improvements welcome.

## Reporting issues

If a skill produces noise (false positives), misses something obvious, or breaks on a specific stack, open an issue with:

- Which skill (`/a11y`, `/drift`, etc.)
- Your stack (Next.js / Astro / Flask / React Native / etc.)
- What you expected
- What happened

Sample output is hugely helpful — even a short snippet of the finding that was wrong.

## Pull requests

Small PRs welcome. A few notes:

- **Keep skills stack-agnostic.** The patterns should work across the common stacks (Next.js, Astro, Flask, React Native, etc.). If a fix is stack-specific, gate it on stack detection rather than baking it into the core logic.
- **Test on a real project before submitting.** Generic-looking edits sometimes break specific stacks; battle-test on at least one of your own.
- **Don't add personal references.** No project names, no internal ticket IDs, no individual paths. Use `myapp` or `<your-project>` placeholders.
- **One skill per PR.** Easier to review.

## Adding a new skill

Skills follow this shape:

```markdown
---
name: <skill-name>
description: <one paragraph — surfaces in /help>
---

# <Skill Title>

<What it does, when to use it>

## Arguments

<--flags>

## Project Registry

| Alias | Path |
|-------|------|
| `myapp` | (add your projects here) |

## Process

<Step-by-step>

## Output

<What the user sees>

## Rules

<Constraints, edge cases>
```

If your skill has stack-specific behavior, follow the project-type detection pattern in `/full-sweep` — read `package.json` / `pyproject.toml` / etc., classify, and gate behavior on the result.
