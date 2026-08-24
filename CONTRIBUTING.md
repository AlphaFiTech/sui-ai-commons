# Contributing

## How to propose changes

Open an issue or a pull request against `main`. Changes to agent methodology should explain the audit-behavior rationale in the PR description.

## How to propose a new agent

Every agent is added **inside the single umbrella plugin** `plugins/sui-ai/`: one `agents/<agent>.md` definition plus an optional `agents/<agent>/` companion directory — never as a new `plugins/<agent>/` directory; components of **any** type go inside the same umbrella. The installed agent surfaces as `sui-ai:<agent>`. Agents must be self-contained under the plugin directory (install cache copies exclude traversal targets), and any companion-dir reference in agent content must be `${CLAUDE_PLUGIN_ROOT}`-prefixed (`${CLAUDE_PLUGIN_ROOT}/agents/<agent>/…` — the pattern the auditor uses).

## Where future skills and tools live

Nothing beyond agents ships this cycle; these are the conventions future skills and tools must follow.

- **Skills:** `skills/<skill-name>/SKILL.md` at the plugin root (optional companion files — `reference.md`, `scripts/`, … — beside it). Skills are auto-discovered on install and surface namespaced as `sui-ai:<skill-name>` (the `SKILL.md` frontmatter `name` controls the invocation name; the directory basename is the fallback).
- **Tools:** "tools" is not a plugin component type of its own — it means MCP servers declared in `.mcp.json` at the plugin root (their tools surface as `mcp__plugin_sui-ai_<server>__<tool>`) and/or executables under `bin/` (added to the Bash tool's PATH as bare commands while the plugin is enabled).
- **`commands/`:** a legacy alias this repo does **not** use — new work goes in `skills/`.
- **Placement:** every component directory sits at the **plugin root** (`plugins/sui-ai/skills/`, `plugins/sui-ai/agents/`, …), never inside `.claude-plugin/` (which holds only `plugin.json`). One warning: the manifest `skills` field *adds to* the default `skills/` scan while the `commands`/`agents` manifest fields *replace* their defaults — so prefer the default directories.
- **Versioning:** the single umbrella version-bump rule below applies to **every** component type — one plugin, one version, one `sui-ai:` prefix.

## Versioning rule (mandatory)

Any change to plugin content — any agent's, and in future any skill's or tool's — MUST bump the single umbrella `plugin.json` `version` (`plugins/sui-ai/.claude-plugin/plugin.json`) — otherwise installed users stay on stale cached content indefinitely (plugin updates only flow on a version change). Patch/minor/major is at the author's judgment; the bump itself is non-negotiable. A consequence of the umbrella: a bump ships all components together — there is no per-component versioning. Likewise mandatory: the `renames` map in `.claude-plugin/marketplace.json` is append-only history and must never be pruned — future plugin renames extend the chain.

## Maintainer operations note

This repository was renamed from `AlphaFiTech/open-agents` to `AlphaFiTech/sui-ai-commons` (and the umbrella plugin `open-agents` → `sui-ai` — recorded in the append-only `renames` map above). A repository named `AlphaFiTech/open-agents` must **never be created**: GitHub's rename redirect — which keeps old links and any legacy plugin registrations working — dies the moment the old name is reused.

## Rulesets note

AlphaFiTech's standard branch rulesets are applied to this repository (since 2026-08-18): changes to `main` go through a pull request, and new branches must use a `feature/`, `bugfix/`, `hotfix/`, or `chore/` prefix. Open PRs from a prefix-compliant branch; maintainers merge after review (repository admins may complete a merge when no second reviewer is available).
