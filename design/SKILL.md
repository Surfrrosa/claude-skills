---
name: design
description: Design psychology audit. Asks "does the system serve the user?" not "does the page match the system?" — evaluates typography appropriateness (display vs body vs UI vs reading-fatigue fonts), color emotional register, visual hierarchy, motion + interaction feedback, brand-voice ↔ visual-register match, reading fatigue indicators, and first-impression speed. Outputs designed-with-intent recommendations per-surface, not bug lists. Use BEFORE running /cohesion to validate that the source of truth is the right target. Different from /a11y (accessibility) and /cohesion (consistency).
---

# Design Psychology Audit

Evaluates whether the project's design system serves users — not whether pages match the system. The premise: enforcing cohesion against a system that's typographically wrong, color-clashing, or voice-mismatched just enforces a bad target perfectly.

This skill outputs **recommendations**, not bugs. Each finding cites the source-of-truth document, the brand voice it should serve, and the typography/perception research it draws on. The user makes the final call on every recommendation.

This skill is different from:
- `/cohesion` — does each page match the canonical system? (target validation)
- `/a11y` — does the page meet WCAG? (accessibility floor)
- `/zombie` — is there dead/duplicate code? (cleanup)
- `/walkthrough` — are there missing-feature gaps? (UX coverage)
- `/voice` — does this content match brand voice? (writing register)

## Arguments

- No args: audit current working directory
- A project name or path (e.g., `myapp`, `other-app`)
- `--surface <name>`: focus on a single surface (`marketing`, `blog`, `chat`, `forms`, `readings`)
- `--write <path>`: write the report to a file path (default: print in chat only)

## Project Registry

| Alias | Path | Notes |
|-------|------|-------|
| `myapp` | (add your projects here) | (add your projects here) |

**For React Native / Expo projects:** native typography is rendered by the platform (system fonts via `Platform.select` or `useFonts`); accessibility/sizing follows OS conventions; color semantics ride iOS/Android dynamic theming. The design-psychology evaluation framework still applies, but the prescription mechanics differ. Note this in the report and adapt — e.g., flag font-weight payload via `expo-font` vs. CSS @font-face.

## Process

### Step 0 — Read the stated intent

Resolve the project. Read in order:

1. `CLAUDE.md` — documented design conventions
2. `DESIGN_SYSTEM.md` (or equivalent) — visual contract
3. `BRAND_VOICE.md` (or equivalent) — written voice the visual register should match
4. Any project-specific design docs (e.g., `observatory_design_contract.md`)

Extract:
- **Stated brand vibe** — adjectives the brand chose for itself (e.g., "warm, mystery-school, late-night, somatic, witty")
- **Typography choices + rationale** — what fonts, why
- **Color palette + rationale** — what colors mean
- **Voice register** — formality level, sentence rhythm, word choice
- **Stated audience** — who the brand is for

These define the *intent*. The audit asks how well the implementation serves the intent.

### Step 0.5 — Design system integrity check (gating)

Before evaluating *whether the design serves the user*, confirm that a *design system actually exists*. Without a single source of truth, downstream cohesion is a moving target — every surface drifts and no audit can stick.

The four-layer minimum every project should have:

| Layer | What it is | Where it lives |
|---|---|---|
| **1. Design tokens** | Colors, fonts, spacing, breakpoints as named CSS variables (or equivalent) | `:root { --foundation, --accent, --font-display ... }` in canonical stylesheet |
| **2. Component library** | Canonical classes/macros for buttons, inputs, cards, states | `.btn-primary`, `.field-label`, etc. in canonical stylesheet + `_macros/*.html` (or framework equivalent) |
| **3. Documentation** | When-to-use rules, decision frameworks, anti-patterns | `docs/DESIGN_SYSTEM.md`, `docs/BRAND_VOICE.md`, design contracts |
| **4. Drift enforcement** | Automated checks that fail CI when a page reinvents the wheel | `tests/test_template_cohesion.py` (or equivalent), pre-commit hooks, project-level Claude Code hooks |

