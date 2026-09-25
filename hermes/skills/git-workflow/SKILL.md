---
name: git-workflow
description: "Git pitfalls: nested repos, staging, identity."
version: 1.0.0
author: Sydney
license: MIT
metadata:
  hermes:
    tags: [git, workflow, pitfalls, staging, nested-repos, commit]
    category: software-development
---

# Git Workflow

Pitfalls and procedures for everyday git operations that fall outside standard
`gh` CLI workflows (for gh-specific flows see the `github` skill).

## Standing Rules

- Always check `git status` and `git remote -v` before staging or pushing.
- Configure per-repo identity before first commit if global config is absent.
- Never assume a branch name — read it from `git branch --show-current`.
- Ahead/behind counts in `git status -sb` are measured against the *configured*
  upstream, which may be a different branch than the one you push to. When the
  tracking ref looks wrong, compare explicitly:
  `git rev-list --left-right --count <branch>...origin/<branch>`.

## Pitfalls

### Nested `.git` directories when copying content

When `cp -r` (or similar) a directory that contains its own `.git` into another repo,
`git add` detects the nested `.git` and stages only a submodule reference (mode 160000),
not the actual files. Clones of the outer repo will not contain the copied content.

**Fix — remove nested `.git` before staging:**

```bash
cp -r /source/dir target/inside/repo
rm -rf target/inside/repo/.git
git add target/inside/repo/
```

If already staged as submodule:

```bash
git rm -r --cached target/inside/repo/
rm -rf target/inside/repo/.git
git add target/inside/repo/
git commit -m "Add dir contents (fix nested repo)"
```

### Missing git identity on fresh clones

New clones may lack both global and per-repo `user.name`/`user.email`.
Commits fail with `empty ident name`.

**Fix — set per-repo config before first commit:**

```bash
git config user.email "user@users.noreply.github.com"
git config user.name "username"
```

Detect from `gh auth status` or set manually.

### Fetching an upstream branch without touching remotes

To compare or sync a branch from another repository when the local remote
configuration is owned by tooling (or must stay unchanged), fetch by URL into
a temporary ref instead of adding a remote:

```bash
git fetch <url> <branch>:refs/tmp/<branch>
git merge-base --is-ancestor refs/tmp/<branch> <local-branch> && echo in-sync
git update-ref -d refs/tmp/<branch>   # clean up when done
```

Temporary refs are not part of `git remote` config, so nothing about the
repository's remote setup changes.

### Catching up a branch that is behind its remote counterpart

Pushes get rejected with `tip of your current branch is behind its remote
counterpart`. Sync with a fast-forward, never `git pull` — a pull can create an
unnecessary merge commit and drags remote changes into your history silently.

```bash
git fetch origin --prune
git rev-list --left-right --count <branch>...origin/<branch>  # 0 <n> => behind only
git stash push -u -m wip -- <dirty files>   # only if the tree is dirty
git merge --ff-only origin/<branch>
git stash pop
git push origin <branch>
```

- Stash first: `--ff-only` refuses to overwrite locally modified files, so the
  dirty tree must be out of the way before the checkout.
- `0 <n>` (no commits of your own) means a clean fast-forward is possible;
  anything on the left side means real divergence — stop and ask rather than
  rebasing/merging unprompted.
- Confirm the result against the ref you actually push to (`origin/<same-name>`),
  not the tracking upstream, before declaring the branch synced.

## Verification

- `git status` shows no unexpected submodule entries.
- `git diff --cached --stat` shows actual file additions, not just mode changes.
- Commit and push succeed without warnings about embedded repos.
