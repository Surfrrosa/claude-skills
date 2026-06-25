---
name: privacy
description: Audit data collection, consent flows, exposed secrets, and privacy policy accuracy
---

# Privacy & Data Handling Audit

Audit what data a project collects, where it goes, whether consent is handled properly, whether anything sensitive leaks to the client, and whether the privacy policy matches reality.

## Arguments

The user may specify:
- No args: audit current working directory
- A project name or path (e.g., `myapp`, `other-app`)
- `--live`: also fetch live URLs to check what's actually sent to the browser
- `all`: audit all web projects

Parse the argument string. Examples:
- `/privacy` -- audit current project
- `/privacy myapp` -- audit myapp
- `/privacy myapp --live` -- audit source + check live site
- `/privacy all` -- audit all projects

## Project Registry

| Alias | Path | Live URL | Collects user data? |
|-------|------|----------|---------------------|
| `myapp` | (add your projects here) | (add your projects here) | (add your projects here) |

## Process

### Step 0: Detect the project

1. Resolve path from alias or cwd
2. Detect the stack
3. Read `CLAUDE.md` for any privacy/data handling notes
4. Determine the risk tier:
   - **High risk:** Collects PII (names, birth dates, email), processes payments, stores user data in a database. Apps that handle account data, payments, and PII are here.
   - **Medium risk:** Has analytics, uses cookies, accepts file uploads. Sites with cookies or analytics are here.
   - **Low risk:** Static content, minimal or no analytics. These projects are here.

Scale the audit depth to the risk tier. Don't spend 20 minutes auditing a static HTML page.

### Step 1: Data inventory

Map every piece of user data the project touches. Search source code for:

#### 1a. Form inputs
Search all templates/components for `<input>`, `<select>`, `<textarea>`, and form submission handlers. For each form, document:
- What fields are collected
- Where the form submits to (action URL or JS handler)
- What happens to the data after submission (stored in DB? sent to API? emailed?)

#### 1b. Cookies
Search for:
- `document.cookie` / `cookies.set` / `setCookie` / `Set-Cookie` header
- Cookie consent libraries (`cookieconsent`, `cookie-consent`, `@posthog/cookie-consent`, custom implementations)
- Third-party scripts that set cookies (PostHog, Stripe, Google Fonts)

Document each cookie:
- Name and purpose
- First-party or third-party
- Session or persistent (and expiry if persistent)
- Set before or after consent

#### 1c. Analytics and tracking
Search for analytics initialization:
- PostHog: `posthog.init`, `posthog.capture`, custom events
- Google Analytics: `gtag`, `ga(`, `analytics.js`
- Plausible: `plausible` script
- Any other tracking scripts

For each, check:
- Is it loaded before or after cookie consent?
- What custom events are tracked? (List them -- these are data collection points)
- Is IP anonymization enabled?
- Are user properties being set? (`posthog.identify`, `posthog.people.set`)

#### 1d. Third-party data sharing
Search for API calls to external services that receive user data:
- **Stripe:** What customer data is sent? (email, name, metadata)
- **Supabase:** What tables store user data? What columns?
- **Resend:** What email addresses are stored? Newsletter lists?
- **Anthropic API:** Is user-provided content sent to the API? (birth data for readings?)
- **Prodigi:** What customer data is sent with print orders?
- **GeoNames:** Are user locations sent?
- **Sentry:** Does error tracking capture user data in breadcrumbs/context?

For each service, note:
- What data is shared
- Whether it's necessary for the service to function
- Whether the privacy policy discloses this sharing

#### 1e. Database storage
If the project has a database (Supabase, SQLite, Postgres):
- Search for table definitions, migrations, or schema files
- Identify tables that store PII (users, customers, orders, readings, etc.)
- Check for data retention: is there any mechanism to delete old data?
- Check for encryption at rest (Supabase handles this, but custom DBs might not)

### Step 2: Consent flows

#### 2a. Cookie consent
- Does a cookie consent banner/modal exist?
- Does it appear before any cookies are set or tracking scripts load?
- Does it offer granular choices (analytics vs. functional vs. marketing) or just "accept all"?
- Does declining actually prevent tracking? (Check: does PostHog init run when cookies are declined?)
- Is the choice persistent? (Cookie or localStorage storing the preference)
- Can the user change their choice later? (Link in footer, settings page)

#### 2b. Data collection consent
For projects collecting PII:
- Is there a clear disclosure before collecting birth data? ("We use this to generate your chart")
- Is there consent before adding email to a newsletter list? (Double opt-in?)
- Is there a way to request data deletion?
- Is there a way to export personal data?

#### 2c. Payment consent
- Stripe handles PCI compliance, but check:
  - Are you storing card details yourself? (You should NEVER be. Stripe handles this.)
  - Are you storing more customer data from Stripe than needed?
  - Is the checkout flow clear about what's being purchased and the price?

### Step 3: Client-side leaks

Search the source code for secrets or sensitive data that could end up in the browser:

#### 3a. Environment variables in client code
- **Next.js:** Only vars prefixed with `NEXT_PUBLIC_` are sent to the browser. Search for `process.env.` in client components (`use client` files, or files imported by them) -- any non-`NEXT_PUBLIC_` var used in client code will be `undefined` but indicates a mistake.
- **Astro:** Only vars prefixed with `PUBLIC_` are sent to the browser.
- **Flask/Jinja:** Check templates for `{{ config.SECRET_KEY }}` or similar. Template variables are rendered server-side but verify no secrets leak into `<script>` tags or data attributes.
- **Vite:** Only vars prefixed with `VITE_` are sent to the browser.

**Flag:** Any secret key, API key with write access, database URL, or webhook secret that could reach the client.

