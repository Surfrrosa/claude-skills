---
name: full-sweep
description: Bundled foundational gauntlet. Orchestrates the audit-class skills (zombie / drift / cohesion / a11y / walkthrough / perf / thatsweird / security-review / privacy / vuln / sentry) in a single Plan Mode run. Identifies foundational debt across the codebase, auto-commits the mechanical wins, files the rest as Linear tickets. Subset modes for code-only, launch-readiness, or UX-focused passes. Use before launch, post-launch monthly, or weekly during dev (with the `code` subset). Not for single-PR review (use /simplify or /review) and not for runtime ops (use /pulse).
---

# Full Sweep — Foundational Gauntlet

Run the audit-class skills in dependency order, classify every finding as auto-fixable or human-decision, auto-commit the auto-fixables and file the rest. Closes the "found a bug → patch is reviewable" loop across 11 audit categories in one pass.

## When to use this

- **Pre-launch (highest payoff).** Full sweep once before App Store submission.
- **Post-launch.** Monthly full sweep keeps foundational debt from compounding.
- **During active dev.** Weekly `code` subset (zombie + drift + cohesion) catches structural drift early.
- **Pre-submission gate.** `launch` subset (security + privacy + a11y + perf + vuln) before any production build.
- **After major UX work.** `ux` subset (walkthrough + a11y + cohesion + thatsweird) after wholesale UX changes.

## When NOT to use this

- Single-PR review → use `/simplify` or `/review`
- Runtime ops monitoring → use `/pulse`
- Financial check → use `/cost`
- One-off targeted audit → invoke the sub-skill directly (`/zombie`, `/drift`, etc.)

## Arguments

| Argument | Effect |
|---|---|
| (none) | Full sweep on the current working directory's project — all 11 audits |
| project name | e.g. `slabcheck`, `synestrology`. Uses the alias table from `/onboard` / `/report` |
| `full` | Explicit full sweep (same as no arg) — all 11 audits |
| `code` | Subset — `zombie` + `drift` + `cohesion` + `coupling` |
| `launch` | Subset — `vuln` + `security-review` + `privacy` + `a11y` + `perf` |
| `ux` | Subset — `walkthrough` + `a11y` + `cohesion` + `thatsweird` |
| `--report-only` | Skip auto-fix and ticket-filing phases. Output findings only. |
| `--no-tickets` | Apply auto-fixes but skip the Linear-ticket-filing phase. |

Subset and project args combine: `/full-sweep slabcheck code` runs the code-subset on slabcheck.

## Process

### Phase 0: Pre-flight (fail fast)

Verify everything before doing any work:

```bash
git status                                  # working tree must be clean
git branch --show-current                   # must be on main (or default branch)
git fetch origin && git status -uno         # must be up to date with origin
```

Then run the project's pre-build check:

```bash
npm run prebuild-check  # or project equivalent
```

**If any pre-flight fails:** halt. Report the specific blocker (uncommitted changes / not on main / behind origin / tests failing). Do NOT proceed — running a sweep against a dirty tree pollutes findings + risks losing work.

### Phase 0.5: Project type detection

Different project types support different audits. Web-only audits (`/cohesion`, `/seo`, `/thatsweird`) self-flag N/A for React Native projects but `/full-sweep` should detect upstream to avoid wasted invocations and noisy "skipped" findings. Detection logic:

1. **Read `package.json`** (if exists). Classify:
   - Contains `"react-native"` or `"expo"` in dependencies → `project_type: "react-native"`
   - Contains `"next"` → `project_type: "web-next"`
   - Contains `"vite"` / `"vue"` / `"svelte"` → `project_type: "web-spa"`
   - Else with package.json → `project_type: "node-backend"`

2. **Else read `pyproject.toml` / `requirements.txt`** (if exists). Classify:
   - Contains `"fastapi"` / `"django"` / `"flask"` / `"starlette"` → `project_type: "python-web"`
   - Else → `project_type: "python-backend"`

3. **Else** (Cargo.toml → `"rust"`, Gemfile → `"ruby"`, etc.) or unknown → `project_type: "generic"`

