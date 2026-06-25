---
name: ship
description: Pre-deploy safety checklist — catches the stuff you miss at 2am
---

# Ship Check

Run a go/no-go checklist before deploying. Catches build failures, leaked secrets, debug artifacts, missing env vars, broken tests, and other landmines that slip through when you're shipping fast and solo.

## Sample output

A successful run produces something like:

```
## Ship Check — marketing-site
Deploy target: Vercel
Branch: main (clean, up to date with origin)

✓ Git state — clean working tree
✓ Build — npm run build passed (12.4s)
✓ Tests — 47 passed, 0 failed
✓ Type check — clean
✗ Secrets — found "sk_live_" in src/lib/stripe.ts:14 (should use env var)
✗ Env vars — NEXT_PUBLIC_GA_ID set locally but not in Vercel dashboard
⚠ Debug — 2 console.log calls in src/app/checkout/page.tsx
⚠ TODO — 1 unresolved TODO in src/checkout.ts:28 (added in this branch)

### Blockers (do not deploy)
1. Hardcoded Stripe secret key — move to env var STRIPE_SECRET_KEY
2. Missing NEXT_PUBLIC_GA_ID in Vercel — analytics will silently fail in prod

### Warnings (deploy with awareness)
1. console.log statements left in src/app/checkout/page.tsx
2. Unresolved TODO in this branch's diff

Verdict: NO-GO. Fix blockers, re-run /ship.
```

## Arguments

