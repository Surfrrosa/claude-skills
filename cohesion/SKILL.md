---
name: cohesion
description: Visual + structural design-cohesion audit. Detects every place a page drifts from the project's design system — different button styles, missing nav/footer, hard-coded colors that should be tokens, parallel CSS classes recreating canonical components, inline `<style>` blocks redefining variables. Cross-page consistency check across the whole site, not just changed code. Use when the site looks visually inconsistent across pages, or as part of a launch-ready audit pass.
---

# Cohesion Audit

Find every place a page drifts from the project's design system. The system has a canonical layer (base templates + base.css + macros + design-system docs); this skill scans everything else against that layer and reports drift, ranked by severity.

This is **not** a code-quality audit. It's specifically about visual + structural consistency. Different from:
- `/zombie` (dead code / duplication)
- `/walkthrough` (UX coverage gaps)
- `/simplify` (reuse / quality on changed code)
- `/a11y` (accessibility)
- `/security-review` (security)

## Arguments

- No args: audit current working directory
- A project name or path (e.g., `myapp`, `mobile-app`)
- `--live`: also fetch live URLs and compare rendered output across pages
- `--fix`: auto-fix safe categories (rare; most fixes need design judgment — default off)

## Project Registry

| Alias | Path | Canonical layer |
|-------|------|-----------------|
| `myapp` | (add your projects here) | (add your projects here) |

**For React Native / Expo projects:** the cohesion model is different (StyleSheet objects, Theme provider, native components). Skip this skill or adapt it heavily — flag and stop.

## Process

### Step 0 — Identify the source of truth

Resolve the project. Read in order:
1. `CLAUDE.md` — any documented design conventions
2. The canonical base template(s) — usually a `base.html` or `App.tsx` or `Layout.astro`
3. The canonical stylesheet — usually `base.css` or `globals.css` or a theme file
4. Any shared components / macros — `_macros/`, `components/ui/`, `src/components/shared/`
5. Design-system docs — `DESIGN_SYSTEM.md`, `BRAND_VOICE.md`, design contract files

From these, **extract the canonical inventory**:

| Token type | What to extract |
|---|---|
| Color tokens | All `--*` CSS variables defined in the canonical stylesheet's `:root` block |
| Font tokens | `--font-display`, `--font-body`, etc. — and the actual font families they map to |
| Component classes | Every class defined in the canonical stylesheet that's meant to be reused: `.btn-primary`, `.btn-secondary`, `.card`, `.form-group`, `.empty-state`, etc. |
| Allowed base templates | Which template names child templates are allowed to extend |
| Allowed `<link>` stylesheets | Which external stylesheets are sanctioned (typically just Google Fonts for the canonical families) |
| Page-shell elements | What every page is expected to have: nav, footer, video-bg, main landmark, lang attr, etc. |

Write this inventory down (mentally or in a scratch buffer); the audit compares against it.

### Step 1 — Page-shape violations (RED)

Highest-severity drift. A page that doesn't extend the base shell shows up looking like a different site entirely.

For every page-level template (the ones a route handler actually renders), check:

#### 1a. Extends an allowed base
- Does it have `{% extends "base.html" %}` or `{% extends "_observatory_base.html" %}` (Jinja) / `<Layout>` wrapper (Astro) / equivalent?
- A page that doesn't extend anything renders without nav/footer/global styles.

#### 1b. Loads canonical stylesheet
- Does it inherit `<link rel="stylesheet" href="/static/base.css">` from the base, OR does it load its own?
- Pages that load their own stylesheet bypass the design system.

#### 1c. Has the canonical wordmark + nav
- If the base has a `<nav class="sticky-nav">`, does the page render with it? (Pages that override layout blocks may skip it.)
- Pages with their own custom `<nav>` implementation = drift.