Persist `project_type` for use in Phase 2's applicability gating.

**Applicability matrix** (`project_type` → which sub-skills to run):

| Sub-skill | react-native | web-next / web-spa / python-web | python-backend / node-backend / generic |
|-----------|--------------|----------------------------------|------------------------------------------|
| /vuln | ✓ | ✓ | ✓ |
| /security-review | ✓ | ✓ | ✓ |
| /privacy | ✓ | ✓ | ✓ |
| /zombie | ✓ | ✓ | ✓ |
| /drift | ✓ | ✓ | ✓ |
| /perf | ✓ | ✓ | ✓ |
| /sentry | ✓ | ✓ | ✓ |
| /a11y | ✓ | ✓ | skip |
| /walkthrough | ✓ | ✓ | skip |
| /cohesion | **skip** | ✓ | skip |
| /thatsweird | **skip** | ✓ | skip |

Cell legend: `✓` = run normally; `skip` = log "Skipped /{skill} — not applicable to {project_type}" and continue. `generic` is conservative — when type is unknown, skip everything that requires a UI surface.

### Phase 1: Audit sequence (read-only — no Plan Mode yet)

**Do NOT enter Plan Mode here.** Plan Mode's workflow restricts Phase 1 to Explore-subagent calls and would block the `Skill` tool invocations below. Sub-audit skills are read-only by design, so running them in normal mode is safe. Plan Mode is reserved for the EXECUTION gate in Phase 4 — the moment that matters is approving the synthesized fix queue + ticket batch in one shot, not approving the discovery process.

### Phase 2 (legacy: Phase 2 of original design — see Phase 1 above)

(Sub-audits run here per Phase 1.)

Run audits in **dependency-layer order**. The order matters: dep updates can invalidate downstream findings; dead-code removal cleans the surface before structural drift checks; production-signal data (Sentry) validates the rest at the end.

For each audit:
1. Invoke the sub-skill via the `Skill` tool with the project's target path
2. Capture every finding
3. Tag each with severity (RED / YELLOW / GRAY per /walkthrough convention) AND auto-fix-eligibility (`auto_fix: true|false`)
4. Append to the running master findings list

**Full sweep order:**

| # | Audit | Why this position |
|---|---|---|
| 1 | `/vuln` | Deps first — a CVE can change priorities downstream |
| 2 | `/security-review` | Security findings set scope for everything else |
| 3 | `/privacy` | Paired with security — overlap on secret scanning, run together to dedupe |
| 4 | `/zombie` | Dead code before linting — don't audit code we're about to delete |
| 5 | `/drift` | Hardcoded values, after dead code is gone |
| 6 | `/cohesion` | Design consistency, after structural drift is fixed |
| 7 | `/a11y` | Accessibility — runs against the cleaned-up surface |
| 8 | `/walkthrough` | UX state-machine + empty-state coverage |
| 9 | `/perf` | Performance budgets — last-mile quality |
| 10 | `/thatsweird` | Browser/OS edge cases — defensive cleanup |
| 11 | `/sentry` | Production patterns last — validates the rest against real user pain |

**Subset orders** preserve relative ordering, just skip the others:

- `code`: 4, 5, 6, +coupling (zombie → drift → cohesion → coupling). Coupling runs last in the code subset since it consumes the cleaned-up surface — dead imports removed by /zombie + canonical sources enforced by /drift produce a more accurate import graph.
- `launch`: 1, 2, 3, 7, 9 (vuln → security → privacy → a11y → perf)
- `ux`: 6, 7, 8, 10 (cohesion → a11y → walkthrough → thatsweird)

**Per-audit invocation pattern (with project-type gating):**

```
For each sub-skill in the configured order:
  if applicability_matrix[project_type][sub_skill] == "skip":
    log: "Skipped /{sub_skill} — not applicable to {project_type} project"
    record in audit summary with status="skipped"
    continue
  Use Skill tool: { skill: sub_skill, args: "<project>" }
```

Pass-through args: target project name resolves from the alias table.

