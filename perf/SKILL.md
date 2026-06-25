---
name: perf
description: Audit performance — bundle size, image weight, render-blocking resources, and Core Web Vitals via PageSpeed
---

# Performance Audit

Analyze a project for performance problems that kill load times and conversions. Two modes: source analysis (what you can catch from code) and live analysis (what Google PageSpeed Insights sees).

## Sample output

A successful run produces something like:

```
## Performance Audit — marketing-site
Stack: Next.js 16

### Images (3.4MB total)
- public/hero.png — 1.8MB PNG, should be WebP (~250KB)
- public/team-photo.jpg — 920KB, unresponsive (no srcset)
- 6 images below the fold loading eagerly (missing loading="lazy")

### JavaScript (480KB bundled)
- src/components/Analytics.tsx — loads PostHog synchronously in <head>; should be async or deferred
- 3 client components could be server components (no hooks, no event handlers)

### Fonts
- Google Fonts <link> missing &display=swap — invisible text during font load

### Caching & Compression
- public/_headers missing — no long cache on static assets

### Summary
- Critical: 2 (hero image weight, synchronous analytics)
- Warning: 3
- Top priority: convert hero.png to WebP (saves ~1.5MB on first load)
```

## Arguments

The user may specify:
- No args: audit current working directory (source only)
- A project name or path (e.g., `myapp`, `marketing-site`)
- `--live`: also hit PageSpeed Insights API for Core Web Vitals on the live URL
- `--fix`: auto-fix safe categories
- `all --live`: audit all web projects against their live URLs

Parse the argument string. Examples:
- `/perf` -- source audit on current project
- `/perf myapp --live` -- source + PageSpeed for myapp
- `/perf all --live` -- PageSpeed for all sites (skip source, just compare live scores)
- `/perf myapp --fix` -- source audit + auto-fix for myapp

## Project Registry

| Alias | Path | Live URL | Stack |
|-------|------|----------|-------|
| `myapp` | (add your projects here) | (add your projects here) | (add your projects here) |

## Source Audit

### Step 0: Detect the project

1. Resolve path from alias or cwd
2. Detect stack: Next.js, Astro, Flask/Jinja, static HTML
3. Read `CLAUDE.md` and build config (`next.config.*`, `astro.config.*`, `vite.config.*`) for relevant settings

### Step 1: Image Weight

Images are the #1 cause of slow pages for most sites. Check thoroughly.

#### 1a. Image formats
Search `public/`, `static/`, `src/assets/`, `src/images/`, and any other asset directories for image files.

- **Flag unoptimized formats:** `.png` and `.jpg`/`.jpeg` files that should likely be `.webp` or `.avif`. Exceptions:
  - Favicons (`favicon.ico`, `favicon.png`) -- these need broad format support
  - OG/social images -- some platforms don't support WebP in meta tags yet. Check on a case-by-case basis.
  - SVGs are fine as-is
- **Flag oversized files:** Any image over 200KB. Any image over 500KB is critical.
- **Count total image weight:** Sum all image file sizes in the project. Compare to a budget:
  - Under 1MB total: good
  - 1-3MB: warning
  - Over 3MB: critical

#### 1b. Image optimization pipeline
Check whether the project has an image optimization strategy:

| Stack | What to check |
|-------|---------------|
| Next.js | Is `next/image` or `<Image>` used? Check for raw `<img>` tags in components. `next/image` auto-optimizes. |
| Astro | Is `astro:assets` or `<Image>` from `astro:assets` used? Check for raw `<img>` tags. |
| Flask/Jinja | No built-in optimization. Check for a build step (sharp, imagemin, squoosh) or CDN with transforms. |
| Static HTML | Check for raw `<img>` tags pointing to unoptimized files. |

**Flag:** Raw `<img>` tags pointing to large files when the framework provides an optimized alternative.

#### 1c. Responsive images
- Check `<img>` tags for `srcset` and `sizes` attributes (or framework equivalents that generate them).
- Check for images with fixed `width`/`height` in pixels that are larger than needed on mobile.
- Above-the-fold hero images should have `loading="eager"` or `fetchpriority="high"`. All other images should have `loading="lazy"`.

**Flag:** Large images without responsive variants, no lazy loading on below-fold images, no priority hints on hero images.

### Step 2: JavaScript Weight

#### 2a. Bundle analysis
For projects with a build step:

| Stack | How to check |
|-------|--------------|
| Next.js | Look for `next build` output in `.next/` or run the build and check the output summary. Check for `@next/bundle-analyzer` in dependencies. |
| Astro | Astro ships zero JS by default. Check for `client:*` directives -- each one adds a JS island. Count them. |
| Vite | Check `dist/assets/` for JS file sizes after build. |
| Static HTML | Count and size all `<script>` tags. |

