---
name: mcp-tooling
description: "Use when verifying or troubleshooting Hermes MCP servers."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [mcp, tooling, troubleshooting, session-start]
    related_skills: [read-the-damn-docs]
---

# MCP Tooling

## When to Use

- Starting a session whose task depends on MCP tools (Laravel Boost, Context7, GitHub, CodeGraph) — verify before relying.
- Any MCP call errors, a server is missing from `tool_search` results, or codegraph reports no index.

Session-start rule: when a task needs MCP tools (Laravel Boost, Context7, GitHub, CodeGraph), prove each server works with a real call before claiming it does — a configured server in `~/.hermes/config.yaml` is not evidence of a running one.

## Verification procedure

1. **Discover:** `tool_search` with any broad query — the response's `available_sources` lists every connected server with its tool count. A server absent from that list failed to start (its config may still look fine).
2. **Probe** each server the task needs with one cheap read call:
   - `laravel_boost` → `application_info`
   - `context7` → `resolve_library_id` (any library)
   - `github` → `search_repositories` (read-only; do not probe with a write)
   - `codegraph` → `codegraph_explore`, or CLI `codegraph status .` in the project root
3. Load schemas with `tool_describe` before invoking (`tool_search` → `tool_describe` → `tool_call`).

## Pitfalls

- **Issue MCP calls one per `tool_call` invocation.** A batch stacking or mixing non-connector entries is rejected before execution — parallel batching only works for `connector__*` names, otherwise the whole batch errors and nothing runs.
- **A call that fails with "lost its stdio subprocess" is retryable, not fatal.** Diagnose first: run the server's command from `~/.hermes/config.yaml` manually and pipe one JSON-RPC `initialize` line into it — if it returns `serverInfo`, the server is healthy and the next tool call (Hermes restarts the subprocess) will succeed. One retry, then escalate to real troubleshooting.
- **Smoke-test any stdio server the same way:** `printf '%s\n' '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"smoke","version":"1"}}}' | <server command>` — expect a JSON result with `serverInfo`; this separates "server broken" from "transport/session problem".
- **CodeGraph state is per-machine and per-project.** Reinstall with `npm i -g @colbymchenry/codegraph`, then in the repo run `codegraph init` + `codegraph sync`; `codegraph status .` must report the index up to date before trusting results. A "Not initialized" index is as broken as a missing binary.
- **Servers fixed mid-session expose tools only next session** (the tool catalog is built at startup). Use the CLI in the meantime — e.g. `codegraph explore "<question>"` — rather than waiting or claiming the tool is unavailable.
- **GitHub MCP writes fail with "Requires authentication" when the token
  reached the server as an unexpanded placeholder** — `--args` values are
  stored verbatim and never shell-expanded, so a `${GH_TOKEN}` inside args
  arrives literal and every write 401s while public reads still work (reads
  don't need auth). Fix by re-adding the server with the secret passed as
  env, which your shell expands before Hermes writes the config:
  `printf 'y\ny\n' | hermes mcp add github --command npx --env "GITHUB_PERSONAL_ACCESS_TOKEN=$GH_TOKEN" --args -y @modelcontextprotocol/server-github`
  — re-adding an existing name prompts twice (overwrite?, then
  enable-tools?), hence the piped answers; `--args` must stay the last
  option. Write tools only appear **next session** (catalog is built at
  startup), so use the `gh` CLI for writes until then.
- **Verify a secret by an HTTP status code, never by reading it back** —
  terminal output redacts token-shaped strings, so a config dump "shows"
  nothing usable: `curl -o /dev/null -w '%{http_code}' -H "Authorization:
  Bearer $TOKEN" <whoami-endpoint>` → 200 proves the value is live.

## Usage order

Follow the project's AGENTS.md debugging checklist (codegraph → boost → context7 → tinker → pest). Prefer boost `database-query` / `database-schema` / `search-docs` over hand-written SQL or shell equivalents; resolve a Context7 library id before querying docs (max 3 resolve calls per question); use CodeGraph before grep/Read for structural questions about indexed code.