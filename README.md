# claude-skills

A free audit toolkit for [Claude Code](https://claude.com/claude-code). Nine slash commands for accessibility, performance, SEO, privacy, design, deploy safety, and workflow.

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

### Foundations

| Skill | What it does |
|-------|--------------|
| `/a11y` | Accessibility (WCAG) audit on source code and optionally live URLs. Can auto-fix safe categories. |
| `/perf` | Performance audit. Bundle size, image weight, render-blocking resources, and Core Web Vitals via PageSpeed. |
| `/seo` | SEO health audit across projects. Score them, identify gaps, optionally fix. |
| `/privacy` | Audit data collection, consent flows, exposed secrets, and privacy policy accuracy. |

### Design

| Skill | What it does |
|-------|--------------|
| `/design` | Design psychology audit. Asks "does the system serve the user?" not "does the page match the system?" |
| `/thatsweird` | Browser and OS edge-case sweep. Chrome auto-dark inversion on dark sites, iOS rubber-band, prefers-reduced-motion violations. |

### Workflow

| Skill | What it does |
|-------|--------------|
| `/onboard` | Digest a repository's architecture, docs, conventions, and current state before starting work. |
| `/ship` | Pre-deploy safety checklist. Catches build failures, leaked secrets, debug artifacts, missing env vars. |
| `/session` | End-of-session log generator. What changed, decisions made, next steps. |

## A note on customization

Each skill resolves projects from a small registry table at the top of the file. The placeholder is `myapp`; add your own projects as you go. Each skill also references conventions specific to certain stacks (Next.js, Astro, Flask/Jinja, static HTML, React Native). They handle the common stacks gracefully but you may want to tweak the detection patterns or the auto-fix rules for your setup.

## License

MIT. Use them, fork them, share them.

## Issues

Something off? Open an issue.
