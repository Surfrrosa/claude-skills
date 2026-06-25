---
name: drift
description: Hardcoded-values audit. Finds every place code drifts from canonical sources of truth — hand-typed astronomical data that should come from ephemeris, model IDs that bypass config.py, email/URL/price literals that duplicate canonical constants, magic numbers in business logic, lists that duplicate enums, year-bound constants that silently expire, embedded copy that drifts from BRAND_VOICE.md. Companion to the keystroke-time gauntlet (`tests/test_no_hardcoded_drift.py`) — the hook catches mechanizable patterns; this skill discovers new patterns + judgment-call drift the hook can't reach. Outputs findings + recommends new hook guardrails. Use as a periodic full-codebase sweep (monthly, or before major launches), or when a customer-visible drift bug surfaces.
---

# Drift Audit

Find every place the code drifts from a canonical source of truth. Catch the kind of mistakes that ship customer-visible bugs: hand-typed reference data that diverges from the source library, paid features running on stale model IDs because the call site never got updated when the central config did, URL literals that break across environments.

This is **not** a code-quality audit. It's specifically drift detection. Different from:
- `/zombie` — dead code / unused things
- `/simplify` — reuse / quality on changed code
- `/cohesion` — visual + structural design drift
- `/walkthrough` — UX coverage gaps
- `/a11y` — accessibility

## Why this exists

The audit catches drift before it ships to users: hand-typed reference data that diverges from the canonical computed value, model IDs that bypass central config, email or URL literals duplicating constants. Run as a periodic full-codebase sweep, or any time a user-visible drift bug surfaces.

Mechanical drift hooks (`tests/test_no_hardcoded_drift.py`) catch known patterns at keystroke time. This skill finds NEW patterns the hook doesn't know about yet, AND patterns that need LLM judgment to evaluate.

## Arguments

- No args: audit current working directory (must be a project root with `CLAUDE.md`)
- A project alias (e.g., `myapp`, `mobile-app`) — see registry below
- `--category=<name>`: limit to one category (astro, models, urls, emails, prices, lists, magic, copy)
- `--linear`: file findings as Linear issues under an umbrella (synestrology + slabcheck only)
- `--propose-hooks`: include "add this to the gauntlet" recommendations for mechanizable findings

## Project Registry

| Alias | Path | Canonical sources |
|---|---|---|
| `myapp` | (add your projects here) | (add your projects here) |

For non-registered projects, attempt to derive canonical sources from `CLAUDE.md`'s "Before Writing" section. If absent, flag and stop.

## Process

### Step 0 — Identify canonical sources

Read in order:
1. `CLAUDE.md` — especially the "Before Writing" section (lists which helpers/constants are canonical)
2. `config.py` or equivalent (constants, env vars, API keys, model IDs, pricing)
3. Calculation modules / enums / shared data (ephemeris, type enums)
4. Voice / brand contracts (`BRAND_VOICE.md`, `BANNED_PHRASES`)
5. Existing mechanical drift tests (`tests/test_no_hardcoded_drift.py` if present) — these are the patterns the hook already catches; don't duplicate them, find what's missing.

Extract:
- **Canonical constants** (BASE_URL, model IDs, email addresses, prices, Stripe IDs, Supabase refs)
- **Canonical enums** (zodiac signs, HD types, lunation types, subscription states)
- **Canonical data sources** (ephemeris functions for astro data, validator regexes for type names)
- **Voice rules** (banned phrases, em-dash ban, brand-voice register)

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

#### Category 1: Hardcoded astronomical data
**What it looks like:** `"2026-05-16"` adjacent to `moon` / `lunation` / `ingress` / `station` / `retrograde` / `eclipse` / `equinox` / `solstice` / `aspect`. Hand-typed degrees + sign names in tables.

**Why it bites:** Hand-typed astro data has been wrong before. Year-bound by definition. Sign-boundary lunations especially error-prone.

**Canonical replacement:** `TransitEphemeris` from `observatory/ephemeris.py` — `find_lunations`, `find_stations`, `find_ingresses`, `find_aspects_in_range`, `get_positions`. Always.

**Allowlist:** `observatory/2026_reference.json` (computed ground truth), `tests/`, lines tagged `# audit-ok: computed-ground-truth`.

**Hook candidate?** ✅ Yes — already covered by `test_no_hardcoded_astronomical_data`.

#### Category 2: Year-bound module constants
**What it looks like:** `LUNATIONS_2026 = [...]`, `EVENTS_2026`, `INGRESSES_YYYY`. Any UPPER_SNAKE_CASE name ending in `_YYYY`.

**Why it bites:** Silently expires at the year boundary. Cron jobs return None forever. Customer calendars go empty.

**Canonical replacement:** Compute at runtime via ephemeris functions. If the data isn't computable, use a year-agnostic name + a generation script that runs on demand.

**Hook candidate?** ✅ Yes — already covered by `test_no_year_bound_constants`.

#### Category 3: Hardcoded model IDs / API client config
**What it looks like:** `"claude-sonnet-4-..."`, `"claude-opus-4-..."`, `"claude-haiku-4-..."`, OpenAI model strings, Stripe price IDs (`price_xxx`), Supabase project refs (`lawjgeayczuidadmwcqn`), etc.

