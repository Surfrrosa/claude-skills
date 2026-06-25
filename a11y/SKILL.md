---
name: a11y
description: Audit accessibility (WCAG) compliance in source code and optionally against live URLs
---

# Accessibility Audit

Scan a project's source code for WCAG 2.1 accessibility violations. Optionally check live URLs for rendered accessibility issues. Can auto-fix safe categories.

## Sample output

A successful run produces something like:

```
## Accessibility Audit — marketing-site
Score: 76/100 (C)
Stack: Next.js 16

### CRITICAL (blocks entire user groups)
1. Keyboard accessibility — src/components/NavMenu.tsx:42 — onClick handler on <div> with no tabIndex or onKeyDown. Keyboard users can't open the menu. Fix: change to <button>.

### SERIOUS
1. Forms — src/app/contact/page.tsx:18 — <input name="email"> has placeholder but no <label>. Fix: wrap in <label> or add aria-label.
2. Images — src/components/Hero.tsx:24 — <img src="/hero.jpg"> missing alt attribute.

### MODERATE
1. Heading hierarchy — src/app/about/page.tsx — page jumps from h1 to h3, skipping h2.

### Summary
- 1 critical, 2 serious, 1 moderate
- Top priority: NavMenu keyboard accessibility (blocks every keyboard user)
```

## Arguments

The user may specify:
- No args: audit current working directory
- A project name or path (e.g., `myapp`, `marketing-site`)
- `--fix`: auto-fix safe categories (missing alt placeholders, missing ARIA labels on obvious elements, lang attribute, etc.)
- `--live`: also fetch live URLs and audit the rendered HTML
- `--scope <path>`: limit to a specific directory or file

Parse the argument string. Examples:
- `/a11y` -- audit current project, source only
- `/a11y myapp` -- audit myapp source
- `/a11y myapp --live` -- audit myapp source + live URL
- `/a11y --fix` -- audit and auto-fix current project
- `/a11y --scope src/components` -- audit only components directory

## Project Registry

| Alias | Path | Live URL |
|-------|------|----------|
| `myapp` | (add your projects here) | (add your projects here) |

## Process

### Step 0: Detect the project

1. Resolve path from alias table or cwd
2. Detect the stack (Next.js, Astro, Flask/Jinja, static HTML, React Native)
3. Identify template/component directories where HTML is generated:
   - **Next.js:** `app/`, `src/app/`, `src/components/`, `components/`
   - **Astro:** `src/pages/`, `src/layouts/`, `src/components/`
   - **Flask/Jinja:** `src/templates/`, `templates/`, `src/**/templates/`
   - **Static HTML:** all `.html` files in root and subdirectories
   - **React Native / Expo:** skip this audit entirely -- native apps use a different accessibility model (accessibilityLabel, accessibilityRole, etc.). Note this and suggest a native-specific check instead.
4. Read `CLAUDE.md` for any project-specific accessibility notes or conventions

### Step 1: Structural / Semantic HTML

#### 1a. Landmark regions
Search layouts and page templates for:
- Does the page have a `<main>` element? (exactly one per page)
- Does the page have a `<nav>` element for primary navigation?
- If there are multiple `<nav>` elements, does each have an `aria-label` or `aria-labelledby` to distinguish them?
- Is there a `<header>` and `<footer>` at the page level?
- Are `<section>` elements used with headings or `aria-label` (a `<section>` without an accessible name is meaningless to screen readers)?

**Flag:** Missing `<main>`, navigation without labels when multiple exist, `<section>` without accessible name.

#### 1b. Heading hierarchy
For each page/template:
- Is there exactly one `<h1>`?
- Do headings follow a logical order (no skipping from h1 to h3)?
- Are headings used for structure, not just styling? (A `<h3>` used because it "looks right" instead of using CSS is a smell)

**Flag:** Missing h1, skipped heading levels, multiple h1s on a single page.

#### 1c. Lists
- Are groups of related links wrapped in `<ul>` or `<ol>`, not just stacked `<a>` tags with `<br>` between them?
- Are navigation menus using list markup?

**Flag:** Navigation links not in list elements.

### Step 2: Interactive Elements

#### 2a. Keyboard accessibility
Search all components/templates for interactive elements:
- **Click handlers on non-interactive elements:** `onClick` / `@click` / `on:click` on `<div>`, `<span>`, `<td>`, `<img>`, or other non-interactive elements without `role` and `tabIndex`. These are invisible to keyboard users.
- **Missing keyboard equivalents:** `onClick` without corresponding `onKeyDown` / `onKeyUp` handler on non-button/non-link elements.
- **Positive tabindex:** `tabindex` values greater than 0 break natural tab order. `tabindex="0"` (add to tab order) and `tabindex="-1"` (programmatic focus only) are fine.
- **Autofocus abuse:** `autoFocus` on elements that aren't the primary action on the page (disorienting for screen reader users).

**Flag:** Click handlers on divs/spans without role+tabIndex, positive tabindex values.

