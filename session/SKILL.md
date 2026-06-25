---
name: session
description: Generate an end-of-session log for any project
---

# Session Log Generator

Generate a session log summarizing what was accomplished in the current session. Works for any project.

## Arguments

The user may specify:
- No args: auto-detect the project from the current working directory
- A project name or path (e.g., `myapp`, `~/code/marketing-site`)

Parse the argument string. Examples:
- `/session` -- log for current project
- `/session synestrology` -- log for synestrology

## Project Aliases

| Alias | Path |
|-------|------|
| `myapp` | (add your projects here) |

If the user provides a path directly, use that. If a name isn't in the alias table, search for it.

## Process

### Step 1: Detect session log location

Check whether the project already has session logs:

1. Look for `docs/sessions/` directory (the established convention)
2. If it exists, read `docs/sessions/README.md` to understand the index format
3. If it doesn't exist, create `docs/sessions/` and a `README.md` index (see Bootstrap below)

### Step 2: Determine the filename

- Base format: `YYYY-MM-DD_session.md`
- Check if a session log already exists for today's date
- If it does, append a counter: `YYYY-MM-DD_session_2.md`, `YYYY-MM-DD_session_3.md`

### Step 3: Gather context

Collect information from multiple sources to build the log:

**From the conversation:**
- Review the conversation history to identify what was accomplished
- Note any decisions made, approaches tried and abandoned, trade-offs weighed

**From git:**
```bash
cd [project-root]
# What changed this session (files modified since last commit before session, or today's commits)
git log --since="$(date +%Y-%m-%d) 00:00" --oneline
git diff --stat HEAD
git diff --cached --stat
```

**From the project:**
- Read the previous session log (if any) to check whether planned "next tasks" were addressed
- Check for any new TODO/FIXME comments added during this session

### Step 4: Write the session log

Create `docs/sessions/YYYY-MM-DD_session.md` with this format:

```markdown
# Session: YYYY-MM-DD

## Status: Complete

## What was accomplished

### [Category 1]
- Bullet points of what was done
- Include file names and specific changes

### [Category 2]
- More bullets

## Files created
- List any new files (or "None")

## Files modified
- List any modified files (or "None")

## Key decisions
- Any decisions made about architecture, approach, brand, etc. (or "None")

## Next tasks (prioritized)
1. Most important next task
2. Second priority
3. etc.

## Known issues
- Any issues discovered (or "None")

## Blockers
- Any blockers (or "None")
```

### Step 5: Update the index

Add a one-line summary to the sessions table in `docs/sessions/README.md`.

Match the existing index format. If the README uses a table, add a table row. If it uses a bullet list, add a bullet. If you just bootstrapped it, use the table format shown in Bootstrap below.

### Step 6: Commit

Stage and commit the session log and README update:

```bash
cd [project-root]
git add docs/sessions/YYYY-MM-DD_session.md docs/sessions/README.md
git commit -m "Add session log for [date] ([brief summary])"
```

## Bootstrap (new projects)

When a project has no `docs/sessions/` directory:

1. Create `docs/sessions/`
2. Create `docs/sessions/README.md`:

```markdown
# Session Logs

| Date | Summary |
|------|---------|
```

3. Proceed with writing the session log as normal.

Note: if the project has no `docs/` directory at all, create `docs/` first. This is a standard convention.

## Notes

- Be specific about files and changes, not vague. "Updated Header component" is bad. "Added mobile nav toggle to `src/components/Header.tsx`, swaps text via `nav-full`/`nav-short` spans at 768px" is good.
- Prioritize next tasks based on what's time-sensitive or blocks other work.
- Include key decisions so future sessions don't re-litigate them. "Decided to use Stripe Checkout instead of custom form because..." is valuable context.
- Keep the one-line README summary under 200 characters.
- Check the previous session's "Next tasks" list. If any were completed, note that. If any were skipped, note why.
- Don't fabricate accomplishments. If the session was mostly debugging one issue, say that. A short honest log is more useful than a padded one.


## Skill Run Logging

After completing this skill, append a log entry to `~/.claude/skill-log.md` under the Run Log table. New entries at the top (most recent first).

Format: `| YYYY-MM-DD | /skill-name | target (details) | one-line result |`