#### 1d. Has the footer (specifically: includes the partial, not its own copy)
- Is `_footer.html` (or equivalent) included? Or does the page roll its own footer markup?
- **Concrete detection:** for every page-level template, check both:
  - Does the file contain a literal `{% include "_footer.html" %}` (or framework equivalent)? AND
  - Does the file contain a `<footer>` tag NOT inside that include?
- Either condition failing is RED. A page with its own inline `<footer>` is the most common drift in old/orphaned pages — and when the inline copy ages, it falls behind the canonical (missing newer links like Home/Blog/social, browser-default link styling, broken indentation copy-pasted from a stale snapshot). 2026-05-04 lesson: blueprint.html and time-travel.html had their own inline `<footer>` for months; the inline copies were missing Home + Blog and rendered with browser-default blue underlined links. /cohesion's prior pass had Step 1d in spec but didn't grep specifically for the inline pattern — sharpen to mechanical.

#### 1e. Has the video-bg / canonical background
- If the base has `.video-bg` markup, does the page rely on it or override it?
- See also Step 1.5 — overriding `.video-bg` rules in page CSS is the load-bearing failure mode here.

#### 1f. Has the canonical viewport + lang
- `<html lang="en">`, viewport meta — usually inherited but worth checking pages with custom `<head>`.

**Red flag heuristic:** any page that defines its own `<html>`, `<head>`, `<body>`, or `<footer>` instead of inheriting from base is RED.

### Step 1.5 — Global selector overrides in page CSS (RED)

The class of bug yesterday's audit missed: a page-specific stylesheet (or inline `<style>` block) that **redefines selectors which already exist in the canonical stylesheet**. The page extends `base.html` correctly, includes `_footer.html` correctly, but its own CSS silently overrides global rules — producing visible drift like wrong z-index on the background, wrong link colors in the footer, layout glitches in the nav.

**The specific bug shape:**
- `base.css` defines `.video-bg { z-index: -1 }` (behind everything)
- `tools/blueprint.css` redefines `.video-bg { z-index: 0 }` (above default-z elements)
- Browser cascade: page CSS loads after base.css, so page rule wins
- Result: the global background gradient renders ON TOP of UI elements that don't have explicit z-index, breaking the layered visual stack

**Detection (mechanical):**

For every project-level `.css` file (NOT the canonical one):
1. Extract every selector that has `display:`, `position:`, `z-index:`, or `font-family:` declarations
2. Cross-reference with the canonical stylesheet
3. **Any overlap is RED** — even if the override "looks the same," the duplication means future canonical changes don't propagate

For every inline `<style>` block in a template:
4. Run the same check on its rules
5. RED if any global selectors are touched

**Forbidden override list (synestrology-specific; adapt per project):**
- `body`, `html`
- `.video-bg`, `.video-bg video`, `.video-bg::after`
- `.sticky-nav`, `.nav-*`, `.nav-dropdown*`, `.nav-hamburger*`
- `footer` (the bare element selector — use `.page-specific-footer-class` if you need page-scoped styling)
- `.footer-links`, `.footer-social`, `.footer-copy`
- `#cookieBanner` and any cookie-banner classes
- `.skip-link`
- `main`, `#main-content`

A grep one-liner for synestrology:

```bash
for css in src/api/static/*.css src/api/templates/**/*.html; do
  rg -n '^\s*(body|html|footer|main|\.video-bg|\.sticky-nav|\.nav-|\.footer-|\.skip-link|#cookieBanner)\s*[:{,]' "$css"
done | grep -v 'src/api/static/base.css'
```

Each hit outside `base.css` is a RED finding. The fix is one of:
- Delete the override (let canonical drive — most common)
- Rename the selector to something page-scoped (e.g., `.blueprint-video-overlay` instead of `.video-bg`)
- Move the rule into `base.css` if it should apply globally

### Step 1.6 — HTML well-formedness in templates (RED, mechanical)

