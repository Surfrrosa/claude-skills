---
name: thatsweird
description: Browser/OS edge-case sweep. Detects and fixes weird-but-real things browsers do to your site without asking — Chrome auto-dark inversion on dark sites, iOS rubber-band background flash, iOS input zoom, prefers-reduced-motion violations. Idempotent. Scoped to "within reason" — does NOT chase 0.1%-traffic browser quirks.
---

# /thatsweird — Browser Edge-Case Sweep

Some browsers and operating systems silently transform your site in ways you didn't ask for. This skill detects and fixes the small set of these that (a) affect real audiences, (b) have cheap fixes, and (c) you can verify quickly. Long-tail browser weirdness is intentionally NOT in scope.

## Arguments

The user will specify a project by name or path. Known project aliases (same as `/onboard`):

| Alias | Path |
|-------|------|
| `myapp` | (add your projects here) |

If a path is given directly, use it. Mobile-only repos (React Native / Expo) are SKIP candidates — flag and ask before running.

---

## What's in scope

Five fixes total. All are CSS / HTML meta tag changes. No JavaScript. Each is idempotent — safe to run multiple times.

### 1. Chrome auto-dark inversion (DARK SITES ONLY)

**Symptom:** Chrome's "Auto Dark Mode for Web Contents" inverts colors on dark-themed sites, producing near-black text on near-black background. Affects iOS Chrome, Android Chrome, and desktop Chrome with the flag enabled.

**Detection:**
- Determine if the site is dark-themed: scan `:root` / `body` background colors in CSS for hex values where the lightness is low (rough heuristic: hex < `#404040` or `rgb()` with all channels < 64). If unsure, also scan for keywords like `--foundation`, `--background`, `--bg` and inspect their values.
- Check for existing `<meta name="color-scheme">` in `<head>` files (templates, layout files, base HTML).
- Check for `color-scheme:` in CSS `:root` or `html` blocks.

**Fix depends on site type:**

- **Pure dark site:** add `<meta name="color-scheme" content="dark">` and `color-scheme: dark;` in `:root`.
- **Mixed-mode site** (dark page with intentional light surfaces — e.g. cream panels, light hero sections, retro-UI windows): add `<meta name="color-scheme" content="light dark">` and `color-scheme: light dark;` in `:root`. This still disables Chrome auto-dark inversion (the goal) while letting native form controls follow `prefers-color-scheme`, preserving the light-surface aesthetic.
- **Pure light site:** SKIP entirely. Browser default handles light correctly.

**How to detect mixed-mode:**
- Search the main CSS for explicit light backgrounds inside the dark page: `background:\s*var\(--white\)`, `background:\s*#f`, `background:\s*#e`, `background:\s*cream`, etc.
- Look for design conventions like cream panels, light hero sections, retro Win3.1/Mac chrome aesthetics, "redacted document" reveal animations.
- If the project has a CLAUDE.md or design docs mentioning light-on-dark surfaces or retro UI chrome, treat it as mixed-mode by default.
- When in doubt, ask the user. Mixed-mode is the safer default — `light dark` doesn't break pure-dark sites, but `dark` can break mixed-mode sites' native form control popups.

### 2. iOS rubber-band background flash

**Symptom:** When users over-scroll on iOS Safari, they briefly see the underlying page background. If `<html>` doesn't have an explicit background color, they get a flash of white at the edges.

**Detection:** Search the main CSS file for `html { ... background ... }` or `html, body { background ... }`. If only `body` has a background and `html` does not, the bug exists.

**Fix:** Add `html { background: <site's main background color>; }` to the main CSS file. Use the same value as the `body` background.

### 3. iOS input zoom

**Symptom:** iOS Safari auto-zooms into form inputs whose `font-size` is below 16px. Visually disorienting, makes the page feel broken on mobile.

**Detection:** Search CSS for `input`, `textarea`, `select` rules. If `font-size` is set below 16px (or 1rem assuming default 16px root), the bug is present. If no font-size rule exists at all on inputs, the browser default is usually fine but worth confirming.

**Fix:** Add or modify CSS rule:
```css
input, textarea, select {
    font-size: 16px;
}
```
Use the existing project's CSS organization (don't create a new file, append to the main stylesheet near other form rules).

### 4. prefers-reduced-motion respect

**Symptom:** Users with vestibular disorders or motion sensitivity set their OS to "reduce motion." Sites that ignore this can cause physical nausea. It's also a basic accessibility expectation.

**Detection:** Search CSS for `animation:`, `transition:`, `@keyframes`. If animations exist, check if there's a `@media (prefers-reduced-motion: reduce)` block somewhere in the CSS.

