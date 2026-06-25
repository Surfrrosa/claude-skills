---
name: seo
description: Audit SEO health across all web projects, score them, identify gaps, and optionally fix issues
---

# SEO Audit

Audit SEO health across web projects. Score each project, identify gaps, and provide actionable fix instructions. Can also verify live URLs to confirm what search engines actually see.

## Arguments

The user may specify:
- A project name, path, or `all` -- defaults to `all`
- `--fix` flag to automatically fix gaps (otherwise report-only)
- `--live` flag to also fetch live URLs and verify rendered output

Parse the argument string. Examples:
- `/seo` -- audit all projects, report only
- `/seo myapp` -- audit one project
- `/seo all --live` -- audit all with live URL verification
- `/seo other-app --fix` -- audit and fix gaps for myapp

## Project Registry

Only audit projects that are web-facing (have a URL users visit). Skip CLI tools, mobile apps, and libraries.

| Alias | Path | Live URL | Stack | Deploy |
|-------|------|----------|-------|--------|
| `myapp` | (add your projects here) | (add your projects here) | (add your projects here) | (add your projects here) |

If a project path doesn't exist (e.g., external SSD not mounted), skip it and note it as "skipped -- path not found".

When the user adds new web projects, they should update this table.

## SEO Checks

Each check has a tier, weight, and applies to specific stacks. Run all applicable checks for each project.

### Tier 1 -- Foundational (blocks indexing if missing)

| Check | Weight | How to verify |
|-------|--------|---------------|
| `robots.txt` exists | 10 | Look for `robots.txt` or `public/robots.txt`. For Astro, also check `src/pages/robots.txt.ts`. For Python, check for a `/robots.txt` route. |
| `sitemap.xml` exists | 10 | Look for sitemap file, sitemap plugin/package (`@astrojs/sitemap`, `next-sitemap`), or a `/sitemap.xml` route. |
| Unique `<title>` per page | 10 | Check templates/layouts/pages for `<title>` tags or metadata exports. Flag if a single hardcoded title is used everywhere. |
| `<meta name="description">` per page | 10 | Check templates/layouts/pages for description meta. Flag if missing or identical across pages. |
| Canonical URLs | 8 | For Next.js: check for `alternates: { canonical: }` in metadata exports of `layout.tsx` and page files. For Astro: look for `<link rel="canonical">` in layouts. For Python/HTML: look for `<link rel="canonical">` in base templates. Every indexable page needs a canonical. Missing canonicals cause "Duplicate without user-selected canonical" in Google Search Console. Also verify `metadataBase` is set (Next.js) so relative canonicals resolve correctly. |
| Mobile viewport meta | 5 | Look for `<meta name="viewport">` in the base layout/HTML. |

### Tier 2 -- Ranking Signals

| Check | Weight | How to verify |
|-------|--------|---------------|
| Open Graph meta (`og:title`, `og:description`, `og:image`) | 7 | Check layouts and page templates for OG tags. |
| Twitter Card meta (`twitter:card`, `twitter:title`, `twitter:image`) | 5 | Check for Twitter/X card meta tags. |
| Structured data (JSON-LD) | 7 | Search for `application/ld+json` script tags. Check schema type matches content (Article for blog, Product for shop, Organization for homepage). |
| Image alt text coverage | 6 | Search for `<img` tags and check what percentage have `alt` attributes. Flag any empty or missing alts. |
| Heading hierarchy | 5 | Check each page/template has exactly one `<h1>`. Check H2s and H3s follow logical order. |
| No render-blocking resources | 3 | Check for `<script>` tags in `<head>` without `async` or `defer`. Check for large CSS files without critical CSS extraction. |
| Image optimization | 4 | Check if images use modern formats (WebP, AVIF) or if an image optimization pipeline exists (Astro Image, next/image, sharp). |

### Tier 3 -- Growth Multipliers

| Check | Weight | How to verify |
|-------|--------|---------------|
| Blog schema (`Article`, `datePublished`, `author`) | 5 | If the project has blog content, check for Article structured data on blog pages. |
| RSS/Atom feed | 3 | Look for feed file, `@astrojs/rss`, or a feed route. |
| Internal linking | 2 | Spot-check that pages link to related content (not just nav). Especially blog posts linking to each other. |
| 404 page exists | 3 | Look for `404.html`, `404.astro`, `not-found.tsx`, or a catch-all error template. |
| Favicon and app icons | 2 | Check for `favicon.ico`, `apple-touch-icon`, and `manifest.json`/`site.webmanifest`. |
| `<html lang>` attribute | 2 | Check the root `<html>` tag has a `lang` attribute. |

## Stack-Specific Detection

Before running checks, detect the stack:

| Signal | Stack |
|--------|-------|
| `astro.config.*` in root | Astro |
| `next.config.*` in root | Next.js |
| `pyproject.toml` or `requirements.txt` + templates dir | Python |
| `vite.config.*` without framework plugin | Vite/vanilla |
| `index.html` only, no build config | Static HTML |

