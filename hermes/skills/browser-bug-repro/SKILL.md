---
name: browser-bug-repro
description: "Use when reproducing or root-causing a reported web-app bug."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [debugging, browser, playwright, repro, spa, evidence]
    related_skills: [systematic-debugging, dogfood, playwright-e2e, github]
---

# Browser Bug Reproduction

Reproduce a reported web-app bug with a scripted, instrumented browser run, derive the root cause as an evidenced ordering of events, and produce an analysis that a tracker can act on. Complements `systematic-debugging` (which owns the phases/iron law) and `dogfood` (exploratory QA) — use those for general process; this skill carries the concrete web-frontend reproduction procedure.

## When to Use

- A bug report/issue arrives and you are asked to analyze, verify, or reproduce it — before writing any fix.
- The symptom is frontend-visible (wrong position, frozen/missing elements, stale data after navigation) and code reading alone cannot decide between interleavings.
- A prior repro attempt only covered a cold load and the bug is timing- or navigation-dependent.

Don't use for backend/test-only failures — plain `systematic-debugging` covers those.

## Done means

1. A runnable script (in scratch) that goes **red on the reported symptom** — or a documented scenario matrix showing which entry paths are red/green if the symptom is timing-dependent.
2. A **timeline of hooked events with captured state** backing the root cause, expressed as `file:line` + ordering, not a guess from final state.
3. A report that **labels reproduced facts vs mechanism-based inference**, with a console diagnostic the reporter can run to classify their variant.
4. No secret ever printed to stdout, logs, or chat.

## Procedure

### 1. Locate the code path before scripting

Read the code behind the reported behavior first: find the exact `file:line` of the state the symptom implicates (globals, layers, caches, listeners). Know which values to watch — the instrumentation targets come from this read, not guesswork. If the repo has a code-intelligence tool (CodeGraph etc.), use it before grep.

### 2. Standalone repro script, not the repo's e2e suite

Write a throwaway Playwright script under the scratch dir: `require('<repo>/node_modules/playwright')`, launch chromium, drive the page. Prereq on a fresh machine: `npx playwright install chromium`. Keep it out of the repo until it becomes a regression test.

### 3. Credential-safe login

1. Browser vault first: `browser_vault_list` on the login page; if an item exists, type the identifier and `browser_vault_fill`.
2. Local dev app with no saved login: a server-side script (artisan/PHP) generates a random password and writes `{identifier, password}` JSON into the scratch dir; the Playwright script reads the file. The password never enters context.
3. Never print the file contents, never echo the password to stdout or chat, never hardcode one in the script.

### 4. Scenario matrix — run every entry path

A green cold load does not clear a navigation-dependent bug. At minimum: cold load; in-app navigation from each sibling page that shares the feature; back/forward if history is involved. After every in-app navigation **assert `location.pathname`** (or `page.url()`), not just a selector — SPA pages share component selectors, so `waitForSelector` can pass while you never left the page you started on. Log the URL after each step.

### 5. Instrument before navigating

Hook the state writes and object factories for the globals/instances you identified in step 1 — setters via `Object.defineProperty`, factories via wrapper functions, element-identity markers on DOM nodes — then run the scenario. Full recipe, sample compression, and element-reuse checks: `references/browser-state-instrumentation.md`.

### 6. Assert behavior, not just presence

Counting elements proves existence, not correctness. After the trigger, capture `getBoundingClientRect()` (or equivalent) before/after an interaction — pan, zoom, drag, re-submit — and compare. "Present but frozen" and "absent" are different failure modes of one code path; your matrix must be able to tell them apart.

### 7. State the root cause as an ordering

Compose the timeline from hook events: which callback ran, against which instance, connected or not, and what wiped/overwrote it afterward — each with its captured state and `file:line`. The final DOM/state alone is identical under several distinct interleavings, so a conclusion drawn only from the end state is a hypothesis, not a finding.

### 8. Write it up for the tracker

- Scenario matrix (red/green per entry path) first — it bounds the bug.
- Hook timeline second — it proves the mechanism.
- `file:line` for every implicated path, plus structural risks noticed in the same area.
- A console snippet the reporter can paste to classify their variant.
- Fix directions as options, not patches — unless asked to implement.
- **Label reproduced facts vs inference explicitly.** If your repro shows a sibling failure mode (e.g. elements missing) while the report describes another (e.g. elements frozen), say so and explain why they share the mechanism — do not claim you reproduced their exact variant.
- Post with `gh issue comment <n> --repo <owner>/<repo> --body-file <file>`; GitHub MCP writes may fail auth even when reads work — `gh` is the fallback that works.

## Always-on rules

- Secrets only via vault handles or scratch files read by scripts — never into context, stdout, or chat.
- Assert the URL after every in-app navigation; a selector match proves nothing about location.
- Cold-load green never closes a navigation-dependent bug report — run the whole entry-path matrix.
- Assert movement/behavior, not just element counts.
- Every claim in the report maps to executed output; inference is labeled as such.

## Pitfalls

- **Polling hides sub-interval interleavings** — a sampling loop can miss an assign-then-delete inside one interval; hook setters/factories for ordering and poll only coarse state.
- **Shared selectors across SPA pages** produce false scenarios ("the test ran" while navigation silently failed) — hence the URL assertion.
- **One symptom, one mechanism** — missing elements and frozen elements can be the same stale-binding bug under different timing; fix/escalate the acceptance condition, not each symptom.
- **Repo e2e suites seed their own data and env** — running your repro through them risks env swaps and destructive `migrate:fresh`; a scratch script against the running dev server is the safer default.
