---
name: onboard
description: Digest a repository and become an expert before starting work
---

# Project Onboarding

Rapidly digest a repository's architecture, documentation, conventions, and current state so we hit the ground running with zero duplication of work.

## Sample output

A successful run produces something like:

```
## Project: marketing-site
Stack: Next.js 16 (App Router), TypeScript, Tailwind CSS, Framer Motion, Vercel
Purpose: Marketing site for a SaaS product, with blog and contact form
Current state: clean working tree, last commit added /pricing page

### Key files
- src/app/layout.tsx — root layout, fonts, metadata
- src/app/page.tsx — homepage with hero + features
- src/app/pricing/page.tsx — recently added, has TODO for annual toggle
- src/lib/posts.ts — blog post loading via gray-matter

### Conventions (from CLAUDE.md)
- Tailwind-first styling; only add custom CSS for what Tailwind can't do
- No em dashes; use commas / periods / colons
- All components TypeScript (.tsx); utilities (.ts)

### Next tasks (from latest session log, 2026-06-20)
1. Wire up the annual/monthly toggle on /pricing
2. Add OG image generation for blog posts
3. Audit accessibility before launch

### Known issues
- /contact form still uses mailto: — needs Resend or similar for production
- No 404 page styling

Ready to work.
```

## Arguments

The user will specify a project by name or path. Known project aliases:

| Alias | Path |
|-------|------|
| `myapp` | (add your projects here) |

If the user provides a path directly, use that. If a name isn't in the alias table, search for it.

## Onboarding Process (code repos)

### Phase 1: Project identity (30 seconds)
Read in this order, stopping at each one that exists:
1. `CLAUDE.md` (project instructions for Claude, highest priority)
2. `README.md`
3. `package.json` or `pyproject.toml` or `Cargo.toml` (tech stack)
4. `.claude/` directory contents (existing skills, settings)

### Phase 2: Architecture and documentation (60 seconds)
Search for and read:
1. `docs/ARCHITECTURE.md` or `docs/architecture.*` or `ARCHITECTURE.md`
2. `docs/INFRASTRUCTURE.md` or similar
3. Any brand/voice/style guides (`BRAND_VOICE.md`, `DESIGN_SYSTEM.md`, `STYLE_GUIDE.md`)
4. `docs/sessions/README.md` or session log index
5. **The most recent session log** (find the latest file in `docs/sessions/` or similar)
6. Any `TODO.md`, `ROADMAP.md`, or `CHANGELOG.md`

### Phase 3: Codebase structure (30 seconds)
Run:
```bash
# Directory structure (top 2 levels)
find [project-root] -maxdepth 2 -type d | head -50

# Key file types and counts
find [project-root] -name "*.py" -o -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" -o -name "*.html" -o -name "*.css" -o -name "*.md" | wc -l

# Git status
git -C [project-root] status
git -C [project-root] log --oneline -10
```

### Phase 4: Current state (30 seconds)
- Check for uncommitted changes
- Read the latest session log for: what was accomplished, next tasks, known issues, blockers
- Check for any open TODO items or planned work
- Identify the most recently modified files (likely where work was happening)

### Phase 5: Briefing
Present a concise briefing:

```
## Project: [Name]
**Stack:** [languages, frameworks, key dependencies]
**Purpose:** [one sentence]
**Current state:** [what was last worked on, any uncommitted changes]

## Key files
[5-10 most important files with one-line descriptions]

## Conventions
[coding style, branch strategy, deployment process, anything documented]

## Next tasks
[from session logs or TODO, prioritized]

## Known issues
[anything flagged in docs or session logs]

## Ready to work.
```

### Phase 6: CLAUDE.md gap check

After the briefing, check whether CLAUDE.md has a **"Before Writing New
Code"** section. If missing, flag it as a gap and offer to draft one,
customized to this repo's actual structure:

- Identify the shared-module locations (e.g. `src/lib/`, `src/utils/`,
  `data-collection/` — whatever exists)
- Identify the rg command tuned to the source dirs
- Name the existing helpers worth advertising (search for exports like
  `SITE_URL`, base classes, shared scanners, theme/config constants)
- Mention any base class / shared module that should be extended rather
  than copy-pasted

The section should end with: *"Don't add a new file for a one-off
variant of an existing pattern. If you're about to copy-paste a file
and tweak two values, extract a helper instead."*

Why this matters: every AI session reads CLAUDE.md before touching code,
so a "Before Writing" section there becomes a hard duplication
checkpoint. Memory rules don't load for coworkers or fresh agents;
CLAUDE.md does.

Don't add the section without asking — just flag the gap and propose.

## Important rules

- **Read first, talk second.** Don't start summarizing until you've actually read the docs.
- **Don't skip the session log.** The most recent session log is often the most valuable file for avoiding duplicate work.
- **Flag gaps.** If there's no CLAUDE.md, no docs, no session logs, say so. Suggest creating them.
- **Don't modify anything during onboarding.** This is read-only. We're learning, not changing.
- **If the project has a CLAUDE.md, follow its instructions.** That file overrides any defaults.
- **Adapt to what exists.** A mature project (like a mature codebase) has extensive docs. A smaller project might just have a README. Scale the briefing to match.


## Skill Run Logging

After completing this skill, append a log entry to `~/.claude/skill-log.md` under the Run Log table. New entries at the top (most recent first).

Format: `| YYYY-MM-DD | /skill-name | target (details) | one-line result |`