The class of bug 2026-05-04 evening surfaced: `tools/blueprint.html` had an unclosed `<script type="application/ld+json">` for the FAQ schema. The browser's HTML parser, while inside a `<script>` element, only exits on `</script>` — content type doesn't matter. So the parser stayed in script-content mode through ALL subsequent `<head>` content (favicons, fonts, base.css link, PostHog `<script>` — all swallowed as script content) until the next `</script>` it could find. Net effect: base.css never loaded. The page rendered with no nav styling, no body background, browser-default link colors. Has been broken in production since the original Jinja migration commit (`ce797cd`); was masked for months by the page's own inline `<style>` block compensating for the missing base.css.

This is the simplest-possible static check and audit skills should not miss it again.

**Detection (mechanical, per template):**

```bash
# Per page-level template, count opens vs closes for tags that MUST balance.
# A mismatch is a RED finding — the page is broken HTML.
for tpl in src/api/templates/**/*.html; do
  for tag in script style; do
    opens=$(grep -cE "<$tag[[:space:]>]" "$tpl")
    closes=$(grep -c "</$tag>" "$tpl")
    if [ "$opens" != "$closes" ]; then
      echo "RED  $tpl: <$tag> $opens opens, $closes closes (mismatch)"
    fi
  done
done
```

Other tags worth balancing on page-level templates: `<head>`, `<body>`, `<html>`, `<main>`, `<form>`, `<table>`, `<div>` (last one is noisy — only flag mismatches > 5).

**Common false-positive shape to ignore:**
- Self-closing or void elements (`<meta>`, `<link>`, `<img>`, `<input>`, `<br>`, `<hr>`) don't need closes.
- `<script>` with `src=""` is self-contained but still needs `</script>` per HTML spec.
- Jinja `{% block %}` boundaries inside a `<script>` tag — fine, the rendered output collapses correctly UNLESS the Jinja block itself never closes the open `<script>` (which is exactly what bit blueprint).

**Auto-fix eligibility:** balance violations are NOT generally auto-fixable — the missing close could go in many places, and inserting it wrong creates a worse bug. Always file as Linear with the exact line number of the unclosed open. Human inspects + adds the closing tag.

The exception: if the unclosed tag is OBVIOUSLY at the end of a `{% block %}` and the missing close is one line, an auto-fix is safe. Tonight's blueprint case fit this — the FAQ JSON-LD just needed `}]}` + `</script>` at the end of the `{% block meta %}`.

### Step 1.7 — Shared UI-component class drift (RED)

The class of bug 2026-05-08 surfaced — the most insidious drift category yet:

- `.form-section-title` was defined in `base.css` with `color: var(--secondary)` (copper)
- The SAME class was redefined in 4 page-level files (blueprint.css, time-travel.css, pages/index.html, pages/gift.html), all with the same wrong color
- A user noticed the visible drift across forms: section headers reading as a different shade than field labels
- Fix required updating all 5 definitions in lockstep AND patching yearly.html + gift-redeem.html that USED the class but didn't define it (relying on browser defaults)

This is **parallel-definition drift**, distinct from Step 3's component drift:
- Step 3 catches: page invents `.checkout-btn` instead of using canonical `.btn-primary` (USAGE drift)
- Step 1.7 catches: page **redefines `.btn-primary` itself** with different properties (DEFINITION drift)

Definition drift is worse because each copy is a subtle variation of the canonical; updating the canonical doesn't propagate. The user perceives it as "100 different shades of white" or "buttons that almost-but-don't match." Standard cohesion checks miss it because each individual rule looks valid in isolation.

**Canonical UI-component class list (synestrology — adapt per project):**

- Form scaffolding: `.form-section-title`, `.form-section`, `.form-group`, `.field-label`, `label.field-label`, `.field-error`, `.field-help`
- Buttons: `.btn`, `.btn-primary`, `.btn-secondary`, `.btn-text` (extend with future variants like `.btn-sm`, `.btn-ghost`)
- Cards: `.card`, `.card-*`
- States: `.empty-state`, `.error-state`
- Layout: `.skip-link`, `.video-bg`, `.sticky-nav`, `.nav-*`

For each class, run **three mechanical checks**:

