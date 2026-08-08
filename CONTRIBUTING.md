# Contributing

## How to propose changes

Open an issue or a pull request against `main`. Changes to agent methodology should explain the audit-behavior rationale in the PR description.

## How to propose a new agent

This marketplace follows a **one-plugin-per-agent** policy: each agent ships as its own plugin directory `plugins/<agent-name>/` with its own `.claude-plugin/plugin.json` and `agents/` dir, and the installed agent surfaces as `<agent-name>:<agent-name>`. New agents must be self-contained under their plugin directory — no cross-plugin file references, because install caches are copied per-plugin.

## Versioning rule (mandatory)

Any change to plugin content MUST bump that plugin's `plugin.json` `version` — otherwise installed users stay on stale cached content indefinitely (plugin updates only flow on a version change). Patch/minor/major is at the author's judgment; the bump itself is non-negotiable.

## Rulesets note

AlphaFiTech's standard branch rulesets are intentionally not applied to this repo this cycle; applying them is a separate, future decision.
