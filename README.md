# claude-skills

A free audit toolkit for [Claude Code](https://claude.com/claude-code). Fourteen slash commands covering accessibility, performance, SEO, privacy, design, deploy safety, and codebase audits.

## What's a skill?

A skill is a reusable slash command for Claude Code. It's a markdown file that lives in `~/.claude/skills/<name>/SKILL.md`. When you type `/<name>` in Claude Code, the skill runs.

## Install

### One skill

```bash
mkdir -p ~/.claude/skills/a11y && \
  curl -L https://raw.githubusercontent.com/Surfrrosa/claude-skills/main/a11y/SKILL.md \
  -o ~/.claude/skills/a11y/SKILL.md
```

Swap `a11y` for any skill name in this repo.

### All of them

```bash
git clone https://github.com/Surfrrosa/claude-skills.git ~/claude-skills-tmp && \
  cp -r ~/claude-skills-tmp/*/ ~/.claude/skills/ && \
  rm -rf ~/claude-skills-tmp
```

Then restart Claude Code (or run `/help`) so it picks up the new skills.

## What's included

### The toolkit (audit-class skills)

| Skill | What it does |
|-------|--------------|
| `/full-sweep` | Orchestrates the audit-class skills in a single Plan Mode run. Auto-commits the mechanical wins, files the rest as issues. |
| `/drift` | Finds every place code drifts from a canonical source of truth: hand-typed reference data, model IDs bypassing config, hardcoded URLs / emails / prices, magic numbers, year-bound constants, copy that fell out of sync with brand docs. |
| `/cohesion` | Finds every place a page drifts from the project's design system. Different button styles, hard-coded colors that should be tokens, parallel classes recreating canonical components, inline styles redefining global variables. |
| `/coupling` | Orthogonality audit. Asks the Pragmatic Programmer's question — "if I change feature X, how many modules light up?" — with mechanical evidence (fan-in / fan-out, circular imports, cross-layer violations, git co-change patterns). |
| `/walkthrough` | UX coverage audit. State-machine gaps, missing features (no logout, no password reset), entry-state combinations nobody walked through, pre-hydration render flashes. |
| `/thatsweird` | Browser and OS edge-case sweep. Chrome auto-dark inversion on dark sites, iOS rubber-band flash, iOS input zoom, prefers-reduced-motion violations. |
| `/design` | Design psychology audit. Asks "does the system serve the user?" not "does the page match the system?" Evaluates typography appropriateness, color register, brand-voice ↔ visual match. |

### Foundations

| Skill | What it does |
|-------|--------------|
| `/a11y` | Accessibility (WCAG) audit on source code and optionally live URLs. Can auto-fix safe categories. |
| `/perf` | Performance audit. Bundle size, image weight, render-blocking resources, and Core Web Vitals via PageSpeed. |
| `/seo` | SEO health audit across projects. Score them, identify gaps, optionally fix. |
| `/privacy` | Audit data collection, consent flows, exposed secrets, and privacy policy accuracy. |

### Workflow

| Skill | What it does |
|-------|--------------|
| `/onboard` | Digest a repository's architecture, docs, conventions, and current state before starting work. |
| `/ship` | Pre-deploy safety checklist. Catches build failures, leaked secrets, debug artifacts, missing env vars. |
| `/session` | End-of-session log generator. What changed, decisions made, next steps. |

## A note on customization

Each skill resolves projects from a small registry table at the top of the file. The placeholder is `myapp`; add your own projects as you go. Each skill also references conventions specific to certain stacks (Next.js, Astro, Flask/Jinja, static HTML, React Native). They handle the common stacks gracefully but you may want to tweak the detection patterns or the auto-fix rules for your setup.

### Sub-skill dependencies for `/full-sweep`

`/full-sweep` is an orchestrator. It expects sub-audits to be installed in `~/.claude/skills/`. This repo includes most of them; a few (`/zombie`, `/security-review`, `/vuln`, `/sentry`) you'll need to source separately or substitute equivalents.

## License

MIT. Use them, fork them, share them.

## Issues

Something off? Open an issue.