**1.7a — Duplicate definitions (RED)**

```bash
# For each canonical class, count definitions across the project:
for cls in form-section-title field-label btn-primary btn-secondary card empty-state; do
  hits=$(rg -ln "^\\s*\\.${cls}\\s*\\{" src/api/static/ src/api/templates/)
  count=$(echo "$hits" | grep -c .)
  if [ "$count" -gt 1 ]; then
    echo "RED  .${cls} defined in $count files:"
    echo "$hits" | sed 's/^/  - /'
  fi
done
```

A canonical class should be defined ONCE, in `base.css` (or your project's canonical stylesheet). Page-level redefinitions are RED, even if the values look identical — the duplication itself is the bug.

**1.7b — Property-value mismatch across definitions (RED)**

When a class IS defined in multiple places, compare property values:

```bash
# Pull every definition + its property block, diff against the canonical:
rg -A 15 "^\\s*\\.form-section-title\\s*\\{" src/api/static/ src/api/templates/
# → eyeball: do `color`, `font-family`, `font-size`, `padding`, `letter-spacing`
#   match across all hits? Mismatches are RED.
```

Especially flag: `color`, `font-family`, `font-size`, `font-weight`, `padding`, `letter-spacing`, `border`. These produce the most visible cross-page drift.

**1.7c — Used-but-undefined (RED)**

A class used in markup with NO matching CSS rule reachable on the page renders with browser defaults — silently broken until someone notices.

```bash
# For each canonical class, find templates that USE it:
for cls in form-section-title field-label btn-primary card; do
  rg -l "class=\"[^\"]*\\b${cls}\\b" src/api/templates/ | while read tpl; do
    # Check if the template loads a stylesheet that defines the class.
    # Simplest heuristic: does base.css load on this page (extends base.html)
    # AND does base.css define the class?
    base_loads=$(grep -l "extends.*base.html\|/static/base.css" "$tpl")
    base_defines=$(grep -c "^\\s*\\.${cls}\\s*\\{" src/api/static/base.css)
    if [ -z "$base_loads" ] && [ "$base_defines" -gt 0 ]; then
      echo "RED  $tpl uses .${cls} but doesn't load base.css"
    fi
  done
done
```

Real example caught: `pages/yearly.html` and `pages/gift-redeem.html` used `.form-section-title` but had no rule for it — relied on browser defaults.

**Auto-fix bundle (per Step 7 pattern):**
- **1.7a duplicates** with verbatim copy → bundle PR that deletes the duplicates, leaves only the canonical in base.css
- **1.7b property mismatch** on a single property (e.g., just `color`) → bundle PR that flips all definitions to the canonical value
- **1.7b property mismatch** on multiple properties → file Linear sub-issue under an umbrella issue, needs design judgment
- **1.7c orphan usage** → file Linear; either add to base.css or stop using

### Step 2 — Token discipline (YELLOW)

Find every place that hard-codes a value that should be a token. These look fine in isolation but produce subtle drift over time.

#### 2a. Hex color codes outside the canonical stylesheet
- Grep for `#[0-9a-fA-F]{3,6}` across all templates + static files.
- Filter out the canonical stylesheet itself + any acceptable surfaces (e.g., favicon manifest, social-card images).
- Every remaining hit is a candidate for tokenization. Cross-reference: does the hex match an existing token? If yes, it's a swap-in fix. If no, it's a new color the design system should know about.

#### 2b. `font-family:` declarations outside the canonical stylesheet
- Already partially covered by `test_no_template_declares_font_face`, but this catches inline `style="font-family: ..."` and page-specific `<style>` blocks redefining font-family.

#### 2c. Inline style attributes that redefine tokens
- `style="color: #B87333"` should be `style="color: var(--accent)"` (or better, a class)
- `style="background: rgba(26, 20, 24, 0.6)"` should be a token

#### 2d. External font imports beyond the canonical set
- Already covered by `test_no_template_imports_external_fonts`, but recheck — drift here is silent and brand-breaking.

### Step 3 — Component drift (YELLOW)

Find every place that recreates a canonical component instead of using the macro/class.

#### 3a. Button-like elements not using `.btn-*` classes
- Every `<button class="...">` and `<a class="...">` whose visual is button-like
- Group by class. If the same visual exists under 5 different class names, that's drift to consolidate.
- Especially flag: page-scoped classes like `.checkout-btn`, `.sample-report-link`, `.primary-action` that recreate `.btn-primary` styling.

#### 3b. Card-like `<div>` clusters that should be the `.card` macro
- Look for `<div class="*-card *-panel *-box">` patterns with manual border + padding + border-radius
- These should use `_macros/cards.html::card` or a future shared card primitive.

#### 3c. Form inputs that bypass `_macros/forms.html`
- Any `<div class="form-group"><label><input></div>` markup that exactly mirrors what `_macros/forms.html::text_input` renders should be migrated to the macro.
- Saves ~5 lines per input + ensures error-slot + autocomplete consistency.

#### 3d. Status/empty/error states bypassing `_macros/states.html`
- Custom "no results" markup, custom error banners — fold into the macro.

### Step 4 — Inline `<style>` block size (smell heuristic)

For each page template:
- Count lines inside `<style>` blocks (excluding comments and whitespace)
- A page with > ~50 lines of inline CSS is probably drifting from the system
- Rank pages by inline-CSS size; the worst offenders get top billing in the report

This isn't a violation per se — sometimes page-specific styling is correct. But it's a smell. Worth eyeballing the largest blocks for token drift, parallel components, or stuff that should be in `base.css`.

### Step 5 — Cross-page rendering compare (`--live` flag)

When `--live` is passed, fetch every public route via `WebFetch` and parse the rendered HTML.

For each page, check the rendered output (not source) for:
- `<main>` landmark present
- `<nav class="sticky-nav">` present
- `<footer>` present
- `.video-bg` element present
- Canonical font preconnects + Google Fonts link present
- No additional `<link rel="stylesheet">` beyond the canonical set
- `<html lang="...">` present

Build a comparison matrix:

| Page | nav | footer | video-bg | base.css | extra stylesheets |
|---|---|---|---|---|---|
| `/` | ✓ | ✓ | ✓ | ✓ | (none) |
| `/blog` | ✓ | ✓ | ✓ | ✓ | (none) |
| `/tools/cosmic-blueprint` | ✓ | ✗ (custom) | ✓ | ✓ | own.css |
| ... | | | | | |

Pages with any ✗ are RED. Inconsistencies across the matrix are the visible drift.

### Step 6 — Ranking

Compose findings into a single prioritized report:

```
## Cohesion Audit — [project]

### 🔴 RED — Page-shape violations
1. **[file]** — extends `base.html` but overrides `{% block content %}` with full custom layout including its own nav and footer. Drift visible to users.
2. **[file]** — does not extend any base; renders with bespoke `<head>`/`<body>`. Carries its own @font-face block + footer.

### 🟡 YELLOW — Component drift
1. `.checkout-btn` recreates `.btn-primary` (used in [N] places). Migrate or alias.
2. `.sample-report-link` recreates `.btn-text`. Migrate.
3. [N] inline `style="color: #B87333"` should be `var(--accent)`.

### 🟡 YELLOW — Token discipline
1. [N] hex colors outside `base.css` ([list top 5 with file:line]).
2. [N] `font-family:` declarations outside `base.css`.

### 🟠 SMELL — Inline `<style>` block size
1. `tools/blueprint.html` — [N] lines of inline CSS (largest)
2. `tools/time-travel.html` — [N] lines
3. `pages/index.html` — [N] lines (may be acceptable; landing pages tend to have hero-specific styling)

### Summary
- Top priority: [the worst offender]
- Next: [the next 2-3 priorities]
- Estimated cleanup effort: [rough size]
```

Each RED finding gets a Linear issue (or one umbrella issue with sub-findings) per project conventions.

## Fix Mode (`--fix` flag) and Step 7 auto-fix bundle

When `--fix` is set (or by default for projects in the auto-fix-eligible list — synestrology, slabcheck, surfrrosa, portfolio), bundle the auto-fixable findings into a single PR following the same pattern documented in `~/.claude/skills/walkthrough/SKILL.md` Step 6 (auto-fix bundle workflow). Pass `--report-only` to suppress.

**Safe to auto-fix (`auto_fix: true`):**
- Replace inline `<footer>` block with `{% include "_footer.html" %}` (or framework equivalent) when the inline copy is a stale partial of the canonical footer — the most common drift category for old/orphaned pages, mechanical fix
- Delete page-CSS rules that override global selectors (`body`, `html`, `.video-bg*`, `.sticky-nav*`, `.nav-*`, `footer`, `.footer-*`, `.skip-link`, `#cookieBanner*`, `main`, `#main-content`) when the override doesn't add new behavior. If the override DOES change behavior, downgrade to `auto_fix: false` (needs design review).
- Replace inline `style="color: #B87333"` with `style="color: var(--accent)"` when the hex unambiguously matches a known token
- Add `rel="stylesheet" href="/static/base.css"` to a page that doesn't load it (only if the page already extends a base — bare-`<head>` rewrites are NOT auto-fixable)
- Remove dead CSS rules (e.g., `.cookie-banner` rules in a project where the canonical cookie banner uses `#cookieBanner` id)

**Requires confirmation (`auto_fix: false`):**
- Migrating a page from custom layout to `extends "base.html"` (changes routing assumptions, may break tests)
- Replacing a parallel button class with `.btn-primary` (may need CSS specificity adjustments + visual review)
- Removing an inline `<style>` block in favor of base.css equivalents (extraction first, then cleanup as separate PR — see two-stage extraction-then-cleanup pattern)
- Renaming a global-selector override to a page-scoped class (touches markup + CSS together)

**Never auto-fix:**
- Hex colors that don't match a known token (could be intentional brand exception or a new token to add)
- Font-family changes (could break ascenders/descenders/em sizing)
- Removing custom layouts (usually represents real design judgment that needs review)
- Page-CSS that overrides a global selector if the override is non-trivial (e.g., adds keyframes, redefines a media query block) — flag and file as Linear, don't touch

**Workflow:** identical to /walkthrough Step 6 — partition findings by `auto_fix`, build branch + commit + push + open PR with the auto-fixable set, file Linear issues for the rest. Cap at 5 fixes per PR; if more, split by category (footer-includes, override-deletions, hex-substitutions, etc.).

## Rules

- **Source of truth is the canonical layer.** Don't invent a "what should be" from thin air; read base.html, base.css, macros, and design docs to compute the inventory FIRST. Then audit against THAT.
- **Severity by impact.** A page without the global nav is RED — users see a different site. A hex color that should be a token is YELLOW — invisible drift today, brand-breaking when the brand color changes tomorrow.
- **Per-token-substitution fixes are easy; layout migrations are hard.** Default scope is detection + reporting. Layout migrations need a separate ticket per page.
- **Watch for grandfathered violations.** Existing cohesion lints (e.g., `tests/test_template_cohesion.py` in synestrology) often have an explicit grandfathered list. Read it; don't re-flag what's already known. Do flag if items have been on the list too long.
- **Be specific.** "tools/blueprint.html drifts from the design system" is useless. "tools/blueprint.html declares its own @font-face for Cormorant Garamond at line 47, has a 480-line inline `<style>` block, uses `.checkout-btn` instead of `.btn-primary`, and includes a custom `<footer>` instead of `_footer.html`" is useful.

## Skill Run Logging

After completing this skill, append a log entry to `~/.claude/skill-log.md` under the Run Log table. New entries at the top (most recent first).

Format: `| YYYY-MM-DD | /skill-name | target (details) | one-line result |`
