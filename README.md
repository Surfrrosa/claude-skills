# claude-skills

A free audit toolkit for [Claude Code](https://claude.com/claude-code). The slash commands I run on my own work, calibrated for my workflow.

## What's a skill?

A skill is a reusable slash command for Claude Code. It's a markdown file that lives in `~/.claude/skills/<name>/SKILL.md`. When you type `/<name>` in Claude Code, the skill runs.

## Install

### One skill

```bash
mkdir -p ~/.claude/skills/full-sweep && \
  curl -L https://raw.githubusercontent.com/surfrrosa/claude-skills/main/full-sweep/SKILL.md \
  -o ~/.claude/skills/full-sweep/SKILL.md
```

Swap `full-sweep` for any skill name in this repo.

### All of them

```bash
git clone https://github.com/surfrrosa/claude-skills.git ~/claude-skills-tmp && \
  cp -r ~/claude-skills-tmp/*/ ~/.claude/skills/ && \
  rm -rf ~/claude-skills-tmp
```

Then restart Claude Code (or run `/help`) so it picks up the new skills.

## What's included

### The toolkit

| Skill | What it does |
|-------|--------------|
| `/full-sweep` | Orchestrates the audit-class skills in a single Plan Mode run. Auto-commits the mechanical wins, files the rest. |
| `/drift` | Finds every place code drifts from a canonical source of truth (hand-typed data, magic numbers, stale copy). |
| `/cohesion` | Finds every place a page drifts from the project's design system. |
| `/coupling` | Orthogonality audit. Asks the Pragmatic Programmer's question, "if I change feature X, how many modules light up?" |
| `/walkthrough` | UX coverage audit. State-machine gaps, missing features, entry-state combinations nobody walked through. |
| `/thatsweird` | Browser and OS edge-case sweep. Chrome auto-dark inversion, iOS rubber-band, prefers-reduced-motion violations. |
| `/design` | Design psychology audit. Asks "does the system serve the user?" not "does the page match the system?" |

### Foundations

| Skill | What it does |
|-------|--------------|
| `/a11y` | Accessibility (WCAG) audit. Source code and optionally live URLs. |
| `/perf` | Performance audit. Bundle size, image weight, render-blocking resources, Core Web Vitals. |
| `/seo` | SEO health audit across projects. Score, identify gaps, optionally fix. |
| `/privacy` | Data collection, consent flows, exposed secrets, privacy policy accuracy. |

### Workflow

| Skill | What it does |
|-------|--------------|
| `/onboard` | Digest a repository's architecture, docs, conventions, and current state before starting work. |
| `/ship` | Pre-deploy safety checklist. Catches build failures, leaked secrets, debug artifacts, missing env vars. |
| `/session` | End-of-session log generator. What changed, decisions made, next steps. |

## Calibrated for my workflow

These skills reference my projects (synestrology, slabcheck, surfrrosa), my issue tracker (Linear), my deploy stack (Vercel, Railway, EAS), and my file structure. They work out of the box, but to fit your setup, you'll want to swap the project aliases, ticketing references, and any tool-specific commands.

## License

MIT. Use them, fork them, share them.

## Issues

Something off? Open an issue.