#### 2b. Buttons and links
- `<a>` tags without `href` (or with `href="#"` or `href="javascript:void(0)"`) -- these should be `<button>` elements.
- `<button>` elements without visible text or `aria-label`. Icon-only buttons need labels.
- Links that open in new tabs (`target="_blank"`) without indicating this to the user (add `aria-label` suffix or visible indicator). Also check for `rel="noopener"`.
- Empty links: `<a href="..."></a>` with no text content or aria-label.

**Flag:** Fake links (href="#"), unlabeled icon buttons, empty links, new-tab links without indication.

#### 2c. Forms
Search for `<form>`, `<input>`, `<select>`, `<textarea>`:
- Every input must have an associated `<label>` (via `for`/`htmlFor` attribute matching `id`, or wrapping the input). `placeholder` is NOT a substitute for a label.
- Required fields should have `aria-required="true"` or the `required` attribute.
- Error messages should be associated with their input via `aria-describedby`.
- Form groups (e.g., radio buttons) should use `<fieldset>` + `<legend>`.
- Submit buttons should have clear text (not just "Submit" -- what are you submitting?).

**Flag:** Inputs without labels, placeholder-only inputs, unassociated error messages.

### Step 3: Images and Media

#### 3a. Images
- Every `<img>` must have an `alt` attribute.
- Decorative images should have `alt=""` (empty alt, not missing alt).
- Informative images should have descriptive alt text, not filenames (`alt="IMG_2847.jpg"`) or generic text (`alt="image"`, `alt="photo"`, `alt="icon"`).
- CSS background images that convey information need an alternative (text or aria-label on the container).
- SVG icons used inline: should have `aria-hidden="true"` if decorative, or `role="img"` + `aria-label` if meaningful.

**Flag:** Missing alt, filename alt text, generic alt text, meaningful SVGs without labels.

#### 3b. Media
If the project has `<video>` or `<audio>` elements:
- Videos should have captions or a transcript
- Audio should have a transcript
- Media players should be keyboard-operable

**Flag:** Video without captions/transcript, audio without transcript.

### Step 4: Color and Visual

#### 4a. Color contrast
This is hard to check from source alone, but look for:
- Inline styles or CSS with text color + background color combinations that can be evaluated. Common failures:
  - Light gray text on white (`#999 on #fff` = 2.8:1, needs 4.5:1)
  - White text on bright colors (yellow, light green, light blue backgrounds)
- Tailwind classes: check for low-contrast combinations (e.g., `text-gray-400 bg-white`, `text-white bg-yellow-300`)
- CSS custom properties: if the project defines color tokens/variables, check them against WCAG AA minimums:
  - Normal text: 4.5:1 ratio minimum
  - Large text (18px+ bold or 24px+): 3:1 ratio minimum
  - UI components and graphical objects: 3:1 ratio minimum

**Flag:** Detectable low-contrast combinations. Note that full contrast audit requires `--live` mode.

#### 4b. Color as sole indicator
- Links distinguished from body text only by color (no underline, no other visual indicator). Links within body text must have a non-color indicator.
- Form errors indicated only by turning the border red. Must also have text or icon.
- Status indicators (success/error/warning) using only color. Must also have text or icon.

**Flag:** Color-only link differentiation in body text, color-only error states.

#### 4c. Motion and animation
- Check for CSS animations or transitions without `prefers-reduced-motion` media query support.
- Flag any `animation` property used without a corresponding `@media (prefers-reduced-motion: reduce)` rule somewhere in the project's CSS.
- Autoplay video or animated content without pause controls.

**Flag:** Animations without reduced-motion support, autoplay without controls.

### Step 5: ARIA Usage

#### 5a. ARIA done right
- `aria-hidden="true"` on elements that contain focusable children (traps keyboard users in invisible elements).
- `role` attribute with an invalid value.
- `aria-labelledby` or `aria-describedby` pointing to an `id` that doesn't exist in the same page.
- Redundant ARIA: `role="button"` on a `<button>`, `role="link"` on an `<a>`, `role="navigation"` on a `<nav>`. These are already implicit.
- `aria-label` on elements that don't support it (e.g., `<div>` without a role, `<span>`).

**Flag:** Broken ARIA references, ARIA on focusable hidden elements, redundant roles.

#### 5b. Dynamic content
- Modals/dialogs: should have `role="dialog"`, `aria-modal="true"`, focus trap, and return focus on close.
- Toast/notification components: should use `role="alert"` or `aria-live="polite"`.
- Loading states: should use `aria-busy="true"` on the container, or a live region announcing the state change.
- Content that updates without page reload: should use `aria-live` regions.

**Flag:** Modals without dialog role/focus management, dynamic content without live regions.

### Step 6: Document-Level

- `<html>` has `lang` attribute (e.g., `lang="en"`)
- Page `<title>` is unique and descriptive per page (not just the site name everywhere)
- Skip-to-content link exists (a visually hidden link at the top of the page that jumps to `<main>`)
- Viewport meta doesn't disable zoom: `user-scalable=no` or `maximum-scale=1` are violations

**Flag:** Missing lang, missing/duplicate titles, missing skip link, zoom disabled.

## Live Check (--live flag)

When `--live` is passed, use `WebFetch` to check each page's rendered HTML for:

