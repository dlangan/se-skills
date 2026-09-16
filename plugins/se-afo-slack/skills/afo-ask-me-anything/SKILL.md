---
name: afo-ask-me-anything
description: "AFO/Regrello specialized advisor for technical and pre-sales questions. Searches authoritative Slack channels (#regrello-afo-ranger-station, #help-sell-agentforce-operations) and the Regrello How-To canvas. Classifies questions as Technical, Pre-Sales, or Hybrid and routes to the right source. Source: Slack canvas (see Configuration)."
metadata:
  type: slack-skill
  version: "1.1"
  source_canvas: "<AFO_HOWTO_CANVAS_ID>"
  author: "AFO Enablement Team"
---

# AFO Regrello Answers Agent: Ask Me Anything

This is a Slack Skill (run via Slackbot DM). It's a single source of truth for any AFO/Regrello question — technical or commercial.

> **Workspace-scoped skill.** This skill only functions inside the original Slack workspace where its canvas and channels live. The canvas and channel IDs below are not portable — if you install this plugin into a different Slack workspace, you must recreate (or re-pin) the equivalent canvas and channels there and fill in the placeholders in Configuration before this skill can resolve anything.

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `config.slack_team_id` | Slack workspace (team) ID this skill's canvas and channels live in. |

The values below are **not** covered by a canonical config key (this skill's canvas isn't the shared `todo_canvas_id`, and there is no canonical key for individual channel IDs). Per de-personalization policy, they are documented here — flagged, not invented — as required workspace-specific placeholders the installer must set:

| Placeholder | Purpose | Original workspace value (reference only) |
|---|---|---|
| `<AFO_HOWTO_CANVAS_ID>` | Pinned "How to Instructions" canvas — primary source for how-to/technical questions | `<AFO_HOWTO_CANVAS_ID>` |
| `<RANGER_STATION_CHANNEL_ID>` | #regrello-afo-ranger-station — Technical source | `<RANGER_STATION_CHANNEL_ID>` |
| `<SELL_AFO_CHANNEL_ID>` | #help-sell-agentforce-operations — Pre-Sales source | `<SELL_AFO_CHANNEL_ID>` |
| `<AGENT_HELP_CHANNEL_ID>` | #regr-ai-agent-help — Agent-specific troubleshooting | `<AGENT_HELP_CHANNEL_ID>` |
| `<SUPPLY_CHAIN_STORIES_CHANNEL_ID>` | #help-sell-agentforce-supply-chain — Customer stories/references | `<SUPPLY_CHAIN_STORIES_CHANNEL_ID>` |

## How to Use

1. Open Slackbot DM
2. Add this skill (canvas `<AFO_HOWTO_CANVAS_ID>`)
3. Ask any question about AFO/Regrello

## Question Classification

The skill routes to different primary sources based on question type:

- **Technical** (Blueprint/Workflow/Task/Agent config, architecture, integrations) → #regrello-afo-ranger-station
- **Pre-Sales** (discovery, positioning, objection handling, competitive) → #help-sell-agentforce-operations
- **Hybrid** (spans both — use case scoping, demo architecture) → searches both equally

## Authoritative Sources

| Source | Channel | Purpose |
|--------|---------|---------|
| Technical | #regrello-afo-ranger-station (`<RANGER_STATION_CHANNEL_ID>`) | Architecture, blueprints, agents, integrations, POC gating |
| Pre-Sales | #help-sell-agentforce-operations (`<SELL_AFO_CHANNEL_ID>`) | Discovery, positioning, objection handling, demos |
| Agent-specific | #regr-ai-agent-help (`<AGENT_HELP_CHANNEL_ID>`) | Which agent to use, agent behavior, troubleshooting |
| Customer stories | #help-sell-agentforce-supply-chain (`<SUPPLY_CHAIN_STORIES_CHANNEL_ID>`) | References, ROI examples |

## Key Reference Document

**Regrello "How to Instructions" Canvas** — pinned in #regrello-afo-ranger-station. Single best source for all "how to" technical questions. Release Notes within it always supersede older how-to content.

## Brand Name Reference

All the same product family:
- Agentforce Operations (AFO) — current umbrella brand
- Regrello — original product name
- Agentforce Process Automation (APA) — commercial SKU
- Agentforce for Supply Chain (ASC) — supply chain SKU

## Notes

- No brain references in this skill.
- Every ID above is a Slack workspace resource identifier (team/channel/canvas), not personal data — but it is still workspace-specific and must be reconfigured per installer per the Configuration section.

## Source

Slack canvas: `https://salesforce.enterprise.slack.com/docs/{config.slack_team_id}/<AFO_HOWTO_CANVAS_ID>` — resolves only inside the origin workspace referenced above.
