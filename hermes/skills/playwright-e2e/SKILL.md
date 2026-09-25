---
name: playwright-e2e
description: "Use when running or fixing Playwright e2e specs."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [playwright, e2e, selectors, testing, lifecycle]
    category: software-development
    related_skills: [laravel-pest-testing, dogfood, test-driven-development]
---

# Playwright E2E Maintenance

Evidence-driven selector fixes, stale-assertion triage, and reliable e2e
lifecycle scripts.

## When to Use

- A Playwright spec fails and you need the correct selector or assertion.
- Updating e2e specs after a frontend/component change.
- An e2e run hangs or reports "running" with no output.
- Adding an e2e test for a feature that lacks one.

## Procedure

0. **Provision the run's inputs before touching the runner** — a
   lifecycle script assumes they exist and fails late (or mutates the
   wrong data) otherwise:
   - **`.env.e2e` is usually gitignored, so a fresh clone does not have
     it.** Build it from `.env.e2e.example`, filling every placeholder
     (`<same as dev>`: `APP_KEY`, `DB_*`, `REDIS_PASSWORD`) from the dev
     `.env`, and keep `BASE_URL`/`APP_URL` on the port the script serves.
     Playwright loads it via dotenv, and the fixtures hard-require the
     `TEST_*` vars — no fallbacks.
   - **`DB_DATABASE` in that file must name an existing, e2e-only
     database**, created from the app role with the same template the
     app's migrations need (e.g. `TEMPLATE=template_postgis` for a
     PostGIS app). The script runs `migrate:fresh --seed` against
     whatever database `.env.e2e` names — point it at the dev or the
     Pest test database and that data is gone.
   - **Browsers**: `npx playwright install chromium` when
     `~/.cache/ms-playwright` is empty; `npx playwright --version`
     succeeding says nothing about the binaries.
   - **The server must render the language the specs assert.** Copy the
     app's `APP_LOCALE` (plus its fallback/faker keys) into `.env.e2e` —
     example env files routinely omit it and `config/app.php` then falls
     back to `en`. The `locale:` option in `playwright.config.ts` is
     browser-side only (`Intl`, `Accept-Language`) and cannot change which
     translations the server sends.
1. **Run through the project's lifecycle entry point** (the script that
   swaps env, migrates/seeds, starts the server, runs tests, restores
   state) — not a bare `npx playwright test`, or credentials/fixtures are
   missing. Pass the specific spec path as argument for a fast loop.
2. **Triage from saved evidence, not memory.** On failure Playwright
   writes `test-results/<slug>/error-context.md` (ARIA page snapshot), a
   screenshot, and a video. Read the snapshot FIRST — it is ground truth
   for what the DOM contained at failure time.
3. **Fix selectors against the snapshot** (role/name, label, or prefix
   attribute matchers), then re-run only the failing spec.
4. **Re-run the whole spec file when green** to catch cross-test
   interactions (beforeEach state, shared seed data). After an
   env/config fix, go further and re-run the **whole suite**: the fix
   changes what the server returns to every spec, so specs that passed
   under the broken config can flip — only a full green run proves it.

## Pitfalls

- **Exact attribute-equality selectors time out while the element is
  visibly present** — UI kit components rewrite attributes at render
  (trailing/trimmed spaces, generated ids), so source-code attributes are
  not DOM attributes. Copy the value from the error-context snapshot or
  use `[attr^="prefix"]` / role+name locators.
- **Timeout on `fill`/`click` means zero matching nodes, not a slow
  page** — the wait budget was spent waiting for a selector. Diff your
  selector against the snapshot instead of raising timeouts.
- **Text assertions rot when the frontend is redesigned** while the
  element itself still exists (the label simply reads something else).
  Prefer structure, visibility, and relative counts (capture before,
  compare after) over literal copy; probe the snapshot to learn the
  current text.
- **Many unrelated specs failing on text at once, with the locators still
  matching, is an env problem — not rot.** When the snapshot shows the
  expected elements carrying the wrong *language* (English pagination or
  validation copy where the spec expects localized text), the server ran
  with the wrong locale. Fix the env file, then re-run the same specs
  untouched; only edit assertions if they still fail.
- **A lifecycle script with `set -e` and no `trap` dies without cleanup:**
  its spawned server survives, inherits the output pipe, and a wrapper
  like `script.sh | tail -60` never sees EOF — the tracked background
  command reports "running" forever with empty output and no completion
  notification. Diagnose with `pstree -p <pid>` /
  `ps -o pid,ppid,cmd`; kill the orphan by EXACT PID (a `pkill -f`
  pattern that appears in your own command line matches and kills your
  own shell — use the `[b]racket` trick or the PID) to release the pipe,
  then restore any swapped env files by hand. When authoring such
  scripts: `trap cleanup EXIT`.
- **You cannot add the `trap` when the lifecycle script is someone
  else's — wrap it instead.** Run it redirected to a file, capture
  `$?`, then unconditionally repair state: restore the backed-up env
  file, delete the run-state file the fixtures read, and kill the
  spawned server with a bracketed pattern (`pgrep -f 'serve --port=800[1]'`),
  exiting with the script's own status. Do the repair in the wrapper
  even when the run is green — the cleanup lines inside the script may
  never have executed.
- **Fixture data must support the assertion you plan**: a filter test
  asserts nothing if every seeded row has NULL in the filtered column
  (NULL fails every LIKE regardless of escaping) — seed a real value so
  only correct escaping can produce the no-match result.
