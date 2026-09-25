---
name: agent-doc-maintenance
description: Keep a repo's AGENTS.md-style agent docs in sync with code.
version: 0.1.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [documentation, agents-md, drift, repo-conventions, maintenance]
    related_skills: [read-the-damn-docs, codebase-inspection]
---

# Maintaining Repo Agent Documentation

Refresh a repository's agent-instruction doc (`AGENTS.md`, `CLAUDE.md`,
`GEMINI.md`) so every claim matches the code as it stands, without
reorganizing the maintainers' structure. The doc is authoritative project
context that every future session loads — stale rules cost more than none.

## When to Use

- The user asks to "update AGENTS.md" / refresh the agent doc.
- You noticed the doc contradict the code while working (wrong command,
  removed symbol, changed schedule, outdated test counts).
- After a large merge or a refactor that touched documented surfaces.
- Don't use for: writing code docs (README/API/ADR), or authoring skills
  (different format, different rules).

## Procedure

1. **Read the doc's own division of labor first.** Agent docs usually open
   with a note saying which detail lives in `references/` (or `docs/`).
   Treat that split as a contract: rules, gotchas, conventions, and
   one-line pointers stay in the root; tables, endpoint inventories,
   command details, and long explanations belong in the reference files.
   *Done when* you can name which of the two files each edit belongs in.
2. **Bound the review to the drift.** Find the last doc commit
   (`git log -1 --format=%h -- AGENTS.md`), then list code commits since
   (`git log --oneline <that-sha>..HEAD -- .`). That list is your scope —
   review it, don't audit the whole repo.
3. **Verify every claim you intend to write.** Code facts from the code
   (`search_files` / CodeGraph / the route file), framework semantics from
   current official docs or vendor source, DB schema from
   `information_schema`, and any count or date from a run you performed
   this session. *Done when* each new sentence has a source you actually
   opened.
4. **Grep the reference files for the same facts before editing the
   root.** A fact stated twice must be fixed twice: schedules, test
   counts, endpoint tables, and FK lists almost always appear in both the
   root and `references/`.
5. **Patch in place, preserving voice.** Keep existing section order,
   table shapes, `>` note blocks, and heading levels. Add a dated review
   line under the doc's own review history rather than rewriting it.
   *Done when* the diff touches only changed facts, not surrounding prose.
6. **Update both sides of every stale fact** — the root's summary and the
   reference's detail — including rows the fact lives in (a scheduler
   table row, a failure-table entry).
7. **Ship it:** check markdown table integrity (consistent column count
   per row), grep all doc files for the old numbers/symbols you replaced
   so no stale copy survives, then commit with a docs-scoped message and
   push to the current branch.

## Pitfalls

- **Fixing only the root doc leaves the stale duplicate.** The reference
  file is where the detail lives, so the root ends up correct while the
  file people actually read stays wrong — always grep `references/` for
  the fact you changed.
- **Docs restate the code; the code wins.** When a documented rule and
  the implementation disagree, verify against the code and an authoritative
  source before editing either — never reconcile by picking the sentence
  you remember.
- **Framework semantics from memory are the most likely wrong claim.**
  Middleware aliases, config keys, and deprecations shift by version —
  resolve them against current docs or the installed vendor source and
  write down which one you used.
- **Never publish a count or date you did not produce.** Test totals,
  spec counts, and migration counts are state; stamp them "verified
  <today>" from a run in this session, or leave the old value and flag it.
- **Don't flatten a deliberate split.** If the maintainers moved detail
  out of the root to keep it lean, adding it back "for completeness"
  reverses their decision — extend the reference instead.
- **Never paste `.env` values or tokens into docs.** Reference key names
  and which file they live in; a doc is committed and public.
- **A doc edit is a code change for workflow purposes** — it gets
  committed and pushed to the current branch like any other, and never
  pushed straight to the shared base branch.

## Verification

- [ ] Every new/changed claim traced to code, docs, schema, or a run.
- [ ] Root doc and every `references/` file agree on the touched facts.
- [ ] Grep finds zero occurrences of the numbers/symbols you replaced.
- [ ] Markdown tables have consistent column counts.
- [ ] Diff is factual edits only — no reflow, no reordering, no secrets.
- [ ] Committed and pushed to the current branch.