**Why it bites:** When the canonical config bumps the model/price/version, hardcoded callers silently stay on the old one. Past audits found this exactly: paid Reading on Sonnet 4 when config said Sonnet 4.6.

**Canonical replacement:** Import from `config.py` (`ANTHROPIC_*_MODEL`, `STRIPE_*_PRICE_ID`, `SUPABASE_URL`).

**Allowlist:** `config.py` itself (it IS the source), inline `# audit-ok: model-id-explicit` for intentional version freezes.

**Hook candidate?** ✅ Yes — already covered by `test_no_hardcoded_model_ids`. **Extend to Stripe IDs + Supabase ref** (not yet covered).

#### Category 4: Email addresses / URLs / domain literals
**What it looks like:** `"readings@yourapp.com"`, `"hello@yourapp.com"`, `"https://www.yourapp.com"`, `"@yourapp.com"` suffixes for UIDs.

**Why it bites:** If the domain changes, ANY rebrand, or if dev/staging environments need different URLs, the hardcoded literals break silently. Past audits found 8 sites with hardcoded prod URLs that broke dev environments.

**Canonical replacement:** `config.EMAIL_DOMAIN`, `config.BASE_URL`, `config.SENDER_*`, `config.PUBLIC_CONTACT_EMAIL`.

**Allowlist:** `config.py` itself; static HTML meta tags that legitimately need the production URL (og:url, canonical) — flag but don't fail.

**Hook candidate?** ✅ Yes — not yet in the hook. PROPOSE adding `test_no_hardcoded_emails_or_domain_urls`.

#### Category 5: Hardcoded prices / costs / tier limits / magic numbers
**What it looks like:** `40` (dollar amount), `4000` (cents), `25` ($25/year calendar), `3` (lunation send window), `365` (gift expiry), `5/min` (rate limits), `6000` (max_tokens).

**Why it bites:** Same drift pattern — when one changes, everyone has to find every site. Past audits cataloged magic numbers across lunation/gift/state/synthesis.

**Canonical replacement:** Named constants in `config.py` (e.g., `LUNATION_SEND_WINDOW_DAYS`, `GIFT_CODE_EXPIRY_DAYS`).

**Allowlist:** Local loop bounds, math constants (pi), test fixtures.

**Hook candidate?** ⚠️ Partial — can detect prices via `price_xxx` / `$XX` regex, but business-logic magic numbers need LLM judgment. The skill flags these; the user decides which deserve extraction.

#### Category 6: Hand-typed lists duplicating canonical enums
**What it looks like:** `["Aries", "Taurus", ...]` in any file other than the canonical source. `["Generator", "Projector", "Manifestor", ...]`. Regex alternations of enum names in validators.

**Why it bites:** Same drift shape: if a label is renamed centrally, duplicated lists silently drift. Typo accidents.

**Canonical replacement:** Import the enum, use `[t.value for t in HumanDesignType]` or `ZodiacSign.NAMES` or similar exported list.

**Allowlist:** The canonical source file itself.

**Hook candidate?** ⚠️ Partial — can detect via "this exact list appears more than once in src/" check, but distinguishing intentional fixtures from drift needs context.

#### Category 7: Embedded prompt / template copy that drifts from BRAND_VOICE.md
**What it looks like:** Em dashes (—) or en dashes (–) in Python string literals that look like prompts (triple-quoted, >100 chars, addressing the reader). Banned phrases ("trust the process", "raise your vibration", etc.) embedded in Python strings outside the canonical list.

**Why it bites:** Past audits found em dashes IN THE PROMPT TELLING THE LLM NOT TO USE EM DASHES. Contradicting itself. Prompts that ship to the model influence model output.

**Canonical replacement:** Replace em dashes with colons / commas / period restructuring. Banned phrases just removed.

**Allowlist:** Comments + docstrings (style debt, not customer-visible). Rule definitions that SHOW the banned char as an example (e.g., `"Don't use em dashes (—)"` is fine).

**Hook candidate?** ⚠️ Partial — em-dash regex inside triple-quoted strings is doable; banned-phrase detection in Python strings is reachable. Voice gauntlet (`test_template_voice.py`) covers templates, not Python strings — propose extension.

#### Category 8: Duplicated type→label mappings
**What it looks like:** Two or more files that map `"raw_type"` → `"Display Label"` independently. A common root cause.

**Why it bites:** If the source mapping changes, downstream mappers silently drift. Bug surfaces only at sign-boundary or rename time.

**Canonical replacement:** Single canonical mapping, exposed via enum or function. All consumers import.

**Allowlist:** Tests that intentionally test the mapping shape.

**Hook candidate?** ✅ Yes — round-trip test pattern (already covered for lunation types by `test_lunation_type_label_round_trip`). **Extend to every enum** that has display labels.

### Step 3 — Consolidate findings + triage

Each agent reports findings as structured markdown. Consolidate into a single table sorted by severity:

```
| File | Line | Category | Severity | Current | Canonical replacement | Hook candidate |
|------|------|----------|----------|---------|----------------------|----------------|
```

