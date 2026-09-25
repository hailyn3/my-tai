---
name: agent-docs-sync
description: Use when asked to update AGENTS.md after code changes.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [docs, agents-md, maintenance, sync, conventions]
    category: software-development
    related_skills: [improve, improve-workflows, requesting-code-review]
---

# Syncing a Repo's Agent Docs

Bring `AGENTS.md` / `CLAUDE.md` and the reference files it points at back
in line with what the code actually does. The deliverable is a doc a
different agent can act on without re-deriving anything.

## When to Use

- The user asks to update `AGENTS.md` or "sync the docs with the latest
  changes".
- A doc cited a count, path, or behavior that turned out to be wrong.
- A batch of commits landed since the doc's own last commit message.
- Onboarding a repo whose agent instructions have drifted from the code.

## Procedure

1. **Find the last doc commit, then diff the world against it:**
   ```bash
   git log -3 --format='%h %ad %s' --date=short -- AGENTS.md
   git log --oneline <last-doc-commit>..HEAD
   ```
   Classify each commit: behavior change (document), refactor/no-op
   (skip), test or CI change (update counts and commands).
2. **Verify every claim against the artifact before writing it.** Read
   the route file for paths, the schema/migrations for columns, the job
   class for timeout/tries, and *run* the suites for counts. Never carry
   an old number forward and never write a test count you did not
   produce this session.
3. **Sweep the sibling files in the same pass:**
   ```bash
   grep -n 'references/' AGENTS.md
   ```
   Anything you change in the umbrella doc must be checked in every file
   it points at — the umbrella holds rules and gotchas, the references
   hold endpoints, scheduler tables, schema detail. Updating one side
   only leaves the two contradicting each other, which is worse than a
   stale doc because both look authoritative.
4. **Keep the split:** long tables and step-by-step detail belong in the
   reference file; the umbrella keeps one-line pointers. Do not copy the
   same table into both.
5. **Sanity-check the markdown you touched:**
   ```bash
   awk -F'|' '/^\|/{print NF-1}' AGENTS.md | sort | uniq -c   # consistent pipe counts per table
   grep -rn '<numbers you replaced>' AGENTS.md references/     # no stale figures left
   ```
   Confirm heading order still makes sense and no section lost its
   table delimiter during a patch.
6. **Commit and push on the current branch** if the repo convention
   says every change ships that way.

## Pitfalls

- **Grep every figure you are about to repeat** (counts, limits, timeouts,
  dates) instead of trusting the existing prose — doc claims drift
  independently of each other, so a spotlessly updated section can still
  cite a number two sections away that nobody touched.
- **Verify route and middleware names against the route file**, not from
  memory: prefixes and group nesting decide the real path, and a table
  entry that names a plausible-but-wrong path sends an executor down a
  dead end.
- **Do not transcribe transient infra failures as project rules.** A
  blip in a local database, a missing binary, or an unconfigured
  credential belongs to the machine, not the doc; document only the
  durable behavior and its cause.
- **Record behavior with its trigger, not just its value.** "Set
  APP_LOCALE=fa" is unactionable the moment the template changes; "the
  example env omits the locale, the app then falls back to `en`, and
  every localized-text assertion fails" survives a rewrite of the file.