### Phase 2.5: Verification gate

Sub-audit skills (especially `/zombie` and `/drift`) produce findings of mixed quality. Without a verification gate, claims like "X is unused" where X is actually used in many files, references to nonexistent directories, and canonical sources flagged as drift can all slip through. Synthesizing all raw findings directly is too expensive and ships bad fixes.

**The fix:** spawn one verification agent per sub-skill's findings (in parallel, since sub-skills are independent), mechanically check each claim against the actual filesystem, classify, and pass only verified + needs-human findings forward to Phase 3 synthesis.

**Verification agent prompt template:**

```
You are verifying findings from a /{skill} audit on {project}. Each finding
claims something about the codebase. Your job is to mechanically verify each
claim against the actual filesystem and classify it as one of:

- VERIFIED: the claim is mechanically true (grep / file read / line check confirms)
- FALSE_POSITIVE: the claim is mechanically false (claimed unused symbol is used,
  claimed nonexistent file exists, etc.)
- NEEDS_HUMAN: the claim involves judgment (design quality, naming, architectural
  preference) that can't be mechanically verified

Required mechanical checks per claim type:
- "X is unused" / "X is dead code" → grep -rn "X" across full codebase, exclude
  the definition site, count remaining usages. If > 0: FALSE_POSITIVE with the
  usage locations as proof.
- "X is in file Y at line Z" → read Y, confirm content at line Z matches the
  claimed snippet. If line/file doesn't match: FALSE_POSITIVE.
- "X duplicates Y" → read both files, compare. If not actually duplicated:
  FALSE_POSITIVE.
- "X is missing" → check the claimed location. If present: FALSE_POSITIVE.
- "X is a design / architecture / naming issue" → NEEDS_HUMAN (can't verify
  mechanically).

Output JSON: { verified: [...], false_positive: [...], needs_human: [...] }
with one-line reason per item, including grep counts / line content as proof
for FALSE_POSITIVE classifications.

DO NOT verify by re-reasoning. Only use grep, file reads, and line checks.
If you can't mechanically verify, classify as NEEDS_HUMAN, not VERIFIED.
```

**Why mechanical-only:** the verifier is itself an LLM and could hallucinate if asked to re-reason about claims. Restricting verification to mechanical checks (grep, read, line) gives deterministic ground truth. The distinction matters: "is `useFoo` imported anywhere?" is mechanical; "is the `useFoo` API design good?" is interpretive — those go to NEEDS_HUMAN.

**Output handling:**

- `verified` + `needs_human` findings → pass to Phase 3 synthesis as the master findings list
- `false_positive` findings → logged in the Phase 6 report but NOT actioned (visible per-audit so we can see which sub-skills are consistently noisy → signal for improving the sub-skill itself)

**Parallelism:** verification agents are independent (one per sub-skill output) and should be spawned in parallel in a single message with multiple Agent tool calls.

### Phase 3: Synthesize plan

After all audits return, build the master plan:

1. **Dedupe overlapping findings.** Common overlap zones:
   - `security-review` ∩ `privacy` (secret scanning + PII handling)
   - `cohesion` ∩ `drift` (design tokens + hardcoded styles)
   - `walkthrough` ∩ `a11y` (interactive component completeness)

   When two audits flag the same root cause, merge into one entry and attribute to both audits in the report.

2. **Classify** every remaining finding as one of:
   - **Auto-fix** — mechanical, ≤10-line fix, no design judgment required, sub-skill has Step-6-equivalent rules. (`/walkthrough` is currently the only sub-skill with Step 6 implemented — others auto-degrade to "file as ticket" until their auto-fix rules are written. See "Known gaps" below.)
   - **File as ticket** — needs design / architectural / legal judgment, OR is too large for a single mechanical fix.
   - **Defer** — known issue with explicit memory rationale. Note + skip.

3. **Group tickets** by project + milestone:
   - Pre-launch class → V1 Launch project / Pre-launch QA fixes milestone (severity RED + YELLOW)
   - Post-launch class → Operations project / Backlog (severity GRAY + post-launch YELLOW)
   - Use the project's labeling rules (synestrology + slabcheck only)

