---
name: hermes-plugin-management
description: "Install or enable Hermes Agent plugins via the CLI."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [hermes, plugins, install]
    related_skills: [hermes-agent, mcp-tooling]
---

# Hermes Plugin Management

## When to Use

- User asks to install or enable a Hermes Agent plugin (catalog name, owner/repo, or git URL).
- A plugin's skills or hooks seem missing after install.

## Install

```bash
hermes plugins install <owner/repo | git-url | catalog-name> [--force]
hermes plugins enable <name>
```

- Git-URL installs work even for imperfect Hermes plugins: warning "doesn't contain plugin.yaml..." appears but it still installs to `~/.hermes/plugins/<name>`.
- A security scan (Tirith threat intelligence) may BLOCK community-source installs with "Decision: BLOCKED". Retry with `--force` only when the user asked for that source.
- Portable packages install disabled by default — run `hermes plugins enable <name>` after install.
- Hooks take effect immediately (gateway reloads them); **plugin skills only load on the next session** — tell the user to start a new session (`/new`) before expecting `plugin:skill` entries.

## Verify

- `hermes plugins list` — Status column shows `enabled`, Source `user`.
- `hermes skills list` — plugin skills appear only after a new session.

## Pitfalls

- `hermes plugins search <name>` only covers the curated catalog — a GitHub plugin not listed returns "No catalog entries matched"; install by URL instead.
- Plugin-installed skills are referenced as `<plugin>:<skill>` and are protected: do not patch them in place.