Then look for SEO elements in the right places:

- **Astro:** `src/layouts/`, `src/pages/`, `astro.config.mjs` (integrations), `public/`
- **Next.js:** `app/layout.tsx` (metadata export), `app/*/page.tsx`, `next.config.*`, `public/`
- **Python (Flask/Jinja):** `src/templates/`, `src/**/templates/`, route definitions, `static/`
- **Vite/vanilla:** `index.html`, `public/`, any HTML files in `src/`
- **Static:** all `.html` files in root and subdirectories

## Scoring

**Max score: 100 points** (sum of all weights = ~107, normalize to 100).

Formula: `score = (earned_points / applicable_points) * 100`

Only count checks that apply to the project. A project with no blog shouldn't be penalized for missing Article schema.

**Grade scale:**

| Score | Grade |
|-------|-------|
| 90-100 | A |
| 80-89 | B |
| 70-79 | C |
| 60-69 | D |
| 0-59 | F |

## Live URL Verification (--live flag)

When `--live` is passed, use `WebFetch` to check each project's live URL for:

1. **robots.txt accessible:** Fetch `{url}/robots.txt` -- should return 200, not block important paths
2. **sitemap.xml accessible:** Fetch `{url}/sitemap.xml` -- should return 200 with valid XML
3. **Meta tags rendered:** Fetch the homepage HTML, parse for title, description, OG tags, canonical
4. **Structured data rendered:** Check the fetched HTML for JSON-LD script tags
5. **Response headers:** Check for `X-Robots-Tag` header (should not be `noindex` unless intentional)

Compare live results against source code findings. Flag discrepancies (e.g., meta tags in source but not rendered, SSR issues).

## Output Format

### Single Project Mode

```
## SEO Audit -- [project name]
**Score:** [score]/100 ([grade])
**Stack:** [detected stack]
**URL:** [live url]

### Passing checks
- [check name] -- [brief note]

### Gaps (fix these)
1. **[TIER 1]** [check name] -- [what's wrong] -- [how to fix for this stack]
2. **[TIER 2]** [check name] -- [what's wrong] -- [how to fix for this stack]
...

### Live verification (if --live)
- robots.txt: [pass/fail]
- sitemap.xml: [pass/fail]
- Meta tags: [pass/fail with details]
- Structured data: [pass/fail]
```

### All Projects Mode

```
## SEO Audit -- [date]

| Project | Score | Grade | Gaps | Top Priority |
|---------|-------|-------|------|--------------|
| ...     | ...   | ...   | ...  | ...          |

[N] of [M] projects scanned. [X] skipped (path not found).

### [project with gaps] ([score]/100)
1. **[TIER 1]** [gap] -- [one-line fix]
2. ...

### [next project with gaps] ([score]/100)
1. ...
```

Sort the table by score ascending (worst first, needs most attention at top).

## Fix Mode (--fix flag)

When `--fix` is passed, after reporting gaps, automatically fix what can be safely auto-generated:

**Safe to auto-fix:**
- Create `robots.txt` with sensible defaults (`User-agent: * / Allow: / / Sitemap: [url]/sitemap.xml`)
- Add `<html lang="en">` if missing
- Add viewport meta if missing
- Add missing `alt=""` to decorative images (but flag content images for manual alt text)
- Create a basic `404.html` / `404.astro` page

**Requires confirmation before fixing:**
- Adding or modifying meta descriptions (content matters)
- Adding structured data (must match actual content)
- Adding OG/Twitter meta (needs correct image URLs)
- Installing sitemap plugins (modifies dependencies)

**Never auto-fix:**
- Internal linking strategy
- Image optimization pipeline
- RSS feed setup (needs content structure understanding)

Always show what was fixed and what still needs manual attention.

## Important Rules

- **Read-only by default.** Only modify files when `--fix` is explicitly passed.
- **Skip gracefully.** External SSD projects may not be mounted. Note and move on.
- **Stack-aware fixes.** A robots.txt fix for Astro goes in `public/`, for Next.js in `public/`, for Python it's a route or static file. Don't apply wrong-stack fixes.
- **Don't penalize what doesn't apply.** No blog = no blog schema check. No shop = no Product schema check.
- **Tier 1 gaps are always the priority.** Present them first, recommend fixing them first.
- **Be specific.** "Add meta description" is useless. "Add `<meta name='description'>` to `src/layouts/BaseLayout.astro` line 12, after the title tag" is useful.
- **No false positives.** If a sitemap is generated at build time by a plugin, that counts as having a sitemap even if the file doesn't exist in source.
- **Sort by score ascending** in the all-projects table so the projects needing the most work are at the top.


## Skill Run Logging

After completing this skill, append a log entry to `~/.claude/skill-log.md` under the Run Log table. New entries at the top (most recent first).

Format: `| YYYY-MM-DD | /skill-name | target (details) | one-line result |`
