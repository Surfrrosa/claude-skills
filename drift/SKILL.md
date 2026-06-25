---
name: drift
description: Hardcoded-values audit. Finds every place code drifts from canonical sources of truth — hand-typed reference data that should come from a library, model IDs that bypass central config, email/URL/price literals that duplicate canonical constants, magic numbers in business logic, lists that duplicate enums, year-bound constants that silently expire, embedded copy that drifts from brand voice docs. Outputs findings and recommends new test guardrails. Use as a periodic full-codebase sweep (monthly, or before major launches), or when a user-visible drift bug surfaces.
---

# Drift Audit

Find every place the code drifts from a canonical source of truth. Catch the kind of mistakes that ship customer-visible bugs: hand-typed reference data that diverges from the source library, paid features running on stale model IDs because the call site never got updated when central config did, URL literals that break across environments.

This is **not** a code-quality audit. It's specifically drift detection. Different from:
- Dead code audits — dead code / unused things
- Reuse / quality audits on changed code
- Cohesion / design-system audits — visual + structural design drift
- UX coverage audits — missing-feature gaps
- Accessibility audits

## Why this exists

Hand-typed reference data, hardcoded model IDs that bypass central config, magic numbers scattered across business logic — these are the most common source of "we shipped the wrong value" bugs. Static type checking doesn't catch them; they're syntactically valid. They surface when the canonical source changes and the hardcoded callers silently don't.

Mechanical drift hooks at keystroke time (e.g., a pre-commit hook or CI test that greps for known patterns) catch the categories you've already seen. This skill finds NEW patterns the hook doesn't know about yet, AND patterns that need LLM judgment to evaluate.

## Arguments

- No args: audit current working directory (must be a project root with `CLAUDE.md`)
- A project alias (e.g., `myapp`) — see registry below
- `--category=<name>`: limit to one category (data, models, urls, emails, prices, lists, magic, copy)
- `--propose-hooks`: include "add this to your CI test suite" recommendations for mechanizable findings

## Project Registry

| Alias | Path | Canonical sources |
|---|---|---|
| `myapp` | (add your projects here) | (add your projects here) |

For non-registered projects, attempt to derive canonical sources from `CLAUDE.md`'s "Before Writing" section. If absent, flag and stop.

## Process

### Step 0 — Identify canonical sources

Read in order:
1. `CLAUDE.md` — especially the "Before Writing" section (lists which helpers/constants are canonical)
2. `config.py` / `config.ts` / `config.js` or equivalent (constants, env vars, API keys, model IDs, pricing)
3. Calculation modules / enums / shared data sources
4. Voice / brand contracts (`BRAND_VOICE.md`, banned-phrases lists, etc.)
5. Existing mechanical drift tests if present — these are the patterns the hook already catches; don't duplicate them, find what's missing.

Extract:
- **Canonical constants** (base URLs, model IDs, email addresses, prices, payment-processor IDs, database refs)
- **Canonical enums** (any type-set with display labels)
- **Canonical data sources** (libraries that compute reference data your code might hand-type)
- **Voice rules** (banned phrases, em-dash bans, brand-voice register)

This becomes the "expected canonical layer" the audit compares against.

### Step 1 — Spawn parallel Explore agents

Launch 2–3 agents in parallel. Each takes a slice of the 8 categories below. Group by overlap in target files for efficiency.

