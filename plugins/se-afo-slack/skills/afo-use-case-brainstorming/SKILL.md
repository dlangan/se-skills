---
name: afo-use-case-brainstorming
description: "AFO account briefing with use case discovery. Provide an account name — it auto-researches Org62, Slack channels, web/IR, and AFO use case channels to produce a one-page executive briefing canvas with top 3 pains mapped to OOTB AFO use cases. Supports both AE self-service and Executive meeting prep. Source: Slack canvas (see Configuration)."
metadata:
  type: slack-skill
  version: "1.1"
  source_canvas: "<AFO_BRIEF_CANVAS_ID>"
  author: "AFO Enablement Team"
---

# AFO: Account Brief with Use Cases

This is a Slack Skill (run via Slackbot DM). Give it an account name and it produces a one-page executive briefing canvas with AFO-specific use cases.

> **Workspace-scoped skill.** This skill only functions inside the original Slack workspace where its canvas and channels (`#afo-pains`, `#help-sell-agentforce-operations`) live. If you install this plugin into a different Slack workspace, recreate the equivalent canvas and channels there and fill in the placeholders in Configuration first.

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `config.slack_team_id` | Slack workspace (team) ID this skill's canvas and channels live in. |
| `config.account_root` | Local account workspace root, if you want research artifacts saved alongside other account files. |

The value below is **not** covered by a canonical config key (this skill's canvas isn't the shared `todo_canvas_id`). It is documented here — flagged, not invented — as a required workspace-specific placeholder the installer must set:

| Placeholder | Purpose | Original workspace value (reference only) |
|---|---|---|
| `<AFO_BRIEF_CANVAS_ID>` | Source canvas for this skill's briefing template | `<AFO_BRIEF_CANVAS_ID>` |

Channels referenced by name only (`#afo-pains`, `#help-sell-agentforce-operations`) — no channel IDs were hardcoded in the original skill, but these channels are also workspace-specific and must exist (or be recreated) in the target workspace.

## How to Use

1. Open Slackbot DM
2. Add this skill (canvas `<AFO_BRIEF_CANVAS_ID>`)
3. Provide the customer account name
4. Optionally provide an Executive name (triggers personalization)

## Two Scenarios

- **Scenario A (no Executive named):** AE self-service — produces a use case brief for meeting prep
- **Scenario B (Executive named):** Personalizes framing for CEO/COO/CFO/CMO/CRO/CSCO communication style

## Research Framework

1. Org62 lookup (Industry, Sector, Sub-Sector, Revenue, SI partners, footprint)
2. Sub-industry pain research (#afo-pains channel)
3. Slack account channel discovery (inside-out intelligence)
4. Web research — Investor Relations, earnings, CEO statements
5. AFO use case mapping from #help-sell-agentforce-operations
6. Rapport opener research (recent wins to celebrate)

## Output: One-Page Canvas

- Account Overview
- Top 3 Customer Pains + AFO Use Cases (OOTB, fast-win, low-integration)
- Why Now (urgency drivers)
- The Ask (what we need from the Executive)
- Red Flags
- FY27 Opportunities
- Next Renewal Date
- Rapport Opener
- Health of Customer & Relationship

## Key Design Principle

Prioritizes OOTB, low-integration, fast time-to-value use cases. Avoids recommending anything that requires heavy integrations or long implementation timelines.

## Notes

- No brain references in this skill.
- Never hardcode a real account name, executive name, or research output as an example in this skill file — the workflow above is intentionally generic and account-agnostic.

## Source

Slack canvas: `https://salesforce.enterprise.slack.com/docs/{config.slack_team_id}/<AFO_BRIEF_CANVAS_ID>` — resolves only inside the origin workspace referenced above.