**Fix (only if animations exist AND no reduced-motion block):**
```css
@media (prefers-reduced-motion: reduce) {
    *, *::before, *::after {
        animation-duration: 0.01ms !important;
        animation-iteration-count: 1 !important;
        transition-duration: 0.01ms !important;
        scroll-behavior: auto !important;
    }
}
```

### 5. theme-color meta tag (mobile address bar)

**Symptom:** On mobile, browsers tint the address bar / status bar based on the page's `theme-color`. Without it, you get the browser default (usually white or gray) which clashes with a dark site.

**Detection:** Search head templates for `<meta name="theme-color"`.

**Fix (if missing):** Add `<meta name="theme-color" content="<site's main background hex>">` to the head template.

---

## What's NOT in scope (take as they come)

Document these in the report so the user knows we considered them and chose not to act:

- **Safari Reader Mode** — strips styling, not a bug, opt-in by user.
- **Browser translation (Google Translate)** — affects ~0.1% of traffic.
- **Ad blockers hiding elements** — only relevant if you name divs `*-ad`.
- **Windows High Contrast Mode** — real audience but tiny; address per-complaint.
- **Browser extensions** (Dark Reader etc.) — `color-scheme: dark` already handles this.
- **PWA save-to-home-screen quirks** — manifest + theme-color cover 95%.
- **Email client dark mode** — separate ecosystem (Outlook, Gmail mobile). Mention in report if the project sends transactional emails, but DO NOT auto-fix here. Email templates are a separate sweep.

---

## Process

### Phase 1: Detect site type (60 seconds)

1. Resolve the project path from the alias.
2. Identify the framework / template structure: is it Jinja2, plain HTML, React, Astro, Next.js, Svelte? Find the main layout / head template and the main stylesheet(s).
3. Determine if the site is **dark, light, or both:**
   - Read `:root` CSS variables. Look for `--foundation`, `--background`, `--bg`, `--surface`, or whatever the project uses.
   - Check the body background color.
   - Convert hex/rgb to lightness. Dark = lightness < 25%. Light = lightness > 75%. Mixed = both, or uses `prefers-color-scheme`.
4. If the project is mobile-only (React Native, Expo) or non-web (CLI tool, library), STOP. Report "not applicable."

### Phase 2: Detect each edge case

For each of the 5 fixes above, run its detection step. Record:
- Yes Already correct (no fix needed)
- Partial Missing / broken (fix needed)
- N/A N/A for this site (e.g., reduced-motion fix on a site with no animations)

### Phase 3: Apply fixes

Apply fixes for everything in the Partial list. Each fix:
- Touches the smallest possible surface (one line in a template, one block in a CSS file).
- Uses the project's existing conventions (Jinja2 inheritance, CSS variable names, file organization).
- Is idempotent — re-running the skill should detect "already fixed" and do nothing.

### Phase 4: Report

Output a clear summary:

```
## /thatsweird sweep: <project name>

**Site type:** dark | light | both
**Framework:** <detected>
**Files touched:** <list>

### Applied fixes
- Yes Chrome auto-dark inversion (added color-scheme dark)
- Yes iOS rubber-band flash (added html background)

### Already correct
- theme-color meta tag

### Skipped
- prefers-reduced-motion (no animations detected)
- iOS input zoom (no inputs on this site)

### Out of scope (intentional)
- Email dark mode (this project sends transactional emails — recommend a separate pass when ready)

### Next steps
- Review diff: `git diff`
- Test locally with Chrome auto-dark flag enabled
- Commit + push when ready (do NOT push without confirmation)
```

### Phase 5: Hand off

DO NOT commit or push. Tell the user the diff is ready to review and ask if they want it committed/pushed.

---

## Important rules

- **Idempotent.** Running this twice on the same repo should produce no second-pass changes.
- **Don't apply `color-scheme: dark` to a light site.** Detection error is the failure mode here. If unsure, ask the user "is this site dark-themed?" before applying.
- **Respect existing CSS organization.** If a project has a `tokens.css`, `reset.css`, `forms.css` separation, put each fix in the topically-correct file. Don't create new files unless none exists.
- **Don't auto-push.** Per user's standing rule, never push without explicit ask. Apply + commit is fine if the user said "fix and commit"; push only on explicit request.
- **Mobile-only repos:** flag and skip. The skill is for web. React Native / Expo apps have their own dark-mode handling.
- **If the site has a CLAUDE.md**, read it first for any project-specific conventions about CSS organization or template structure.
- **Don't reformat or restyle unrelated code.** Surgical edits only.

---

## Skill Run Logging

After completing this skill, append a log entry to `~/.claude/skill-log.md` under the Run Log table. New entries at the top (most recent first).

Format: `| YYYY-MM-DD | /thatsweird | target (site type) | what was fixed / what was already correct |`
