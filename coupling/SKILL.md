---
name: coupling
description: Orthogonality / module-coupling audit. Asks the Pragmatic Programmer's central orthogonality question — "if I change feature X, how many modules light up?" — and surfaces the answer with mechanical evidence. Finds circular imports, cross-layer architecture violations, centralization magnets (large file × high fan-in), hidden coupling via git file-co-change patterns, and wildcard / dead imports. Use as a periodic full-codebase sweep (monthly, or before launches that grow surface area), or after a wholesale architectural refactor.
---

# Coupling Audit

Find every place in the codebase where independence broke down. Orthogonality, in The Pragmatic Programmer's framing, is the property that changing one feature touches one module. When that breaks — when a request lights up half the import graph — the cost compounds: changes get expensive, regressions hide, refactors stall.

This audit produces a mechanical answer to the orthogonality question for every module in the project, then applies judgment to surface the *surprising* cases — the ones not already documented as intentional in `CLAUDE.md`.

This is **not** a code-quality audit. It's specifically coupling / orthogonality. Different from:
- Dead-code audits — dead code / unused things
- Drift audits — hardcoded values that should reference canonical sources
- Cohesion / design-system audits — visual + structural design drift
- Reuse / quality audits on changed code
- UX coverage audits — missing-feature gaps

## Why this exists

A tightly-coupled module can pass every other audit and still be the source of compounding pain. Dead-code audits check what's unused. Drift audits check what's stale. Cohesion audits check what's inconsistent. None answer *"if I dramatically change feature X, how many modules are affected?"* That's this skill's job.

## Arguments

- No args: audit current working directory (must be a project root with `CLAUDE.md`)
- A project alias (e.g., `myapp`) — see registry below
- `--scope <path>`: limit analysis to a subdirectory (e.g., `--scope src/components`)
- `--category=<name>`: limit to one category (fan, layer, cycles, centralization, co-change, imports)
- `--report-only`: skip the Step 6 auto-fix bundle, emit findings only

## Project Registry

| Alias | Path | Architectural intent (seeds the allowlist) |
|---|---|---|
| `myapp` | (add your projects here) | (describe your intentional couplings here: config singletons, mandated single files, enforced layer boundaries) |

For non-registered projects, attempt to derive intentional couplings from `CLAUDE.md`'s "Before Writing" / "BEFORE WRITING NEW CODE" section. If absent, flag and stop.

## Process

### Step 0 — Seed allowlist from CLAUDE.md

Read in order:
1. `CLAUDE.md` — especially the "BEFORE WRITING NEW CODE" section. Every entry that mandates a single canonical helper or single source of truth is an *intentional* coupling. Do not flag these.
2. Existing isolation tests (e.g., custom CI tests that protect layer boundaries) — patterns already enforced; defer to them, don't duplicate.
3. The drift gauntlet's `PROJECT_CONFIG` if present — for canonical source files. These are allowed to have high fan-in by design.

Extract into a working allowlist:
- **Canonical singletons** — single-file truths intentionally imported widely (config modules, single ORM files, single prompt sources)
- **Architectural boundaries** — directories that must NOT import from another direction
- **Pre-enforced rules** — any isolation already protected by a test; we don't re-flag it

This becomes the "noise filter" the audit applies before reporting. Get this wrong and every run is noise.

### Step 1 — Compute mechanical signals

No LLM judgment yet. Run the following on the project's source directory:

**1a. Fan-in / fan-out per module.** Parse all imports (`import X`, `from X import Y`, both absolute and relative). For each module, count:
- *Fan-in* (afferent coupling, Ca): how many other modules import it
- *Fan-out* (efferent coupling, Ce): how many other modules it imports
- *Instability*: `I = Ce / (Ca + Ce)` — Martin's metric. 0 = stable (lots of dependents, few dependencies); 1 = unstable (lots of dependencies, no dependents)

**1b. Cross-layer import detection.** For projects with declared layers (read from CLAUDE.md or infer from top-level src/ directory names), check whether any module imports across a forbidden boundary. Example shape: a pure-calculation module is supposed to be web-framework-free and must not import from `routes/` or `controllers/`. Defer to existing tests if they already cover this; flag only NEW directions not yet protected.

**1c. Circular import detection.** Build the import graph; find strongly connected components of size > 1. Each cycle is a finding.

**1d. Centralization-risk score.** For each file, compute `LOC × fan_in`. Files above the 90th percentile in this product are "centralization magnets" — touching them is high-blast-radius. Flag the top 10, MINUS anything on the allowlist from Step 0.

**1e. Git file-co-change matrix.** Run `git log --name-only --since="90 days ago" --pretty=format:""` and tally pairs of files that appear in the same commit. For each pair with ≥3 co-changes and a co-change rate ≥ 50% (changed together at least half the time either changed), flag as "hidden coupling." These pairs often suggest a missing abstraction or a logical module spread across files.