- **Flag:** Any single JS bundle over 200KB (gzipped). Any total JS payload over 500KB.
- **Flag:** Third-party scripts loaded synchronously in `<head>` without `async` or `defer`.

#### 2b. Third-party scripts
Search for external script URLs in templates/layouts:

- Analytics scripts (PostHog, Google Analytics, Plausible, etc.)
- Chat widgets, support tools
- Font loaders
- Ad scripts
- Social embeds

For each, check:
- Is it loaded with `async` or `defer`?
- Is it loaded from a CDN or self-hosted?
- Could it be loaded later (after interaction, after page load)?

**Flag:** Synchronous third-party scripts in `<head>`, more than 3 third-party scripts total, any third-party script without `async`/`defer`.

#### 2c. Unused JavaScript
- For Next.js: check for `use client` components that could be server components. Any component that doesn't use hooks, event handlers, or browser APIs doesn't need `use client`.
- For Astro: check for `client:load` directives that could be `client:idle` or `client:visible` (defer hydration until needed).
- Search for large libraries imported but potentially underused: `moment` (use `date-fns`), `lodash` (use individual imports), full `@mui/material` or `@chakra-ui/react` imports.

**Flag:** Client components that could be server components, eager hydration that could be deferred, heavy libraries with lighter alternatives.

### Step 3: CSS Weight

#### 3a. CSS delivery
- Check for large CSS files loaded in `<head>` (render-blocking).
- Check for inline critical CSS in the `<head>` with deferred loading of the full stylesheet.
- For Tailwind projects: check if CSS purging is configured (it is by default in v3+, but verify in `tailwind.config.*`).

**Flag:** CSS files over 50KB in `<head>` without critical CSS extraction, Tailwind without purge.

#### 3b. Unused CSS
- For Tailwind: this is handled by purge. Just verify it's on.
- For vanilla CSS: check if the project has multiple stylesheets with overlapping selectors or dead selectors (hard to check exhaustively from source, but flag obviously large CSS files).

### Step 4: Font Loading

Search for `@font-face` declarations, Google Fonts links, or font files in assets:

- **Flag:** Google Fonts loaded via `<link>` in `<head>` without `display=swap` parameter (blocks rendering while fonts load).
- **Flag:** More than 3 font files/variants loaded (each is a separate network request).
- **Flag:** Font files over 100KB each (should be subset or use woff2).
- **Check:** Is `font-display: swap` set in `@font-face` declarations? Missing = invisible text during load.
- **Good:** Self-hosted woff2 fonts with `font-display: swap` and preload hints.

### Step 5: Caching and Compression

Check deploy/server config for:

- **Cache headers:** Check `vercel.json`, `_headers` (Netlify), `railway.json`, server code, or `.htaccess` for cache-control headers on static assets. Images, fonts, and hashed JS/CSS should have long cache (`max-age=31536000, immutable`). HTML should have short/no cache.
- **Compression:** Most CDNs (Vercel, Netlify, Cloudflare) handle gzip/brotli automatically. For self-hosted (Railway/Flask): check for compression middleware (`flask-compress`, `gzip` middleware, nginx config).

**Flag:** No compression for self-hosted projects, no cache headers for static assets.

### Step 6: Stack-Specific Checks

#### Next.js
- `next/image` usage for all content images
- `next/font` for font loading (auto-optimizes and self-hosts)
- `next/script` for third-party scripts with `strategy` prop
- `generateStaticParams` for static generation of dynamic routes
- ISR/SSG vs SSR: are pages that could be static being server-rendered on every request?
- Check for `dynamic = 'force-dynamic'` or `revalidate = 0` on pages that don't need it

#### Astro
- Minimal client directives (zero JS is the goal)
- `<Image>` component for auto-optimization
- Static output mode when possible
- Check `astro.config.mjs` for `compressHTML: true`

#### Flask/Jinja
- Static files served via CDN or with cache headers?
- `flask-compress` or equivalent for gzip?
- Template caching enabled in production?
- Are database queries in views optimized? (N+1 queries, missing indexes -- hard to check from source but flag obvious patterns like queries in loops)

#### Static HTML
- Files minified? (HTML, CSS, JS)
- Are assets fingerprinted/hashed for cache busting?
- CDN in front of hosting?

## Live Audit (--live flag)

Use the Google PageSpeed Insights API via WebFetch:

```
https://www.googleapis.com/pagespeedonline/v5/runPagespeed?url={URL}&category=performance&strategy=mobile
```

Also run with `strategy=desktop` for comparison.

Parse the response for:

