---
name: read-the-damn-docs
description: "Use when implementing third-party APIs. Read docs first."
---

# Read The Damn Docs

Do not guess where authoritative docs can answer the question. The most common right move is to web-search for the current official docs, open the relevant pages, and read them before coding. For APIs, versions, provider behavior, config, limits, lifecycle hooks, or security-sensitive flows, ground the answer in what the docs actually say.

## Docs-First Triggers

Read docs before proceeding when any of these are true:

- The user asks for "latest", "current", "official", "supported", "best practice", "recommended", "today", "now", or "look it up".
- The needed docs are not already in the repo or supplied by the user. Search the web for the official docs rather than hoping model memory is current.
- The task adds, upgrades, configures, or imports a package, SDK, framework, plugin, CLI, model, cloud resource, or provider integration.
- The API is fast-moving or version-sensitive: AI SDKs, OpenAI/Anthropic/Google APIs, Next.js, React, Tailwind, Vite, Nitro, Drizzle, Prisma, Stripe, GitHub, Slack, Notion, browser APIs, deployment platforms, auth libraries.
- The implementation depends on auth, OAuth scopes, permissions, secrets, webhooks, billing, payments, PII, encryption, data retention, migrations, retries, rate limits, quotas, caching, deploys, or compliance.
- An error mentions deprecation, unknown options, missing exports, invalid config, unsupported fields, changed defaults, or version mismatch.
- A repo has local docs, ADRs, generated schemas, OpenAPI specs, route/action registries, design-system docs, or package-level READMEs that could define the contract.
- The choice is expensive to reverse: public wire formats, database schema, migration strategy, persistent IDs, event names, customer-visible behavior, or external automation contracts.
- You catch yourself about to write "usually", "probably", "I think", "from memory", or code copied from model memory for an external API.

## What Counts As Docs

Use the most authoritative source available:

- Local repo docs, specs, ADRs, schemas, generated types, package READMEs, and tests for project-specific behavior.
- Official product docs, API references, migration guides, changelogs, release notes, and SDK source/types for third-party behavior. Find these with web search when you do not already have the exact URL.
- Package registry metadata for versions. Before adding a dependency, verify the latest version and major release line.

## Workflow

1. **Identify the authoritative source.** For the specific package/feature/API at the exact version in use, what is the canonical reference?
2. **Read it.** Use `web_search`, `web_extract`, or local file reads to open the primary docs. For APIs, read the reference, not just the tutorial.
3. **Check version alignment.** The repo may use a different version than the latest. Compare what the docs say against the installed version.
4. **Extract the few facts needed for the task:** option names, imports, lifecycle rules, default behavior, breaking changes, limits, permissions, and examples for the current major version.
5. **Apply and verify.** Use the facts to guide implementation. If a docs example exists, adapt it rather than inventing structure.

## When A Quick Local Read Is Enough

Do not browse the web for every tiny edit. A docs pass can be local and brief when:

- The repo already has the authoritative docs or ADR on file.
- The package version and behavior are well-known and stable.
- The change is clearly within existing project conventions and does not touch external surfaces.

## Anti-patterns

- Assuming an API shape from memory without verifying the current docs.
- Copying old code examples without checking the target SDK/framework version.
- Treating a blog post, StackOverflow answer, or LLM snippet as authoritative when official docs exist.
- Using a library feature marked as experimental or unstable without noting the risk.
- Skipping local ADRs or README conventions that override upstream defaults.