The user may specify:
- No args: auto-detect the project from the current working directory
- A project name or path (e.g., `myapp`, `~/code/marketing-site`)
- `--skip-tests`: skip the test suite (useful when tests are slow and you've already run them)
- `--skip-build`: skip the build step (useful when you just built successfully)

Parse the argument string. Examples:
- `/ship` -- full check on current directory
- `/ship myapp` -- full check on myapp
- `/ship --skip-tests` -- skip test suite
- `/ship myapp --skip-build` -- skip build for myapp

## Project Aliases

| Alias | Path |
|-------|------|
| `myapp` | (add your projects here) |

## Process

### Step 0: Detect the project

1. Resolve the project path (alias table or cwd)
2. Read `package.json`, `pyproject.toml`, `Cargo.toml`, or equivalent to identify the stack
3. Read `CLAUDE.md` if it exists for project-specific deploy instructions or conventions
4. Detect the deploy target: check for `vercel.json`, `railway.json`, `fly.toml`, `.github/workflows/deploy*`, `Procfile`, `netlify.toml`, or `eas.json`

### Step 1: Git state

Run these checks:

- **Uncommitted changes:** `git status`. Flag any unstaged or staged-but-uncommitted changes. These won't be in the deploy.
- **Unpushed commits:** `git log @{u}..HEAD --oneline` (if tracking a remote). Flag commits that exist locally but haven't been pushed. If the deploy triggers from a push, these are the changes that will ship.
- **Branch check:** What branch are you on? If it's not `main`/`master` and the project deploys from main, warn. If it's a feature branch, note it.
- **Merge conflicts:** Search for `<<<<<<<`, `=======`, `>>>>>>>` in tracked files. These should never ship.

### Step 2: Build

Skip if `--skip-build` is passed.

Detect and run the build command:

| Signal | Build command |
|--------|--------------|
| `package.json` with `build` script | `npm run build` or `yarn build` or `pnpm build` |
| `pyproject.toml` or `setup.py` | Check for build/compile steps in project docs |
| `Cargo.toml` | `cargo build --release` |
| `Makefile` with `build` target | `make build` |
| Static HTML (no build step) | Skip, note "no build step" |
| Expo/React Native (`eas.json`) | Skip build (EAS builds are expensive). Just verify the config. |

**Pass:** Build exits 0 with no errors.
**Fail:** Build exits non-zero. Show the error output. Stop here — nothing else matters if the build is broken.
**Warning:** Build succeeds but has warnings. Show them. Common warnings that matter: unused exports (tree-shaking issues), large bundle size warnings, deprecation warnings from dependencies.

### Step 3: Tests

Skip if `--skip-tests` is passed.

Detect and run the test command:

| Signal | Test command |
|--------|-------------|
| `package.json` with `test` script | `npm test` |
| `pytest.ini` or `pyproject.toml [tool.pytest]` | `pytest` |
| `Cargo.toml` | `cargo test` |
| `.rspec` | `bundle exec rspec` |
| Go files with `_test.go` | `go test ./...` |

**Pass:** All tests pass.
**Fail:** Any test fails. Show the failure. This is a hard stop.
**No tests:** Note "no test suite found." This is a warning, not a blocker — but flag it. Production apps should have tests.

### Step 4: Secrets and sensitive data

Search the **tracked files** (not node_modules, not .git, not build output) for:

**Hard stops (these must not ship):**
- Hardcoded API keys: patterns like `sk_live_`, `sk_test_`, `pk_live_`, `AKIA`, `ghp_`, `gho_`, `glpat-`, `xoxb-`, `xoxp-`
- `.env` files that are tracked (check `git ls-files` for `.env`, `.env.local`, `.env.production`)
- Private keys: `-----BEGIN (RSA |EC |DSA )?PRIVATE KEY-----`
- Database connection strings with credentials: `postgres://user:pass@`, `mongodb+srv://user:pass@`
- Hardcoded passwords: `password = "`, `PASSWORD = "`, `secret = "` (with a non-empty string value, not a variable reference)

**Warnings (review these):**
- `console.log` / `console.debug` / `console.warn` / `print()` / `debugger` statements in non-test source files. Exclude:
  - Test files (`*.test.*`, `*.spec.*`, `__tests__/`)
  - Config files (`*.config.*`)
  - Server-side logging that's intentional (look for structured loggers like `winston`, `pino`, `logging.getLogger`)
  - Lines that are clearly error handling (`console.error` in a catch block)
- `TODO` or `FIXME` comments in files that changed in the current branch (compared to main). Old TODOs in untouched files are not your problem right now.
- Commented-out code blocks in files that changed. Shipping commented-out code is sloppy.

### Step 5: Environment variables

Check that the project's expected env vars are accounted for:

1. **Find env var references:** Search source code for `process.env.`, `os.environ`, `os.getenv`, `env::var`, `import.meta.env.` patterns. Collect the variable names.
2. **Find env var definitions:** Check `.env.example`, `.env.local.example`, `vercel.json` (env section), `railway.json`, `fly.toml`, `docker-compose.yml`, and any documented env var lists in README or CLAUDE.md.
3. **Compare:** Flag any env var that's referenced in code but has no definition in any example/config file. This is a var that a fresh deploy might be missing.
4. **Check .env.example freshness:** If `.env.example` exists, check whether all vars referenced in code appear in it. Flag missing ones.

This check does NOT read actual `.env` files (those contain secrets). It only compares what code expects vs. what's documented.

### Step 6: Dependency check

Quick sanity check:

- **Lockfile exists:** Does the lockfile match the package manager? (`package-lock.json` for npm, `yarn.lock` for yarn, `pnpm-lock.yaml` for pnpm, `Pipfile.lock` for pipenv, `poetry.lock` for poetry). Missing lockfile = non-deterministic installs.
- **Lockfile in sync:** Run `npm ls --all 2>&1 | grep "missing"` or equivalent. If the lockfile is out of sync with the manifest, the deploy might install different versions than what you tested.
- **No `file:` or `link:` dependencies:** These work locally but break in CI/deploy.

Don't run a full vulnerability audit here — that's what `/vuln` is for. Just check the basics.

### Step 7: Stack-specific checks

#### Next.js
- Check `next.config.*` for `output: 'export'` if deploying to static hosting
- Search for `use client` in files that use server-only features (or vice versa)
- Check for `generateStaticParams` on dynamic routes if using static export

#### Astro
- Check `astro.config.*` for `output` mode matching deploy target
- Verify integrations are installed (e.g., `@astrojs/sitemap` in config but not in dependencies)

#### Python (Flask/Django)
- Check `DEBUG = True` or `debug=True` is not hardcoded in production config
- Check `SECRET_KEY` is not hardcoded
- Check `ALLOWED_HOSTS` or equivalent is configured for production domain

#### Expo / React Native
- Check `app.json` / `app.config.js` for correct bundle identifier and version
- Check `eas.json` profiles match intended build type
- Do NOT trigger an EAS build — those cost money. Just verify the config.

#### Static HTML
- Check for broken relative links (`href="./` or `src="./` pointing to files that don't exist)
- Check that `index.html` exists

### Step 8: Report

Present a clear go/no-go:

```
## Ship Check -- [project name]
**Branch:** main (3 commits ahead of origin/main)
**Stack:** Next.js 16 / Vercel

### BLOCKERS (must fix before deploy)
- [ ] Build failed: [error summary]
- [ ] Merge conflict markers in src/components/Header.tsx
- [ ] Hardcoded Stripe key in src/lib/stripe.ts:14

### WARNINGS (review before deploy)
- [ ] 3 console.log statements in changed files
- [ ] TODO in src/checkout.ts:28 (added in this branch)
- [ ] ENV var RESEND_API_KEY referenced in code but not in .env.example

### CLEAN
- [x] Git state: 0 uncommitted changes
- [x] Tests: 14 passed, 0 failed
- [x] Dependencies: lockfile in sync
- [x] Secrets: no leaked keys
- [x] Stack checks: all clear

### VERDICT: [GO / NO-GO / CAUTION]
```

**Verdicts:**
- **GO** -- zero blockers, zero warnings. Ship it.
- **CAUTION** -- zero blockers, some warnings. Probably fine, but review the warnings.
- **NO-GO** -- one or more blockers. Fix them first.

## Rules

- **Blockers are hard stops.** Failed build, failed tests, leaked secrets, merge conflicts. These are non-negotiable.
- **Warnings are judgment calls.** A `console.log` in a debug utility might be intentional. A TODO in a non-critical path can wait. Report them, let the user decide.
- **Don't fix anything.** This skill reports. It doesn't modify code. If something needs fixing, say what and where — the user (or another skill) handles the fix.
- **Be fast.** Skip slow operations when the user asks (`--skip-tests`, `--skip-build`). A ship check that takes 5 minutes won't get used.
- **Respect project conventions.** If CLAUDE.md says "we don't write tests for this project," don't flag the lack of tests as a warning. If the project has a custom deploy process, note it.
- **Don't duplicate other skills.** This is not `/vuln` (no full vulnerability scan), not `/zombie` (no dead code audit), not `/seo` (no SEO checks), not `/autopsy` (no test quality review). This is strictly "is this safe to deploy right now."
- **Scope to what's shipping.** When checking for TODOs, console.logs, and commented-out code, focus on files that changed in the current branch vs. main. Legacy junk in untouched files is not a deploy blocker.


## Skill Run Logging

After completing this skill, append a log entry to `~/.claude/skill-log.md` under the Run Log table. New entries at the top (most recent first).

Format: `| YYYY-MM-DD | /skill-name | target (details) | one-line result |`
