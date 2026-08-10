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

## Provenance & licensing

The `sui-move-auditor` agent and its six companion playbooks were originally developed in [`jangid/tools-skills-agents`](https://github.com/jangid/tools-skills-agents); **`AlphaFiTech/sui-ai-commons` (formerly `AlphaFiTech/open-agents`) is now the canonical home**. All ported content was solely authored by the repo owner `jangid` (Pankaj Jangid), as evidenced by the `jangid/tools-skills-agents` commit history for the ported paths (`agents/sui-move-auditor.md`, `agents/sui-move-auditor/`). The source repo carried no license, so the sole author's relicensing under MIT here is unencumbered. The LICENSE copyright line `Copyright (c) 2026 Pankaj Jangid (AlphaFiTech)` names the author as the sole copyright holder, publishing under the org's canonical home — the parenthetical marks the publishing venue without assigning copyright to the org.