**1f. Wildcard + dead-import scan.** `from X import *` is namespace pollution and a coupling magnifier (consumers don't even know what they depend on). Unused imports are dead weight that creates phantom coupling in static analysis. Both are mechanical and auto-fixable.

For each signal, emit a structured record: `{file, signal_type, value, threshold, evidence}`.

### Step 2 — Spawn parallel Explore agents over signals

Launch 2–3 agents in parallel. Each takes a slice of the signal categories, applies judgment, and produces structured findings.

**Agent prompts must include:**
- The mechanical signal evidence (specific file:line, counts, thresholds)
- The Step 0 allowlist (intentional couplings — do not re-flag)
- The severity ladder below
- Output format: `{file, signal_type, severity, mechanical_evidence, judgment, suggested_action, auto_fix_eligible}`
- Word cap (500–800 words per agent report)

**Severity ladder:**

- **HIGH** — Active risk: circular imports (any cycle is a bug waiting), cross-layer architecture violations not yet protected by a test, centralization magnets where the file already shows historical coupling pain (e.g., consistently appears in unrelated PRs' diffs).
- **MEDIUM** — Risk-of-future-pain: high fan-in modules NOT on the allowlist, file co-change pairs with ≥70% correlation suggesting missing abstraction, wildcard imports.
- **LOW** — Style debt: dead imports, redundant re-exports, instability metric outliers without other supporting signal.

### Step 3 — Consolidate findings + triage

Each agent reports findings as structured markdown. Consolidate into a single table sorted by severity:

```
| File | Signal | Severity | Evidence | Judgment | Suggested action | Auto-fix? |
|------|--------|----------|----------|----------|------------------|-----------|
```

Triage rules:
- HIGH: file issue immediately (if your team tracks them)
- MEDIUM: file at lower priority
- LOW: catalog in umbrella; don't file individually

### Step 4 — Verification gate (mechanical re-check)

Every finding must pass a mechanical re-check before reaching output:

- **"X has fan-in of N"** → re-grep for X across project, count importers, confirm.
- **"X has a circular dependency with Y"** → trace the cycle path explicitly, file:line by file:line.
- **"X is a cross-layer violation"** → confirm the import statement on the claimed line, confirm the layer assignment via CLAUDE.md.
- **"X and Y co-change 80% of the time"** → re-run `git log --name-only` filtered to both files, confirm the ratio.
- **Judgment-only claims (e.g., "this is a centralization risk")** → demote to NEEDS_HUMAN; do not include in HIGH/MEDIUM without a mechanical anchor.

Findings that fail re-check are dropped, NOT silently demoted. Log the false-positive count per signal type so you can see which signals are noisy and improve them.

### Step 5 — File issues (optional)

If your team uses an issue tracker, file at this point. Pattern:

1. File umbrella issue: title `Audit: module coupling + orthogonality across <project>`, priority High. Description includes the consolidated finding table grouped by signal type.
2. For each HIGH finding, file a sub-issue with title naming the file + signal, body including the mechanical evidence + judgment + suggested action + estimated work.
3. MEDIUM findings filed as sub-issues at priority Medium.
4. LOW findings catalogued in the umbrella body, no individual issues.

If you don't use an issue tracker, write the report to `docs/audits/coupling-YYYY-MM-DD.md` instead.

### Step 6 — Auto-fix bundle (default ON; suppress with `--report-only`)

After Step 5, partition findings by `auto_fix_eligible`.

**Auto-fixable (`auto_fix: true`)** — single mechanical answer, ≤10 lines per finding, no architectural judgment:

| Finding pattern | Fix |
|---|---|
| Unused / dead imports | Delete the import line. Verify nothing in the file references the symbol. |
| `from X import *` (wildcard import) | Replace with explicit `from X import a, b, c` listing the names actually used. Run tests; if anything breaks, downgrade to `auto_fix: false`. |
| Redundant re-exports (module re-exports a name solely so another module can import it from the wrong place) | Update consumers to import from the canonical source. Delete the re-export line if no consumers remain. |

**Not auto-fixable (`auto_fix: false`) — file as issues:**

- Circular imports (the right break point is judgment — which dependency direction is "correct" depends on the domain)
- Cross-layer violations (the fix is "move this code somewhere else" — architectural decision)
- Centralization-risk hotspots (splitting a large central file into smaller files is design work, not mechanical)
- High file-co-change correlation (the right abstraction extraction is judgment)

**Workflow:**

1. Verify a clean working tree on the project's main branch (`git status` clean, `git pull --ff-only`).
2. Create a branch named `coupling-fixes-YYYY-MM-DD`.
3. Apply each fix as a small reviewable hunk.
4. Run the project's test suite. If anything fails, abort and report — do NOT commit broken state.
5. Commit with `coupling: <one-line summary of fix categories>`.
6. Push the branch.
7. Open a PR titled `coupling: fix <category 1>, <category 2>, ...`. PR body lists each finding fixed + each follow-up filed for non-auto-fixable findings + a test plan.
8. Output to caller: PR URL + each follow-up URL.

**Skip Step 6 entirely** if `--report-only` was passed.

**Abort Step 6 mid-flight** if: tests fail after applying fixes; a "mechanical" fix turns out to need judgment; the user has an unmerged PR open on the project; more than 5 fixes accumulated (split into two PRs for tractable review).

### Step 7 — Output the audit report

Final output to user:

```
## Coupling Audit — <project> — <date>

### Headline finding
<the single most consequential coupling issue, if any>

### By signal type
| Signal | HIGH | MEDIUM | LOW | False+ |
|--------|------|--------|-----|--------|
| Fan-in / fan-out | N | N | N | N |
| Cross-layer | N | N | N | N |
| Circular imports | N | N | N | N |
| Centralization risk | N | N | N | N |
| File co-change | N | N | N | N |
| Wildcards / dead imports | N | N | N | N |

### Allowlist applied
- N intentional couplings excluded (config singletons, mandated ORMs, etc.)

### Filed issues
- <umbrella URL> + N sub-issues (HIGH) + N sub-issues (MEDIUM)
- N LOW findings catalogued in umbrella

### Auto-fix PR
- <branch> — N fixes applied — <PR URL>

### Signal-to-noise
- Verification gate dropped N findings as false positives (X% rate)
- If FP rate > 50%: signal threshold needs tuning before encoding into CI
- If FP rate ≤ 50%: ready to encode this baseline as a CI guardrail

### Recommended next actions
- <prioritized list>
```

## What guards quality (skill-internal)

The skill produces actionable findings, not just a list. Each HIGH finding must include:

1. **Concrete mechanical evidence** — file:line, import counts, cycle path, co-change ratio. No "this feels coupled" findings.
2. **Allowlist check passed** — finding is NOT one of the documented intentional couplings.
3. **Customer / dev impact** — why this matters (regression risk, refactor cost, hidden blast radius).
4. **Fix shape** — what the PR / architectural change would look like in 1–2 sentences.
5. **Auto-fix eligibility** — true/false with reason.

If the skill can't fill all 5 for a finding, demote to MEDIUM/LOW or drop it. False positives erode trust in the audit.

## Anti-patterns / what NOT to flag

The skill avoids four classes of false positive:

1. **Documented intentional couplings.** Anything in CLAUDE.md's "BEFORE WRITING NEW CODE" section as a canonical-singleton is not coupling debt. Common examples: a single config module, a single ORM file, a single canonical prompt source. These are *high fan-in by design*.

2. **Layer boundaries already enforced by tests.** If a test already protects a layer boundary, don't re-flag any clean direction — the test would have caught a violation already.

3. **Single-direction utility dependencies.** A module that's imported widely BUT imports nothing itself (instability = 0) is a stable utility, not a coupling problem. Flag only when high fan-in combines with high LOC (centralization-risk score) or with cross-layer violation.

4. **Test files.** `tests/` directories legitimately import broadly. Skip entirely unless analyzing the test suite itself as a target.

## Companion: CI baseline

Once this skill has run on a project and the false-positive rate is acceptable (≤50%), the patterns it surfaced can be encoded as CI guardrails: a `tests/test_coupling_budget.py` (or equivalent) that fails when fan-in / centralization / co-change exceeds the baseline. The skill's mechanical signals become the test's thresholds. Don't encode thresholds before running the skill at least once — encoding a priori is the band-aid to avoid.

## Important rules

- **Allowlist is load-bearing.** Step 0 must be done thoroughly. Documented intentional couplings flagged as findings is the skill failing, not finding a problem.
- **Verification gate is mechanical-only.** No "this feels coupled" findings reach output. If the verifier can't re-check mechanically, demote to NEEDS_HUMAN; never silently keep it.
- **Auto-commit, never auto-push without authorization.** All Step 6 PRs are pushed to remote for review but never merged automatically — merge is explicit authorization.
- **Don't reinvent a dead-code audit.** Dead imports overlap with dead-code audit territory. Use a dead-code audit's detection if it's already been run on the project recently; otherwise flag the dead imports here and cross-reference.
- **Don't reinvent a drift audit.** Hardcoded values that should reference canonical sources are a drift audit's domain. This skill's "centralization magnet" finding doesn't include "X hardcodes a config value" — that's a separate audit.

## What this skill is NOT

- Not a refactor planner. Findings include suggested actions, not committed refactor plans. Decisions about how to break a circular import or split a centralized module are architectural and belong in a separate Plan Mode session.
- Not a code-quality audit. High coupling can coexist with clean code; low coupling can coexist with bad code. This skill measures one dimension.
- Not a dependency-injection rewrite. Module-level imports are the unit of analysis. The skill flags the coupling; the *fix* may or may not involve DI patterns, and that's not the skill's call.

## Skill Run Logging

If you keep a skill-run log, append an entry after each run:

Format: `| YYYY-MM-DD | /coupling | <project> (<args>) | <one-line headline finding> + N HIGH / N MED / N LOW; X false-positive rate |`