**Agent prompts must include:**
- Specific patterns to grep for (with regex)
- Allowlist criteria (what's legit to hardcode)
- Severity ladder (HIGH = drift already biting / imminent expiration; MEDIUM = drift risk waiting to happen; LOW = style debt)
- Output format (file:line + snippet + category + severity + canonical replacement)
- Word cap (500–800 words per agent report)

### Step 2 — The 8 categories to sweep

#### Category 1: Hand-typed reference data that a library would compute

**What it looks like:** A file declares a list / dictionary / array of values that look like they were computed elsewhere and then copy-pasted. Common shapes: arrays of dates with labels, tables of numeric values for known phenomena, "lookup" tables that mirror what a library function returns.

**Why it bites:** Hand-typed reference data has been wrong before. Type a number, get the precision wrong, miss a leap year, copy-paste a row twice — all silent bugs. The canonical library never makes those mistakes; the copy of it does.

**Canonical replacement:** Call the source library at runtime. If the data needs to be precomputed for performance, generate it from the library via a build-time script, never by hand.

**Allowlist:** Test fixtures, computed-ground-truth snapshots tagged as such with a comment.

**Hook candidate?** Yes — pattern-detectable per project.

#### Category 2: Year-bound module constants

**What it looks like:** `HOLIDAYS_2026 = [...]`, `EVENTS_2026`, `SCHEDULE_YYYY`. Any UPPER_SNAKE_CASE name ending in `_YYYY`.

**Why it bites:** Silently expires at the year boundary. Cron jobs return `None` forever. Customer calendars go empty. The bug surfaces months after the original commit.

**Canonical replacement:** Compute at runtime. If the data isn't computable, use a year-agnostic name + a generation script that runs on demand.

**Hook candidate?** Yes — universal pattern across projects.

#### Category 3: Hardcoded model IDs / API client config

**What it looks like:** `"claude-sonnet-4-..."`, `"claude-opus-4-..."`, `"claude-haiku-4-..."`, OpenAI model strings, Stripe price IDs (`price_xxx`), Supabase project refs, etc.

**Why it bites:** When the canonical config bumps the model/price/version, hardcoded callers silently stay on the old one. A common shape: paid feature running on an older model than what central config specifies.

**Canonical replacement:** Import from your config module (e.g., `ANTHROPIC_*_MODEL`, `STRIPE_*_PRICE_ID`, `SUPABASE_URL`).

**Allowlist:** The config module itself (it IS the source), inline `# audit-ok: model-id-explicit` comments for intentional version freezes.

**Hook candidate?** Yes — substring detection of known SDK model names is mechanical.

#### Category 4: Email addresses / URLs / domain literals

**What it looks like:** `"hello@yourapp.com"`, `"https://www.yourapp.com"`, `"@yourapp.com"` suffixes for UIDs. Any literal that ties the code to a specific domain or environment.

**Why it bites:** If the domain changes, ANY rebrand, or if dev/staging environments need different URLs, the hardcoded literals break silently. Past audits have found many sites with hardcoded prod URLs that broke dev environments.

**Canonical replacement:** `config.EMAIL_DOMAIN`, `config.BASE_URL`, `config.SENDER_*`, `config.PUBLIC_CONTACT_EMAIL`.

**Allowlist:** The config module itself; static HTML meta tags that legitimately need the production URL (og:url, canonical) — flag but don't fail.

**Hook candidate?** Yes — domain literal regex check.

#### Category 5: Hardcoded prices / costs / tier limits / magic numbers

**What it looks like:** `40` (dollar amount), `4000` (cents), `25` (annual fee), `3` (some business-rule window), `365` (expiry), `5/min` (rate limit), `6000` (max_tokens).

**Why it bites:** Same drift pattern — when one changes, everyone has to find every site. Magic numbers cluster: find one, find a category.

**Canonical replacement:** Named constants in your config module (e.g., `EMAIL_SEND_WINDOW_DAYS`, `GIFT_CODE_EXPIRY_DAYS`).

**Allowlist:** Local loop bounds, math constants (pi), test fixtures.

**Hook candidate?** Partial — can detect prices via `price_xxx` / `$XX` regex, but business-logic magic numbers need LLM judgment. The skill flags these; the user decides which deserve extraction.

#### Category 6: Hand-typed lists duplicating canonical enums

**What it looks like:** `["Aries", "Taurus", ...]` in any file other than the canonical source. `["Generator", "Projector", ...]`. Regex alternations of enum names in validators.

**Why it bites:** If a label is renamed centrally, duplicated lists silently drift. Typo accidents.

**Canonical replacement:** Import the enum, use `[t.value for t in YourEnum]` or `YourEnum.NAMES` or similar exported list.

**Allowlist:** The canonical source file itself.

**Hook candidate?** Partial — can detect via "this exact list appears more than once in src/" check, but distinguishing intentional fixtures from drift needs context.

#### Category 7: Embedded prompt / template copy that drifts from brand voice

**What it looks like:** Em dashes (—) or en dashes (–) in string literals that look like prompts (triple-quoted, >100 chars, addressing the reader). Banned phrases embedded in strings outside the canonical list.

**Why it bites:** Past audits have found em dashes IN THE PROMPT TELLING THE LLM NOT TO USE EM DASHES. Contradicting itself. Prompts that ship to the model influence model output.

**Canonical replacement:** Replace em dashes with colons / commas / period restructuring. Banned phrases just removed.

**Allowlist:** Comments + docstrings (style debt, not customer-visible). Rule definitions that SHOW the banned char as an example (e.g., `"Don't use em dashes (—)"` is fine).

**Hook candidate?** Partial — em-dash regex inside triple-quoted strings is doable; banned-phrase detection in source-code strings is reachable.

#### Category 8: Duplicated type → label mappings

**What it looks like:** Two or more files that map `"raw_type"` → `"Display Label"` independently.

**Why it bites:** If the source mapping changes, downstream mappers silently drift. Bug surfaces only at rename time, or when a new enum value lands in the canonical source but not the duplicates.

**Canonical replacement:** Single canonical mapping, exposed via enum or function. All consumers import.

**Allowlist:** Tests that intentionally test the mapping shape.

**Hook candidate?** Yes — round-trip test pattern. **Extend to every enum** that has display labels.

### Step 3 — Consolidate findings + triage

Each agent reports findings as structured markdown. Consolidate into a single table sorted by severity:

```
| File | Line | Category | Severity | Current | Canonical replacement | Hook candidate |
|------|------|----------|----------|---------|----------------------|----------------|
```

Triage rules:

- **HIGH** = drift currently affecting customer-visible output (e.g., wrong data shipping in emails), OR imminent silent failure (year-boundary expiration), OR mappings drifting in production code paths.
- **MEDIUM** = drift risk that hasn't bitten yet (e.g., model IDs that match config but bypass import; emails scattered without central registry).
- **LOW** = style debt, duplicated list in test fixtures, magic numbers that aren't business-critical.

### Step 4 — File issues (optional)

If your team uses an issue tracker, file findings as issues at this point. Pattern:

1. File umbrella issue: title `Audit: hardcoded values + duplicated mappings across <project>`, priority High. Description includes the consolidated finding table grouped by category.
2. For each HIGH finding, file a sub-issue with: title naming the file + category, body explaining the bug + fix + estimate.
3. MEDIUM findings filed as sub-issues with priority Medium.
4. LOW findings catalogued in umbrella body, no individual issues.

If you don't use an issue tracker, write the report to `docs/audits/drift-YYYY-MM-DD.md` instead.

### Step 5 — Propose new hook guardrails (only if `--propose-hooks`)

For each mechanizable HIGH/MEDIUM finding, draft the test that would catch it at PR time. Write the proposed test code as a markdown code block in the umbrella issue's "Hook expansion" section.

Examples of guardrails to propose:
- `test_no_hardcoded_emails` — grep for `@<your-domain>` literal outside the config module + HTML templates
- `test_no_hardcoded_stripe_ids` — grep for `price_[a-zA-Z0-9]+` outside the config module
- `test_no_hardcoded_supabase_ref` — grep for the literal project ref outside env-var reads + migration files
- `test_no_hardcoded_urls` — grep for `https?://(www\.)?<your-domain>` outside the config module (HTML templates are OK)
- `test_enum_label_round_trip` — round-trip test pattern covering all enums with display labels

User decides which to add. After adding to CI, the corresponding allowlist entries become temporary scaffolding for the cleanup PR.

### Step 6 — Output the audit report

Final output to user, summarized:

```
## Drift Audit — <project> — <date>

### Headline finding
<the single most consequential drift, if any>

### By category
| Category | HIGH | MEDIUM | LOW |
|----------|------|--------|-----|
| Hand-typed reference data | N | N | N |
| Year-bound | N | N | N |
| Model IDs | N | N | N |
| ...

### Filed issues
- <umbrella URL> + N sub-issues (HIGH) + N sub-issues (MEDIUM)
- N LOW findings catalogued in umbrella

### Recommended new hook guardrails
1. test_<name> — would catch <pattern> — covers N findings
2. ...

### Estimated cleanup work
~N hours across N PRs, in priority order
```

## What guards quality (skill-internal)

The skill produces actionable findings, not just a list. Each HIGH finding should include:

1. **Concrete reproduction** — the exact file:line + snippet
2. **Why it's drift** — what canonical source it should reference
3. **Customer impact** — is the drift currently affecting users (HIGH) or just risk (MEDIUM)
4. **Fix shape** — what the PR would look like in 1–2 sentences
5. **Hook candidacy** — can this be mechanized for the gauntlet?

If the skill can't fill all 5 for a finding, demote it to MEDIUM/LOW or drop it. False positives erode trust in the audit.

## Anti-patterns / what NOT to flag

The skill avoids these classes of false positive:

1. **Configuration that IS the canonical source.** Your config module is allowed to hardcode model IDs / prices / URLs / email addresses because that's literally its job. Don't flag canonical sources.

2. **Static HTML meta tags.** `<meta property="og:url" content="https://www.<your-domain>">` needs the production URL by spec. Flag separately as "consider templating" but don't include in HIGH.

3. **Test fixtures.** `tests/` files intentionally hardcode values for test cases. Skip entirely.

4. **Math constants + truly local variables.** `pi = 3.14159`, loop bounds in functions, etc. — these are constants, not drift candidates.

5. **One-off UI copy.** "Welcome back" on the signin page doesn't need to live in a constants file. Flag only when the same string appears in 3+ places (duplication signal) or violates brand voice.

## Companion: keystroke-time hook

A periodic skill-based audit is the cleanup-and-learning layer. A pre-commit / CI test that greps for known drift patterns is the always-on enforcement layer. Skill complements it by:

1. Finding patterns the hook doesn't know about yet → recommending new guardrails (Step 5)
2. Surfacing judgment-call drift that needs LLM evaluation (categories 5, 6, 7, 8 above)
3. Periodic full-codebase sweep vs the hook's per-edit scope

Together: hook catches at keystroke time, skill finds new shapes monthly. Both write to the same canonical mental model.

## Skill Run Logging

If you keep a skill-run log, append an entry after each run:

Format: `| YYYY-MM-DD | /drift | <project> (<args>) | <one-line headline finding> + N HIGH / N MED / N LOW |`
