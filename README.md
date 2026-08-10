# sui-ai-commons

## What this is

The **Sui community's AI commons**: a public [Claude Code](https://claude.com/claude-code) plugin marketplace of AI agents, skills, and tools for Sui development, served as the single `sui-ai` umbrella plugin. Agents ship today; skills and tools are future additions under the documented scaffold (see [CONTRIBUTING.md](CONTRIBUTING.md)). Utilities and other non-AI tooling are explicitly out of scope — they live in a separate, future repo.

## Available agents

All components ship inside the single umbrella plugin `sui-ai` and surface under the `sui-ai:` prefix.

| Agent | Description | Scoped agent name |
|---|---|---|
| `sui-move-auditor` | Zero-trust adversarial security auditor for Sui Move smart contracts | `sui-ai:sui-move-auditor` |

## Installation

From any Claude Code session:

```
/plugin marketplace add AlphaFiTech/sui-ai-commons
/plugin install sui-ai@sui-ai-commons
```

## Usage

Once installed, invoke the `sui-ai:sui-move-auditor` agent — via the agent picker or by asking Claude Code to use it — from a session whose working directory is the Sui Move project under audit. The agent audits the *target* project's sources (`sources/*.move`, its docs) and loads its own domain playbooks on demand. Typical uses: pre-deployment reviews, PR security audits, and DeFi protocol assessments.

## Migrating from the old `open-agents` (or `sui-move-auditor`) plugin

This repository was renamed from `AlphaFiTech/open-agents` to `AlphaFiTech/sui-ai-commons`, and the umbrella plugin was renamed `open-agents` → `sui-ai` (v2.0.0), so installed agent ids changed to `sui-ai:*`. If you added the marketplace under the `open-agents` name, migrate with either path below.

### Path 1 — clean (recommended)

> **Warning:** removing a marketplace **uninstalls the plugins installed from it**. That is expected here — the install two lines later restores the auditor.

```
/plugin marketplace remove open-agents
/plugin marketplace add AlphaFiTech/sui-ai-commons
/plugin install sui-ai@sui-ai-commons
```

Your end state matches a fresh install exactly (`sui-ai@sui-ai-commons`).

### Path 2 — do-nothing / minimal

GitHub's repository redirect keeps `/plugin marketplace update` working under your legacy `open-agents` registration. On Claude Code >= 2.1.193 the marketplace's `renames` map rewrites your settings automatically — transitively even from the oldest `sui-move-auditor` install — and a single

```
/plugin install sui-ai@open-agents
```

closes the one-time `plugin-cache-miss`. Fully functional, but your local marketplace keeps the legacy `open-agents` name indefinitely.

Whichever path you take, start a new session afterwards and confirm the agent roster shows `sui-ai:sui-move-auditor`.

Clients older than Claude Code 2.1.193 ignore the `renames` map — they see `plugin-not-found`; use the clean path instead.

**Maintainer rule (permanent):** a repository named `AlphaFiTech/open-agents` must **never be recreated** — a new repo under the old name kills the redirect that keeps un-migrated users updating.

## Provenance & licensing

The `sui-move-auditor` agent and its six companion playbooks were originally developed in [`jangid/tools-skills-agents`](https://github.com/jangid/tools-skills-agents); **`AlphaFiTech/sui-ai-commons` (formerly `AlphaFiTech/open-agents`) is now the canonical home**. All ported content was solely authored by the repo owner `jangid` (Pankaj Jangid), as evidenced by the `jangid/tools-skills-agents` commit history for the ported paths (`agents/sui-move-auditor.md`, `agents/sui-move-auditor/`). The source repo carried no license, so the sole author's relicensing under MIT here is unencumbered. The LICENSE copyright line `Copyright (c) 2026 Pankaj Jangid (AlphaFiTech)` names the author as the sole copyright holder, publishing under the org's canonical home — the parenthetical marks the publishing venue without assigning copyright to the org.
