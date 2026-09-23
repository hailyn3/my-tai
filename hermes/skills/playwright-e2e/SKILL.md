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
   interactions (beforeEach state, shared seed data).

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
- **Fixture data must support the assertion you plan**: a filter test
  asserts nothing if every seeded row has NULL in the filtered column
  (NULL fails every LIKE regardless of escaping) — seed a real value so
  only correct escaping can produce the no-match result.