4. **Write the plan document** to `~/.claude/plans/full-sweep-<project>-<YYYY-MM-DD>.md`. Structure:

   ```
   # Full Sweep — <project> — <date>

   ## Pre-flight
   - Working tree clean ✓
   - On main, up to date with origin ✓
   - prebuild-check passing ✓

   ## Audit summary

   | Audit | Findings | Auto-fix | File | Defer |
   |---|---|---|---|---|
   | vuln | 3 | 0 | 3 | 0 |
   | security-review | 2 | 0 | 2 | 0 |
   | ... | | | | |

   ## Auto-fix queue (will be committed after approval)

   ### chore(zombie): apply N auto-fixable findings
   - file/path.tsx:42 — <one-line fix description>
   - ...

   ### chore(drift): apply N auto-fixable findings
   - ...

   ## Linear tickets to be filed

   ### V1 Launch / Pre-launch QA milestone
   - **<title>** (Priority: High, severity/critical) — <one-line description>
   - ...

   ### Operations / Backlog
   - **<title>** (Priority: Medium, Improvement) — <one-line description>
   - ...

   ## Deferred
   - <finding> — <reason this is intentionally not actioned>
   ```

### Phase 4: Plan Mode approval gate

After the master plan file is written (Phase 3), call `EnterPlanMode` followed immediately by `ExitPlanMode`. ExitPlanMode reads the plan file you just wrote and presents it for one-shot user approval.

**The user approves the entire batch once.** No per-audit re-confirmation.

If the user rejects or modifies the plan, halt execution and ask what to change. Don't apply partial state.

Plan Mode exists in this skill specifically to gate the EXECUTION phase (auto-commits + Linear ticket filing in Phase 5), not the discovery phase. That's the actual valuable role: one approval before any writes hit the codebase or Linear.

### Phase 5: Execute (post-approval, autonomous)

After approval:

1. **Apply auto-fixes, one audit at a time.** For each audit's auto-fix queue:
   - Apply the fixes via Edit / Write
   - Run `npm run prebuild-check` (or project equivalent)
   - **If prebuild-check fails:** abort that audit's commit. Move all of that audit's auto-fix findings into the "file as ticket" queue. Continue with the next audit.
   - **If prebuild-check passes:** commit with subject `chore(<audit>): apply N auto-fixable findings from /full-sweep` and a body listing the files + one-line per fix.

2. **File Linear tickets.** Use `mcp__linear-slabcheck__save_issue` (or the right MCP for the project). Batch by milestone. Apply project labels.

3. **Do NOT push.** all commits stay local. The user pushes when they're ready.

4. **Do NOT auto-merge** any open PRs surfaced during audit. Surface them in the report, defer to user.

### Phase 6: Report

Final output to the user:

```
## /full-sweep complete — <project> — <date>

### Project type detected
<react-native | web-next | web-spa | python-web | python-backend | node-backend | generic>

### Summary

| Audit | Status | Raw | Verified | False+ | Needs human | Auto-fixed | Filed | Deferred |
|---|---|---|---|---|---|---|---|---|
| vuln | ran | 4 | 3 | 1 | 0 | 0 | 3 | 0 |
| zombie | ran | 12 | 2 | 10 | 0 | 1 | 1 | 0 |
| cohesion | skipped (RN) | — | — | — | — | — | — | — |
| ... | | | | | | | | |
| **Total** | | X | V | F | H | Y | Z | W |

False-positive rate per audit is visible so consistently noisy sub-skills become obvious — signal for improving the sub-skill itself, not for tolerating the noise.

### Commits ready to push (local)
- abc1234 chore(zombie): apply 5 auto-fixable findings from /full-sweep
- def5678 chore(drift): apply 3 auto-fixable findings from /full-sweep
- ...

### Linear tickets filed
- SLA-XXX <title> [V1 Launch / Pre-launch QA]
- SLA-YYY <title> [Operations / Backlog]
- ...

### Deferred (with reasons)
- ...

### Recommended next actions
- Review the N pending commits, push when ready
- The K Urgent / High tickets in V1 Launch are the gate for next milestone
- Estimated cleanup time for the ticket backlog: <rough estimate>
```