#### 3b. API keys with wrong scope
- **Stripe:** `pk_live_` and `pk_test_` (publishable keys) are safe for client. `sk_live_` and `sk_test_` (secret keys) must NEVER be in client code.
- **Supabase:** `anon` key is safe for client (with RLS). `service_role` key must NEVER be in client code.
- **PostHog:** Project API key is safe for client (it's designed for this).
- **Anthropic:** API key must NEVER be in client code.
- **Resend:** API key must NEVER be in client code.

**Flag:** Any secret-tier key in a file that could be bundled for the client.

#### 3c. Sensitive data in HTML/JS
- Check server-rendered pages for user data embedded in `<script>` tags, data attributes, or JSON-LD that shouldn't be public
- Check for user data in error messages (stack traces showing DB queries, user IDs in URLs)
- Check for sensitive data in `console.log` statements

### Step 4: Privacy policy audit

Search for privacy policy pages/files:
- `/privacy`, `/privacy-policy`, `privacy.html`, `privacy.md`
- Footer links to privacy policy
- Check `<a>` tags and navigation for privacy-related links

If a privacy policy exists, verify it covers:

| Requirement | What to check |
|-------------|---------------|
| **What data is collected** | Does the policy list all data types found in Step 1? |
| **Why it's collected** | Is there a stated purpose for each data type? |
| **Third-party sharing** | Does it name the services data is shared with? (Stripe, Supabase, PostHog, Anthropic, Resend, Prodigi) |
| **Cookies** | Does it describe what cookies are set and why? |
| **Data retention** | Does it state how long data is kept? |
| **User rights** | Does it explain how to request deletion or export? |
| **Contact** | Is there an email for privacy questions? |
| **Children** | If the site could attract minors, does it have a COPPA/age disclaimer? |

**Flag mismatches:** If the code collects data the policy doesn't mention, or the policy claims data handling that the code doesn't implement.

If no privacy policy exists and the project collects any user data beyond basic analytics, flag this as critical.

### Step 5: GDPR / CCPA basics

Not a legal audit (you need a lawyer for that), but check the bare minimum:

**GDPR (if serving EU users):**
- Cookie consent before non-essential cookies
- Right to access / right to deletion mechanism
- Clear privacy policy
- Data processing purpose stated
- No pre-checked consent boxes

**CCPA (if serving California users):**
- "Do Not Sell My Personal Information" link (if applicable -- most small sites don't sell data, but if you're sharing with analytics/ad services, it could apply)
- Privacy policy accessible from homepage

**Flag:** Missing consent flows for EU visitors, no deletion mechanism for sites storing PII.

### Step 6: Live check (--live flag)

Use `WebFetch` to check the live site:

1. **Cookie consent:** Fetch the homepage. Does a consent banner appear in the HTML? Are tracking scripts present before any consent interaction?
2. **Privacy policy link:** Is it in the footer? Does it return 200?
3. **Exposed data:** Check the page source for embedded user data, API keys, or debug info
4. **Security headers:** Check response headers for:
   - `Strict-Transport-Security` (HSTS)
   - `X-Content-Type-Options: nosniff`
   - `X-Frame-Options` or `Content-Security-Policy frame-ancestors`
   - `Referrer-Policy`
   - `Permissions-Policy`
5. **Mixed content:** Check for `http://` resource URLs on an `https://` page

## Output Format

```
## Privacy Audit -- [project name]
**Risk tier:** [High / Medium / Low]
**Stack:** [detected stack]

### Data Inventory
| Data type | Source | Stored where | Shared with | Disclosed in policy? |
|-----------|--------|-------------|-------------|---------------------|
| Email | Signup form | Supabase users table | Resend | Yes |
| Birth date/time | Blueprint form | Supabase | Anthropic API | No -- MISSING |
| ... | ... | ... | ... | ... |

### Consent
- Cookie consent: [present / missing / partial]
- Analytics gating: [respects consent / loads before consent / no consent needed]
- Data collection disclosure: [clear / vague / missing]
- Deletion mechanism: [exists / missing]

### Client-Side Security
- Exposed secrets: [none / list]
- Security headers: [X/Y present]

### Privacy Policy
- Exists: [yes / no]
- Matches code: [yes / gaps found]
- Gaps: [list any data collected but not disclosed]

### CRITICAL (legal risk)
1. [issue] -- [what to do]

### WARNINGS (should fix)
1. [issue] -- [what to do]

### CLEAN
- [x] [things that are handled correctly]
```

## Rules

- **This is not legal advice.** Flag issues and explain what to check, but always note that a real privacy audit requires a lawyer. This catches the obvious technical gaps.
- **Scale to risk.** Don't write a 50-item report for a static HTML page with Plausible analytics. Focus effort on projects that actually collect PII.
- **Privacy policy mismatches are critical.** A policy that says you don't share data when the code sends birth dates to Anthropic's API is a real problem.
- **Secrets in client code are critical.** A leaked `sk_live_` Stripe key or `service_role` Supabase key is an emergency, not a warning.
- **Don't read actual .env files.** They contain real secrets. Only read `.env.example` and source code references to understand what variables exist.
- **Don't store findings in memory.** Privacy audit results contain sensitive architectural details. Report in conversation only.
- **Cookie consent must be real.** A banner that says "we use cookies" but loads PostHog before any interaction is theater. The consent must actually gate the tracking.
- **Be practical.** A solo founder with 100 users doesn't need GDPR Article 30 records of processing. But they do need a privacy policy that matches what the code does, a working consent banner, and no leaked API keys.


## Skill Run Logging

After completing this skill, append a log entry to `~/.claude/skill-log.md` under the Run Log table. New entries at the top (most recent first).

Format: `| YYYY-MM-DD | /skill-name | target (details) | one-line result |`