For each layer, mark ✅ / 🟡 (partial) / 🔴 (missing). If any layer is 🔴, the design audit's recommendations cannot stick — **fixing the missing layer is the immediate next task**, before any per-surface psychology evaluation.

If two or more layers are 🔴, the audit STOPS here and reports: "no design system to evaluate against — bootstrap one before continuing." This is a strict gate, not a recommendation.

If everything is ✅ or 🟡, proceed to Step 1.

**Output of this step** (always include in the report):

```
### Design system integrity
1. Tokens: [✅ / 🟡 / 🔴] — [evidence + gaps]
2. Components: [✅ / 🟡 / 🔴] — [evidence + gaps]
3. Documentation: [✅ / 🟡 / 🔴] — [evidence + gaps]
4. Enforcement: [✅ / 🟡 / 🔴] — [evidence + gaps]
```

Reasoning: a design audit asking "is this typography choice serving the user?" is meaningless if the typography choice is just one of 12 inconsistent variations across the codebase. Establish the source of truth first.

### Step 1 — Identify surfaces

Different surfaces have different psychological demands. Inventory which surfaces this project actually has:

| Surface | Reading mode | Time on surface | Cognitive load |
|---|---|---|---|
| **Marketing landing** | Glance + scan | 5-30 sec | Low — first impression dominates |
| **Marketing scrollers** (about, gift, services) | Scan + occasional read | 30 sec - 2 min | Low-medium |
| **Long-form** (blog) | Sustained read | 2-15 min | High — fatigue accumulates |
| **Chat** (real-time conversation) | Fast back-and-forth | 5-30 min per session, multiple sessions/day | Medium — conversational not academic |
| **Forms** (signup, checkout, profile) | Brief active task | 30 sec - 3 min | Medium — error-recovery sensitive |
| **Readings/reports** (paid output, often PDF) | Sustained ritual read | 10-30 min | High — but tone matters more than ergonomic optimization |
| **Daily card / micro-content** | Single moment, ritual | 30 sec - 2 min | Low — single emotional moment |
| **Account / settings** | Brief active task | 1-5 min | Medium — trust + clarity |

For each surface the project has, find the representative file(s) and gather:
- The base template / component used
- The fonts in use (resolve `var(--font-*)` to actual family names)
- Body text size + line-height + max-width
- Background + foreground contrast
- Primary CTA styling

### Step 2 — Per-surface evaluation

For each surface, score across these dimensions:

#### 2a. Typography appropriateness

Display vs body vs UI vs reading fonts have different properties. Match each surface to the typography it deserves:

| Surface | Wants typography characteristics |
|---|---|
| Marketing landing (headline) | Display fonts. High contrast, character, brand-defining. Bodoni / Didot / Playfair — fine here. |
| Marketing body (short scroll) | Either body serif or humanist sans. Whatever matches the brand. Less fatigue-sensitive. |
| Long-form blog | **Screen-optimized body serif or sans.** Higher x-height, taller lowercase, balanced stroke contrast. Lora, Source Serif, Charter, Merriweather, Inter, IBM Plex. Display fonts FAIL here. |
| Chat | **System or humanist sans.** Real-time scanability, native messaging-app feel, no formal register. system-ui, Inter, Geist, SF Pro. Display serifs FAIL here. |
| Forms (input fields) | Whatever the body uses, but text must render reliably at 14-16px (some display fonts blur at small sizes). |
| Readings/reports (sustained read in ritual context) | Body serif acceptable; ritual register can carry slight readability cost. |
| Daily card / micro-content | Either display or body — depends on whether the moment is "iconic" or "intimate." |
| Account / settings | UI-optimized sans; clarity > character. |

For each surface, evaluate:
- Is the font actually appropriate for this surface's reading mode?
- Are the weights loaded the right ones for the use case?
- Is the font payload reasonable (target: <200KB total fonts for marketing pages)?

Common smells:
- Display serif used as body text on long-form (high reading fatigue)
- Body serif used in chat (formality mismatch)
- 5+ static weights of one family loaded (variable font candidate)
- Italic + bold + light + regular all loaded but page only uses regular + bold (waste)

