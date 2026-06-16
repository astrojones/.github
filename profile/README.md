# astrojones

**Repos that teach any coding assistant how to work on them.** Every app we build is *born
harnessed* — it carries safe, repo-aware tooling and symbol-level code navigation, so you can
point Claude Code, Codex, or Cursor at it and start shipping like a senior engineer from
minute one.

The public piece of the platform is one Claude Code plugin: [**`astrojones`**](https://github.com/astrojones/astrojones)
— a generic coding harness that works in *any* repo, not just ours.

---

## Quickstart

You need [Claude Code](https://docs.claude.com/en/docs/claude-code) and (for the harness's
code navigation) [`uv`](https://docs.astral.sh/uv/).

```bash
claude plugin marketplace add astrojones/claude-plugins
claude plugin install astrojones@astrojones      # the harness — public, works anywhere
```

That's it. `astrojones` bundles its MCP server and **auto-connects at session start** — open
any repo and it harnesses it on connect (opt out via `AGENTS.md`). For non-MCP clients, run
`/astrojones:harness-init` explicitly.

> **Org members:** also install the deploy layer — `claude plugin install deploy@astrojones`
> — which adds `/new-app`, `/harness-app`, `nuklaut-deploy`, and `deploy-doctor` on top of the
> harness. It's private to the org.

---

## What the harness gives you

Drop any MCP-capable assistant into a harnessed repo and you get:

- A bundled, auto-connecting **MCP server** exposing deterministic `repo_*` tools — overview,
  status, search, precise range reads, impact analysis, diff, change-verification, health —
  instead of raw shell.
- **Serena** proxied through the same server as `serena_*` tools: semantic symbol navigation
  and editing — find definitions, references, and call sites; edit by symbol, not by line.
- Generic workflow **skills** (`bugfix` / `feature` / `refactor` / `test` / `implement` /
  `commit`) and **subagents** (`explorer` / `implementer` / `reviewer` / `test-runner` /
  `fullstack-architect`).
- Safety hooks: destructive commands and secret reads are policy-blocked with an actionable reason.

The discipline it expects: orient with `repo_context_overview`, locate with Serena, read
precise ranges (never dump whole files), check blast radius with `repo_impact_file` before a
cross-file edit, edit small, then run `repo_verify_changed` (narrow lint/typecheck/tests for
what changed — not the whole suite).

---

## Public repos

| Repo | What it is |
|------|-----------|
| [`astrojones`](https://github.com/astrojones/astrojones) | The public coding-harness plugin — the bundled MCP server + `serena_*` navigation, safety hooks, workflow skills/subagents, `/harness-init`. The thing that makes apps born harnessed. |
| [`standards`](https://github.com/astrojones/standards) | Single source of truth for engineering standards (Python tooling, CI/CD). Generated from, never hand-copied. |
| [`.github`](https://github.com/astrojones/.github) | Org-wide reusable GitHub Actions workflows. Apps call these; they don't duplicate them. |
| [`claude-plugins`](https://github.com/astrojones/claude-plugins) | The plugin marketplace index. |

---

*New here? `claude plugin marketplace add astrojones/claude-plugins` then
`claude plugin install astrojones@astrojones`, and point it at any repo.*
