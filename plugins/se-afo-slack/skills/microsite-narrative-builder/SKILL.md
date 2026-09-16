---
name: microsite-narrative-builder
description: "Builds executive-quality strategic narratives for customer microsites. Auto-researches CRM, Slack, web, and Highspot; asks targeted gap-filling questions; generates a polished narrative with brand observation, account team, and Claude Code handoff. Source: Slack canvas (see Configuration)."
metadata:
  type: slack-skill
  version: "1.1"
  source_canvas: "<MICROSITE_NARRATIVE_CANVAS_ID>"
  author: "David Langan (plugin migration); original Slack skill contributed by a Salesforce colleague — name and email withheld from this public repo"
---

# Microsite Narrative Builder

This is a Slack Skill (run via Slackbot DM). It builds executive-quality account narratives that are then handed to Claude Code to populate a pre-built microsite template.

> **Workspace-scoped skill.** This skill only functions inside the original Slack workspace where its canvas lives. If you install this plugin into a different Slack workspace, recreate the equivalent canvas there and fill in the placeholder in Configuration first.
>
> **Historical note:** this skill's canvas originated in the same Slack workspace as `account-partner-intelligence` above, but a *different* workspace than the three `afo-*` skills bundled in this same plugin. All skills in this plugin now read a single `config.slack_team_id` value — before relying on more than one of these skills together, confirm that value actually matches the workspace where each skill's canvas exists.

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `config.slack_team_id` | Slack workspace (team) ID this skill's canvas lives in. |
| `config.account_root` | Local account workspace root, if you want the generated narrative saved alongside other account files. |

The value below is **not** covered by a canonical config key (this skill's canvas isn't the shared `todo_canvas_id`). It is documented here — flagged, not invented — as a required workspace-specific placeholder the installer must set:

| Placeholder | Purpose | Original workspace value (reference only) |
|---|---|---|
| `<MICROSITE_NARRATIVE_CANVAS_ID>` | Source canvas for this skill's narrative-builder template | `<MICROSITE_NARRATIVE_CANVAS_ID>` |

**External dependency (not a Slack ID, kept as-is):** this skill hands its output to a public GitHub template repository, `https://github.com/2outof5aintbad/agentic-enterprise-microsite-template`, which the Claude Code handoff step populates. That repository is already public and is required for the skill to function, so it has been left as a literal URL rather than a config placeholder — flagging it here for visibility since the org/user name in that URL looks like a personal handle rather than an official Salesforce org.

## How to Use

1. Open Slackbot DM
2. Add this skill to your Slackbot tab
3. Provide the customer name — it auto-researches CRM, Slack, Highspot, and the web
4. Answer 3-5 targeted gap-filling questions
5. Confirm the narrative angle
6. Receive a full narrative + Claude Code handoff block

## What It Produces

- Strategic narrative (not JSON, not a deployment plan)
- Brand observation (typography, color, density) for template theming
- Account team block with Slack photo URLs
- Sanitization review (flags deal values, internal refs, unsupported claims)
- Ready-to-paste Claude Code handoff

## Key Capabilities

- Auto-pulls: Salesforce CRM, Slack channels, Highspot/Seismic, web (earnings, press, CEO statements)
- Asks provocative discovery questions (not generic)
- Supports multiple narrative angles: Workflow Transformation, Platform Consolidation, Customer Zero, AI Transformation, Data Foundation, Competitive Urgency
- Personalizes by executive role (CEO, COO/CFO, CMO, CRO, CSCO)
- Iteration support: "make it more executive", "rewrite for a CIO", "sharpen the thesis"

## Notes

- No brain references in this skill.
- This skill's own "Sanitization review" step (flagging deal values, internal refs, unsupported claims before the narrative leaves Slack) is part of its methodology and has been preserved as-is — it is a good practice this skill already enforces on its own output.
- Never hardcode a real customer name, executive name, or account-team roster as an example in this skill file.

## Source

Slack canvas: `https://salesforce.enterprise.slack.com/docs/{config.slack_team_id}/<MICROSITE_NARRATIVE_CANVAS_ID>` — resolves only inside the origin workspace referenced above.