### Core Web Vitals
| Metric | Good | Needs Work | Poor |
|--------|------|------------|------|
| LCP (Largest Contentful Paint) | < 2.5s | 2.5-4s | > 4s |
| INP (Interaction to Next Paint) | < 200ms | 200-500ms | > 500ms |
| CLS (Cumulative Layout Shift) | < 0.1 | 0.1-0.25 | > 0.25 |

### Additional Metrics
- FCP (First Contentful Paint)
- Speed Index
- Total Blocking Time (TBT)
- Time to Interactive (TTI)

### Opportunities
Parse the `lighthouseResult.audits` for failed audits and their savings estimates. The most common ones:
- `render-blocking-resources` -- CSS/JS blocking first paint
- `unused-javascript` -- JS loaded but not executed
- `unused-css-rules` -- CSS loaded but not applied
- `modern-image-formats` -- images that should be WebP/AVIF
- `efficiently-encode-images` -- images that could be compressed further
- `properly-size-images` -- images larger than their display size
- `offscreen-images` -- below-fold images loaded eagerly

## Output Format

### Single Project (source)

```
## Performance Audit -- [project name]
**Stack:** [detected stack]

### Images ([total weight])
- [findings]

### JavaScript ([total weight])
- [findings]

### CSS
- [findings]

### Fonts
- [findings]

### Caching & Compression
- [findings]

### Stack-Specific
- [findings]

### Summary
- **Critical:** [count] -- [list]
- **Warning:** [count] -- [list]
- **Top priority:** [the single most impactful fix]
```

### Single Project (source + live)

Same as above, plus:

```
### PageSpeed Scores
| | Mobile | Desktop |
|---|--------|---------|
| Performance | [score] | [score] |
| LCP | [value] | [value] |
| INP | [value] | [value] |
| CLS | [value] | [value] |
| FCP | [value] | [value] |
| TBT | [value] | [value] |

### Top Opportunities (from Lighthouse)
1. [opportunity] -- est. [savings]
2. [opportunity] -- est. [savings]
3. [opportunity] -- est. [savings]
```

### All Projects (--live only)

```
## Performance Audit -- All Sites

| Site | Mobile | Desktop | LCP | CLS | INP | Top Issue |
|------|--------|---------|-----|-----|-----|-----------|
| myapp | 82 | 95 | 2.1s | 0.02 | 120ms | -- |
| myapp | 94 | 99 | 1.2s | 0.01 | 80ms | -- |
| ... | ... | ... | ... | ... | ... | ... |

Sort by mobile score ascending (worst first).
```

## Fix Mode (--fix flag)

**Safe to auto-fix:**
- Add `loading="lazy"` to below-fold `<img>` tags that are missing it
- Add `fetchpriority="high"` to above-fold hero images
- Add `decoding="async"` to images
- Add `async` or `defer` to synchronous `<script>` tags in `<head>` (unless they must be synchronous)
- Add `font-display: swap` to `@font-face` declarations missing it
- Add `&display=swap` to Google Fonts `<link>` URLs missing it
- Replace `<img>` with framework `<Image>` component where applicable and straightforward
- Add `compressHTML: true` to Astro config if missing

**Requires confirmation:**
- Replacing image formats (PNG/JPG to WebP) -- needs build pipeline
- Adding `next/font` or self-hosted fonts -- changes font loading behavior
- Changing `client:load` to `client:visible` in Astro -- may break user expectations
- Converting SSR routes to SSG -- changes data freshness model

**Never auto-fix:**
- Bundle splitting strategy
- Third-party script removal
- Image content resizing (need to know target dimensions)
- Caching policy changes (affects freshness)
- Database query optimization

## Rules

- **Images first.** For most sites, image optimization gives the biggest performance win for the least effort. Always lead with image findings.
- **Mobile matters more.** PageSpeed mobile scores are what Google uses for ranking. Always show mobile prominently.
- **Don't shame slow scores.** A 60 on mobile for a dynamic Flask app with no CDN is expected. Context matters. Focus on what's fixable.
- **Be specific about savings.** "Optimize images" is useless. "Convert `hero.png` (1.2MB) to WebP (~180KB) -- saves ~1MB on first load" is actionable.
- **Don't duplicate other skills.** This is not `/seo` (no meta tags), not `/a11y` (no accessibility), not `/ship` (no deployment checks). Strictly performance.
- **PageSpeed API is free but rate-limited.** Don't hammer it. One request per URL per strategy is enough. If running `all`, add a brief pause between requests.
- **Source analysis is always valuable.** Even without `--live`, you can catch most issues from the code. Live mode adds real numbers but the source audit is where fixes come from.


## Skill Run Logging

After completing this skill, append a log entry to `~/.claude/skill-log.md` under the Run Log table. New entries at the top (most recent first).

Format: `| YYYY-MM-DD | /skill-name | target (details) | one-line result |`