Triage rules:

- **HIGH** = drift currently affecting customer-visible output (e.g., wrong data shipping in emails), OR imminent silent failure (year-boundary expiration), OR mappings drifting in production code paths. Files Linear issue immediately.
- **MEDIUM** = drift risk that hasn't bitten yet (e.g., model IDs that match config but bypass import; emails scattered without central registry). Files Linear issue, lower priority.
- **LOW** = style debt, duplicated list in test fixtures, magic numbers that aren't business-critical. Catalog in umbrella; don't file individually.

### Step 4 — File Linear issues (synestrology + slabcheck only)

Pattern:

1. File umbrella issue: title `Audit: hardcoded values + duplicated mappings across <project>`, labels `["Architecture", "Improvement"]`, priority High. Description includes the consolidated finding table grouped by category.
2. For each HIGH finding, file a sub-issue (`parentId=<umbrella>`) with: title naming the file + category, body explaining the bug + fix + estimate.
3. MEDIUM findings filed as sub-issues with priority Medium.
4. LOW findings catalogued in umbrella body, no individual issues.

Use the `linear-<project>` MCP server.

### Step 5 — Propose new hook guardrails (only if `--propose-hooks`)

For each mechanizable HIGH/MEDIUM finding, draft the test that would catch it at PR time. Write the proposed test code as a markdown code block in the umbrella issue's "Hook expansion" section.

Examples of guardrails to propose:
- `test_no_hardcoded_emails` — grep for `@yourapp.com` literal outside `config.py` + HTML templates
- `test_no_hardcoded_stripe_ids` — grep for `price_[a-zA-Z0-9]+` outside `config.py`
- `test_no_hardcoded_supabase_ref` — grep for the literal project ref outside env-var reads + migration files
- `test_no_hardcoded_urls_in_python` — grep for `https?://(www\.)?synestrology\.com` outside `config.py` (HTML templates are OK)
- Extended `test_lunation_type_label_round_trip` → generalized `test_enum_label_round_trip` covering all enums with display labels

User decides which to add. After adding to the hook, the corresponding allowlist entries become temporary scaffolding for the cleanup PR.

### Step 6 — Output the audit report

Final output to user, summarized:

```
## Drift Audit — <project> — <date>

### Headline finding
<the single most consequential drift, if any>

### By category
| Category | HIGH | MEDIUM | LOW |
|----------|------|--------|-----|
| Astro data | N | N | N |
| Year-bound | N | N | N |
| Model IDs | N | N | N |
| ...

### Filed Linear issues
- <umbrella URL> + N sub-issues (HIGH) + N sub-issues (MEDIUM)
- N LOW findings catalogued in umbrella

### Recommended new hook guardrails
1. test_<name> — would catch <pattern> — covers N findings
2. ...

### Estimated cleanup work
~N hours across N PRs, in priority order: SYN-X first (customer-trust win), then ...
```

## What guards quality (skill-internal)

The skill produces actionable findings, not just a list. Each HIGH finding should include:

1. **Concrete reproduction** — the exact file:line + snippet
2. **Why it's drift** — what canonical source it should reference
3. **Customer impact** — is the drift currently affecting users (HIGH) or just risk (MEDIUM)
4. **Fix shape** — what the PR would look like in 1–2 sentences
5. **Hook candidacy** — can this be mechanized for the gauntlet?

If the skill can't fill all 5 for a finding, demote it to MEDIUM/LOW or drop it. False positives erode trust in the audit.

## Skill Run Logging

After completing this skill, append a log entry to `~/.claude/skill-log.md` under the Run Log table. Format:

`| YYYY-MM-DD | /drift | <project> (<args>) | <one-line headline finding> + N HIGH / N MED / N LOW issues filed |`

## Anti-patterns / what NOT to flag

The skill avoids three classes of false positive:

1. **Configuration that IS the canonical source.** `config.py` is allowed to hardcode model IDs / prices / URLs / email addresses because that's literally its job. Don't flag canonical sources.

2. **Static HTML meta tags.** `<meta property="og:url" content="https://www.yourapp.com">` needs the production URL by spec. Flag separately as "consider templating" but don't include in HIGH.

3. **Test fixtures.** `tests/` files intentionally hardcode values for test cases. Skip entirely.

4. **Math constants + truly local variables.** `pi = 3.14159`, loop bounds in functions, etc. — these are constants, not drift candidates.

5. **One-off UI copy.** "Welcome back" on the signin page doesn't need to live in a constants file. Flag only when the same string appears in 3+ places (duplication signal) or violates brand voice.

## Companion: keystroke-time hook

The hook (`tests/test_no_hardcoded_drift.py`) is the always-on enforcement layer. Skill complements it by:

1. Finding patterns the hook doesn't know about yet → recommending new guardrails (Step 5)
2. Surfacing judgment-call drift that needs LLM evaluation (categories 5, 6, 7, 8 above)
3. Periodic full-codebase sweep vs the hook's per-edit scope

Together: hook catches at keystroke time, skill finds new shapes monthly. Both write to the same canonical mental model.