## Concerns + known gaps

### Sub-skill auto-fix coverage


> Audits default to opening a small PR for auto-fixable findings + filing rest as Linear. /walkthrough has Step 6 implemented; pattern needs propagating to /perf /a11y /cohesion /seo /security-review /privacy.

**As of skill creation, only `/walkthrough` has Step 6 (the auto-fix taxonomy + bundle workflow).** Other sub-skills don't yet classify findings as `auto_fix: true|false`. `/full-sweep` handles this gracefully:

- If a sub-skill returns findings with `auto_fix` tags → apply per-skill auto-fix workflow.
- If a sub-skill returns findings without `auto_fix` tags → treat ALL as `file as ticket`, no auto-commit attempted.

The first `/full-sweep` run will therefore file more tickets and auto-commit fewer fixes than steady-state. As Step 6 propagates to the other audit skills (filed as separate work), the ratio shifts toward auto-fix.

### Token / time cost

Running 11 sub-audits sequentially is expensive (~30-60 minutes wall time, significant token cost). Use subset modes when appropriate. Don't run full sweep more often than monthly post-launch unless you have a specific reason.

### Overlap dedupe is judgment

The 3 known overlap zones (security/privacy, cohesion/drift, walkthrough/a11y) are explicit. Other overlaps exist but are less common. When merging two findings into one, attribute to both audits in the report so future-reader knows both flagged it.

### Stop-on-blocker mode is NOT supported

Even if `/security-review` finds something critical, `/full-sweep` runs all sub-audits. Prioritization happens at synthesis. Halting mid-sweep hides parallel issues + makes the run non-reproducible.

If a finding is so critical it needs immediate attention before other work, the user can interrupt the sweep and re-run after addressing.

## Important rules

- **Plan Mode wraps the execution phase, not discovery.** Sub-audits are read-only and run freely in Phases 1-2.5. Plan Mode is entered in Phase 4 immediately before the approval gate. User approves once via `ExitPlanMode`. After approval, execution is autonomous up to the final report.
- **Project-type-aware.** Phase 0.5 detects the project type; Phase 2 skips sub-skills that don't apply (e.g., `/cohesion` + `/thatsweird` skip for React Native). Reduces noise + saves token cost on inapplicable invocations.
- **Verification gate is mechanical-only.** Phase 2.5 verifies each finding via grep / file read / line check before synthesis. Never re-reasons. Interpretive claims (design quality, naming) go to NEEDS_HUMAN, not VERIFIED. Keeps raw sub-audit noise from contaminating the final synthesis.
- **Auto-commit, never auto-push.** The user controls when commits go to origin.
- **Auto-fix is conservative.** If a "mechanical" fix turns out to need judgment, downgrade to "file as ticket" rather than guess.
- **prebuild-check is the gate.** If any auto-fix batch fails prebuild-check, abort that batch's commit and re-route to tickets. Never commit broken state.
- **Project-scoped ticket filing.** Only file Linear issues for projects where that's configured. For other repos, write findings to a single audit report file inside the repo instead.
- **Output policy.** Foundational fixes only, structured report, confident forward-motion summary.
- **Don't propagate Step 6 mid-sweep.** Adding auto-fix rules to a sub-skill is its own project. `/full-sweep` consumes whatever the sub-skills currently emit.

## Skill Run Logging

After completing this skill, append a log entry to `~/.claude/skill-log.md` under the Run Log table. New entries at the top (most recent first).

Format: `| YYYY-MM-DD | /full-sweep | target (subset, project, type) | N raw findings → V verified, F false-positive, H needs-human; X auto-fixed, Y filed, Z deferred |`

False-positive count is now load-bearing — it tells you whether the noise rate is dropping over time as sub-skills improve, or whether a specific sub-skill (e.g., `/zombie`) needs its own refactor.