1. **Rendered heading hierarchy** -- does the final HTML have correct h1-h6 order?
2. **Rendered landmarks** -- are `<main>`, `<nav>`, `<header>`, `<footer>` present in output?
3. **Rendered form labels** -- do inputs have associated labels in the final DOM?
4. **Contrast estimation** -- parse inline styles and class-based colors in the rendered output
5. **ARIA reference integrity** -- do all `aria-labelledby`/`aria-describedby` IDs resolve?

Compare live results against source findings. Flag discrepancies (SSR issues, hydration bugs that break accessibility).

## Scoring

**Max score: 100 points.** Weighted by impact:

| Category | Weight | Rationale |
|----------|--------|-----------|
| Keyboard accessibility (2a) | 20 | Complete blocker for keyboard/switch users |
| Forms (2c) | 15 | Blocks task completion |
| Images and alt text (3a) | 15 | Core screen reader experience |
| Heading hierarchy (1b) | 10 | Navigation and comprehension |
| Landmarks (1a) | 10 | Screen reader navigation |
| Buttons and links (2b) | 10 | Core interaction |
| Document-level (Step 6) | 8 | Foundation |
| ARIA usage (5a, 5b) | 5 | Correctness |
| Color/contrast (4a, 4b) | 5 | Partial check from source (full check needs --live) |
| Motion (4c) | 2 | Important but narrower impact |

Formula: `score = (earned_points / applicable_points) * 100`

Skip checks that don't apply (no forms = skip form checks, no images = skip image checks).

**Grade scale:**

| Score | Grade | Meaning |
|-------|-------|---------|
| 90-100 | A | Solid. Minor polish only. |
| 80-89 | B | Good foundation, some gaps. |
| 70-79 | C | Usable but excluding people. Fix the keyboard and form issues. |
| 60-69 | D | Significant barriers. Needs dedicated work. |
| 0-59 | F | Major accessibility failures. Prioritize before shipping. |

## Output Format

```
## Accessibility Audit -- [project name]
**Score:** [score]/100 ([grade])
**Stack:** [detected stack]
**WCAG Level:** AA (target)

### CRITICAL (blocks entire user groups)
1. **[Category]** [file:line] -- [what's wrong] -- [how to fix]

### SERIOUS (degrades experience significantly)
1. **[Category]** [file:line] -- [what's wrong] -- [how to fix]

### MODERATE (should fix)
1. **[Category]** [file:line] -- [what's wrong] -- [how to fix]

### MINOR (polish)
1. **[Category]** [file:line] -- [what's wrong] -- [how to fix]

### Summary
- [X] critical, [Y] serious, [Z] moderate, [W] minor issues
- Top priority: [the one thing to fix first]
```

## Fix Mode (--fix flag)

**Safe to auto-fix:**
- Add `lang="en"` to `<html>` if missing
- Add `alt=""` to images that are clearly decorative (inside links with text, CSS background replacements). Add `alt="TODO: describe this image"` to content images missing alt.
- Add `aria-hidden="true"` to decorative SVG icons
- Remove `role="button"` from `<button>` elements (redundant)
- Remove `role="link"` from `<a>` elements (redundant)
- Add `rel="noopener"` to `target="_blank"` links if missing

**Requires confirmation:**
- Adding `aria-label` to icon buttons (needs the right label text)
- Adding skip-to-content link (needs to match page structure)
- Wrapping inputs with `<label>` (needs to not break layout)
- Adding `prefers-reduced-motion` CSS (needs to know which animations to disable)

**Never auto-fix:**
- Color contrast (requires design decisions)
- Focus management in modals (requires understanding the component logic)
- Heading hierarchy (restructuring headings can break design intent)
- ARIA live regions (requires understanding what content updates dynamically)

## Rules

- **React Native / Expo projects:** Skip this audit. Native accessibility uses `accessibilityLabel`, `accessibilityRole`, `accessibilityState`, etc. -- a completely different model. Note this and stop.
- **Don't flag framework internals.** Next.js `<Head>`, Astro `<Layout>`, Jinja `{% block %}` are structural. Check what they render, not the templating syntax.
- **Stack-aware fixes.** In JSX it's `htmlFor`, in HTML it's `for`. In JSX it's `tabIndex`, in HTML it's `tabindex`. Get it right.
- **Severity by impact.** A missing skip link is moderate. A click handler on a div with no keyboard equivalent is critical -- it's a complete blocker for keyboard users.
- **Don't penalize what doesn't exist.** No forms = no form audit. No media = no media audit.
- **Be specific about fixes.** "Add alt text" is useless. "Add `alt=\"<describe the image content>\"` to `<img>` at `src/templates/<file>.html:<line>`" is useful.
- **Prefer semantic HTML over ARIA.** The fix for a clickable div is almost always "make it a button," not "add role=button and tabindex=0 and onKeyDown."


## Skill Run Logging

After completing this skill, append a log entry to `~/.claude/skill-log.md` under the Run Log table. New entries at the top (most recent first).

Format: `| YYYY-MM-DD | /skill-name | target (details) | one-line result |`
