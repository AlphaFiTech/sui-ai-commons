# Contributing

## How to propose changes

Open an issue or a pull request against `main`. Changes to agent methodology should explain the audit-behavior rationale in the PR description.

## How to propose a new agent

Every agent is added **inside the single umbrella plugin** `plugins/open-agents/`: one `agents/<agent>.md` definition plus an optional `agents/<agent>/` companion directory — never as a new `plugins/<agent>/` directory. The installed agent surfaces as `open-agents:<agent>`. Agents must be self-contained under the plugin directory (install cache copies exclude traversal targets), and any companion-dir reference in agent content must be `${CLAUDE_PLUGIN_ROOT}`-prefixed (`${CLAUDE_PLUGIN_ROOT}/agents/<agent>/…` — the pattern the auditor uses).

## Versioning rule (mandatory)

Any change to plugin content — **any agent's** — MUST bump the single umbrella `plugin.json` `version` (`plugins/open-agents/.claude-plugin/plugin.json`) — otherwise installed users stay on stale cached content indefinitely (plugin updates only flow on a version change). Patch/minor/major is at the author's judgment; the bump itself is non-negotiable. A consequence of the umbrella: a bump ships all agents together — there is no per-agent versioning. Likewise mandatory: the `renames` map in `.claude-plugin/marketplace.json` is append-only history and must never be pruned — future plugin renames extend the map.

## Rulesets note

AlphaFiTech's standard branch rulesets are intentionally not applied to this repo this cycle; applying them is a separate, future decision.
