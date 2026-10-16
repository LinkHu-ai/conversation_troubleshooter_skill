# Conversation Troubleshooter Skill

Conversation Troubleshooter is a read-only Agent Skill for investigating authorized AI-support behavior across logs, tenant-scoped databases, Sentry incidents, and local-to-remote dependency code. It also defines bounded, validated data-export workflows.

This repository contains only the portable Skill and marketplace metadata. It does not contain the MCP service implementation, upstream credentials, tokens, private keys, or environment files.

## Environments

Supported runtime environments: `dev` (`devnet`), `chess` (`testnet`), `uat` and `production` (`main`).
Dev queries require an MCP server version that accepts `dev`, grants the account `env:dev`, and
configures isolated dev data sources. Updating this skill alone does not upgrade or deploy the MCP
server. An unconfigured dev database must fail explicitly rather than use production data.

## Prerequisite

The Skill expects an authenticated MCP connection named `conversation_troubleshooter`. The MCP service owns all log-platform, database, Sentry, GitHub, and shared-host credentials. Installing this repository does not grant access to those systems.

The OpenAI Skill metadata declares the public MCP endpoint used by the current deployment. Hosts that do not consume `agents/openai.yaml` must configure the MCP connection separately.

## Repository layout

```text
.
├── .agents/plugins/marketplace.json
├── .cursor-plugin/marketplace.json
└── plugins/conversation-troubleshooter/
    ├── plugin.json
    └── skills/conversation-troubleshooter/
        ├── SKILL.md
        ├── agents/openai.yaml
        └── references/
```

## Install in Codex

Add the Git-backed marketplace and install the plugin:

```bash
codex plugin marketplace add LinkHu-ai/conversation_troubleshooter_skill --ref main
codex plugin add conversation-troubleshooter@synergyai-tools
```

Start a new Codex task after installation so the new Skill is discovered.

To refresh a later release:

```bash
codex plugin marketplace upgrade synergyai-tools
codex plugin add conversation-troubleshooter@synergyai-tools
```

## Install in Cursor

In Cursor, open **Customize**, choose **From GitHub Repository**, and enter:

```text
https://github.com/LinkHu-ai/conversation_troubleshooter_skill
```

Install `conversation-troubleshooter` at user or project scope, then start a new Agent session.

## Cross-agent command-line installation

The third-party `skills` CLI can install the Skill directly into Codex or Cursor:

```bash
npx skills add LinkHu-ai/conversation_troubleshooter_skill \
  --skill conversation-troubleshooter \
  --agent codex cursor \
  --global \
  --yes
```

Review third-party installer behavior and telemetry policy before using it in managed environments.

## Security model

- The Skill is read-only and requires server-enforced authorization.
- Production queries remain bounded by environment, tenant, timezone, and explicit time ranges.
- The Skill never asks users to paste upstream credentials into a conversation.
- Export resources remain owner-bound, encrypted at rest, and time-limited by the MCP service.
- A visible tool or installed Skill does not grant permission; the MCP service enforces role, environment, repository, and data-source scopes.

## Development validation

Validate the Skill directory before publishing an update:

```bash
python3 /path/to/skill-creator/scripts/quick_validate.py \
  plugins/conversation-troubleshooter/skills/conversation-troubleshooter
```

Keep the plugin version in `plugin.json` and both marketplace manifests synchronized for each release.
