---
name: laravel-pest-testing
description: "Use when a Laravel+Pest+Postgres suite must stay green."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [laravel, pest, phpunit, postgres, flaky, ci, testing]
    category: software-development
    related_skills: [test-driven-development, systematic-debugging, github]
---

# Laravel + Pest + Postgres Test Reliability

Keep a Laravel/Pest suite on Postgres deterministic locally and in CI.
Signature → root-cause table: `references/failure-signatures.md`.

## When to Use

- A Pest run fails locally, or CI's test job is red on a Laravel/Postgres repo.
- Suspected flaky test: green alone, red in the suite, or flips across runs.
- Changing shared setUp traits, test helpers, factories, or seed wiring.
- Writing query/filter tests or asserting on factory-generated data.

## Procedure (every test-touching change)

1. **Classify ALL failures before editing.** Strip ANSI
   (`sed 's/\x1b\[[0-9;]*m//g'`) then group by root cause — fix-one-rerun
   loops hide the rest of the list. Count from the `Tests:` line and the
   exit code, never by eyeballing.
2. **Local gates, in order:** `composer test` → `composer pint` →
   `composer phpstan`. All three green is the bar; CI runs the same three
   plus coverage/parallel variants.
3. **Suspected flake → loop the single file** (`for i in 1 2 3 4 5; do php
   artisan test <file>; done`): each run reshuffles Pest's random execution
   order, so a green/red flip across runs proves order dependence.
4. **Remote-only failure → base drift or env flags first**, not test bugs:
   PR CI runs the merge of head into the current base, so sync the base
   (`git fetch <upstream> <base>` + compare) before editing test code, then
   reproduce with CI's extra flags (`--parallel`, `--coverage`, pinned PHP).
   For a finished job's logs while the run is still going:
   `gh api --allow-escape-sequences repos/<OWNER>/<REPO>/actions/jobs/<JOB_ID>/logs`
   (`gh run view --log-failed` refuses until the whole run completes, and
   the API call needs the flag when logs carry ANSI escapes).
5. **Touching someone else's PR?** Check push rights first:
   `gh api repos/<OWNER>/<REPO> --jq .permissions` — `maintainerCanModify`
   only admits writers of the BASE repo. No rights → deliver via your own
   fork branch (their commits + your fixes, PR crediting the author) or a
   comment with cherry-pick instructions; confirm the route with the user.

## Pitfalls

- **Postgres sequences do not roll back** with RefreshDatabase's per-test
  transactions: rows revert, `nextval` does not — factory-seeded lookups
  drift past hardcoded ids and FK violations then name whatever table is
  checked first (often only the first test in a run fails). Seed shared
  lookups with explicit `id = 1` plus
  `setval('<table>_id_seq', GREATEST(COALESCE(MAX(id),1),1))` in the shared
  setUp trait.
- **Random execution order × code that gives a magic id special meaning**
  ("id 1 is headquarters", "id 0 is default") makes tests order-dependent:
  green alone or in some orders, red in others. Detect with the loop run;
  fix by pinning the sequence in setUp so rows start past the magic id —
  never by loosening the assertion.
- **Changing a helper's return shape** (list → assoc array, adding a field):
  enumerate EVERY call style across the test tree — positional destructure,
  named destructure, plain `$x = helper()` — and reconcile your match count
  against the search's total; paginated results hide call sites, and a
  hidden one only fails in CI. Either fix all styles or keep both keys
  working during the transition.
- **A regression guard that passes on first run has proven nothing.** Prove
  its signal by mutating the implementation (make the guarded code a no-op,
  watch the guard fail, revert, watch it pass) before trusting it as a
  refactor tripwire.
- **Assert against data the factory randomizes** (names, coordinates,
  addresses): capture the model you just created and assert its real
  attribute; literals copied from a seed table will flake.
- **Spatial/bbox queries over randomized coordinates**: factory lat/lng
  lands outside the fixed query window and returns "count 0" — pin the
  coordinates inside the window in the test helper at creation time.