#### 2b. Color psychology

Colors carry register. Cool/warm, saturated/muted, light/dark all communicate.

Evaluate the palette against the stated brand:
- Foundation (background) — does it match the stated vibe? (e.g., near-black foundation = late-night, mystery, intimacy. Off-white foundation = airy, accessible, clinical.)
- Accent — does it carry the right emotional register? (e.g., copper/gold = alchemical/devotional. Bright primary = energetic/playful. Muted earth = grounded/somatic.)
- Text — is the contrast right for the foundation? (Pure white on near-black is harsh; off-white softens.)
- Hover/active states — are they tokenized? (If pages use ad-hoc lightened/darkened versions of accent without canonical tokens, that's drift.)

Dark-mode-specific concerns:
- Pure white on pure black = high contrast but eye strain. Off-white + warm-black is gentler.
- Color saturation reads differently on dark — accents can over-pop.
- White text on bright accent backgrounds may fail contrast or feel harsh.

#### 2c. Visual hierarchy + flow

Where does the eye go first? Second? Third? Test against:

- **F-pattern vs. Z-pattern.** Long-form content reads as F (top-left scan, then progressive descent). Marketing pages typically read as Z (corner-to-corner). Does the layout match the reading mode?
- **CTA invitation.** Does the primary call-to-action *look* clickable, *feel* inviting, and stand out from secondary actions? Or does it blend into the body and require explicit attention?
- **Scanability.** Are headings strong enough to anchor scrolling? Are list items distinguishable from paragraphs? Are pricing/key-numbers visually emphasized?

Smells:
- Multiple visual contenders for "first thing the eye lands on" (decision paralysis)
- Primary CTA visually identical to secondary text-link (lost conversion)
- All-equal headings (no hierarchy)
- Body text and metadata indistinguishable (no scan path)

#### 2d. Motion + interaction feedback

Motion communicates. Test:
- Does animation **invite** (ease-out for incoming content, gentle parallax) or **distract** (autoplay, looping, blocking)?
- Does interactive feedback feel responsive? (Click → immediate visual change vs. silent latency)
- Does `prefers-reduced-motion` get honored across the codebase?
- Is the perceived performance fast on slow networks? (Skeleton screens vs. spinners vs. nothing)

Smells:
- Loading spinners that block content for >500ms
- Hover states that don't show on touch devices (feature-detection failure)
- Animation timing that doesn't reflect physics (linear easing for natural motion is wrong)
- Blocking transitions that prevent rapid interaction

#### 2e. Brand-voice ↔ visual-register match

The voice and the visual must agree.

If the voice is "warm friend who texted you something specific" but the typography is wedding-invitation formal, they're at war. The user feels something is off without knowing why.

Compare:
- Stated voice (from BRAND_VOICE.md): formality level, sentence rhythm, word register
- Implemented visual: font formality, color warmth, layout density, motion personality

Smells:
- Conversational voice + display-serif body type (mismatch)
- Friendly imagery + clinical color palette (mismatch)
- Casual copy + ALL CAPS section labels (mismatch — ALL CAPS reads as institutional/cold)
- Modern hyper-minimal layout + warm/intimate copy (mismatch — minimalism reads as distant)

#### 2f. Reading fatigue indicators (long-form surfaces)

For blog / readings / chat / any surface with sustained text:

- **Line-length.** Sweet spot: 50-75 characters per line. Too narrow = constant return saccades. Too wide = lost-place fatigue.
- **Line-height.** Body serif: 1.5-1.7 typical. Display serif used as body needs MORE (1.7-1.9) because of stroke contrast.
- **Font size.** 16-18px for body on screen. 14-15px is too small for sustained read. 20px+ gets stuttery for fast readers.
- **x-height-vs-cap-height ratio.** Higher x-height = easier read at smaller sizes. Cormorant has low x-height (designed for large display use); Lora has high x-height (designed for screen body).
- **Paragraph rhythm.** Are paragraphs long blocks (heavy) or do they breathe (skimmable)? Voice docs typically prescribe this.

#### 2g. First-impression speed

What does a stranger feel in 0.4 seconds on the marketing landing?

Test by closing your eyes, opening them on the page, immediately closing them again, and asking: what was the dominant emotional tone? Does it match the stated brand?

This is squishy but real. Smells:
- Page feels generic (no character → forgettable)
- Page feels overwhelming (too much hierarchy → bouncing)
- Page feels institutional when brand wants "warm friend" (mismatch)
- Page feels playful when brand wants "serious authority" (mismatch)

### Step 3 — Verdict per surface

For each surface, render a verdict:

- **✅ SERVES** — the design choice for this surface matches the intent + the user's psychological needs. Keep.
- **⚠️ DRIFT** — the design choice for this surface partially serves but has gaps. Specific recommended adjustments.
- **🔄 CONSIDER TESTING** — the design might serve, but it's worth A/B testing or trying an alternative.

### Step 4 — Recommendations + decisions to surface

For every DRIFT or CONSIDER finding:
- Cite the specific brand voice / design doc passage that establishes intent
- Cite typography/color/UX research the recommendation draws on (succinct — one sentence reference is enough)
- Propose 1-3 concrete alternatives with reasoning
- Flag what user input is needed to lock the decision

### Step 5 — Output

Compose a designed-with-intent report:

```
## Design Psychology Audit — [project]
**Audit date:** YYYY-MM-DD
**Stated brand:** [vibe + voice + audience]

### Per-surface verdicts

#### Marketing landing
**Verdict:** [✅ / ⚠️ / 🔄]
**Findings:** ...
**Recommendations:** ...
**Decisions for user:** ...

#### Blog (long-form)
**Verdict:** ...
...

#### Chat
**Verdict:** ...
...

[continue for every applicable surface]

### Cross-cutting findings
- Color tokens worth canonizing: ...
- Font payload concerns: ...
- Brand-voice ↔ visual register mismatches: ...

### Recommended source-of-truth changes
1. Add token X
2. Swap font Y on surface Z
3. ...

### Decisions to lock with user before code changes
1. ...
2. ...

### What this audit explicitly DOES NOT recommend
- [things the audit considered but rejected, with reasoning — defends against scope creep]

### Confidence + uncertainty
- High confidence on: ...
- Uncertain (worth testing): ...
- Out of scope (needs separate audit): ...
```

If `--write <path>` flag passed, write the report to that path AND print it in chat.

## Rules

- **Recommendations, not bugs.** Use this skill to evaluate the system's appropriateness, not enforce conformity. Bug-finding is /cohesion's job.
- **Cite the brand intent.** Every recommendation should reference what the brand stated about itself; this is not generic typography advice, it's "here's how the implementation matches what YOU said you wanted."
- **Surface-by-surface.** Don't pretend marketing-page typography logic applies to chat. Different surfaces have different demands; the audit must respect that.
- **User has final call.** This is brand judgment, not a math problem. Surface decisions; let the user lock them.
- **One audit cycle ≠ one redesign.** Findings get prioritized; not every drift gets fixed in one sweep. The audit ranks; the user picks scope.
- **Confidence calibration matters.** "Cormorant Garamond is wrong for chat" is high-confidence (well-established UX research). "Try Fraunces for blog" is medium-confidence (Fraunces is good but Lora or Source Serif might suit better — testing helps). Be explicit about confidence.

## What this skill explicitly does NOT do

- **Doesn't run accessibility checks** — that's `/a11y`.
- **Doesn't enforce consistency** — that's `/cohesion`.
- **Doesn't redesign anything** — produces recommendations the user reviews.
- **Doesn't second-guess voice docs** — assumes the voice docs are correct; audits visual register against THAT.
- **Doesn't do live A/B testing setup** — flags candidates, points at testing tools if relevant.

## Skill Run Logging

After completing this skill, append a log entry to `~/.claude/skill-log.md` under the Run Log table. New entries at the top (most recent first).

Format: `| YYYY-MM-DD | /skill-name | target (details) | one-line result |`
