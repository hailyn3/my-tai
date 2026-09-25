---
name: pull-request-verification
description: "Use when asked to verify a pull request actually works."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [pull-request, review, ci, verification, deploy]
    category: software-development
    related_skills: [laravel-pest-testing, playwright-e2e, git-workflow, github]
---

# Verifying a Pull Request

Establish that a PR does what it claims AND that it runs — with evidence you
produced, not evidence you read. The output is a verdict split into
verified-by-execution vs verified-by-reading, plus an explicit list of what
was not checked.

## When to Use

- The user asks whether a PR (theirs or another author's) is correct,
  "really works", or safe to merge.
- Deciding whether to approve/merge after CI is green.
- A PR body claims test counts and you need to confirm them.

## Standing Rules

- **Never merge unless the user explicitly asks.** Verifying stops at the
  verdict; merge is a separate instruction.
- **Never check out the PR's branch in your working repo.** Verify in a
  detached scratch worktree so your branch, index, and untracked files stay
  untouched (recipe: `laravel-pest-testing`, procedure step 6).
- A PR body's "Verification: N passed" section is a **claim**. Reproduce the
  headline numbers yourself or cite the CI job URL that produced them —
  never repeat the claim as fact.

## Procedure

1. **Read the PR and its CI.** Fetch metadata (title, body, head/base SHA,
   mergeable state), then the checks actually run on the head SHA:
   `gh pr checks <N> --repo <owner>/<repo>` or
   `gh pr view <N> --repo <owner>/<repo> --json statusCheckRollup,mergeable,mergeStateStatus`.
   Note which required jobs passed and how long they took.
2. **Fetch the head from the remote that HOSTS the PR:**
   `git fetch <hosting-remote> pull/<N>/head:<tmp-branch>`. A fork's
   `origin` does not carry the base repo's `refs/pull/*` — fetching the PR
   ref from the wrong remote fails with `couldn't find remote ref`.
3. **Review the diff for wiring, not just style.** Concentrate on what can
   pass tests and still break production:
   - Data moved from hardcoded arrays/consts into a table → verify the
     seeder copies the old values **programmatically** (parse both sides and
     compare); a hand-retyped dump is where IDs get corrupted.
   - New route → middleware on the route *and* an authorization call inside
     every mutating action.
   - New cache namespace → registered in whatever prune/reset list the repo
     uses, or stale entries accumulate forever.
   - PR-body statements about other files ("granted to admin via
     RoleSeeder") → confirm the file actually changed, or that an existing
     mechanism (e.g. `syncPermissions(Permission::all())`) covers it.
4. **Execute against the ref.** Run the repo's gates and the new tests in
   the worktree; reproduce the claimed counts. For UI work also run the new
   e2e specs (recipe: `playwright-e2e`). Mismatched counts are findings.
5. **Check the deploy assumptions — tests never exercise the pipeline.**
   - Migration + seeder split: does the deploy job run `db:seed`? A
     `migrate`-only deploy leaves the new table empty (pages render empty
     states) and a new permission absent.
   - Unknown permission in middleware: it resolves **false**, not an error
     — everyone including admins gets 403 until the permission seeder runs.
     Flag as a merge/deploy blocker when the pipeline does not seed.
   - Frontend build artifacts: does CI or deploy run `npm run build` when
     Blade views gained new markup?
6. **Report honestly.** Three buckets: (a) executed and green — with the
   command and counts, (b) verified by reading the diff — with file:line,
   (c) not checked. Never fold (c) into (a).
7. **Clean up:** `git worktree remove --force`, delete the temp branch,
   kill any server the e2e script left on its port.

## Pitfalls

- **The commit *status* API shows `pending` / 0 statuses for GitHub Actions
  workflows** — Actions report as check runs, not statuses. A tool returning
  `statuses: []` has not proven CI is missing; read `statusCheckRollup` or
  `gh pr checks` before concluding anything.
- **`mergeable: MERGEABLE` and a clean `mergeStateStatus` are merge-conflict
  math only.** They say nothing about correctness; never present them as
  quality signals.
- **Verifying only the new test file proves the change in isolation.** Run
  the full suite too — shared seeders, permissions, and cache namespaces
  regress suites far from the new code.
- **Comparing data arrays by eye misses one-row drift.** Diff counts and
  tuple-by-tuple comparison in a script; the totals can match while a row's
  ID differs.
- **A gate that returns in a fraction of a second with empty output never
  ran.** Exit 0 is not evidence when the tool printed nothing — the
  invocation silently no-oped (wrong runner for that binary). Demand
  positive evidence from every gate: the test-count line, the linter's
  result line, the analyzer's "No errors"; invoke PHP tools as
  `php vendor/bin/<tool>` and assert on their output, not on exit status.
- **Failures caused by the scratch worktree's missing untracked artifacts
  (built assets, autoload map, env file) are environment, not PR defects.**
  Repair the worktree per the `laravel-pest-testing` worktree recipe and
  re-run before reporting anything against the change — an unfixed worktree
  turns a manifest/autoload gap into a false "this PR breaks tests" verdict.
