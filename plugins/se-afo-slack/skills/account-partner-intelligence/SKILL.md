---
name: account-partner-intelligence
description: "Queries Org62 opportunities for a named account, surfaces top SI/referral partners by deal frequency and value, identifies recurring partner relationships. Source: Slack canvas (see Configuration)."
metadata:
  type: slack-skill
  version: "1.1"
  source_canvas: "<PARTNER_INTEL_CANVAS_ID>"
  author: "Unknown (originally shared internally via Slack; original contributor and channel name withheld — this repo is public)"
---

# Account Partner Intelligence Report

This is a Slack Skill (run via Slackbot DM). It queries Org62 for an account's opportunity history and surfaces partner patterns.

> **Workspace-scoped skill.** This skill only functions inside the original Slack workspace where its canvas lives. If you install this plugin into a different Slack workspace, recreate the equivalent canvas there and fill in the placeholder in Configuration first.
>
> **Historical note:** this skill's canvas originated in a different Slack workspace (team) than the three `afo-*` skills bundled in this same plugin. All skills in this plugin now read a single `config.slack_team_id` value — before relying on more than one of these skills together, confirm that value actually matches the workspace where each skill's canvas exists, since they were not all originally in the same workspace.

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `config.slack_team_id` | Slack workspace (team) ID this skill's canvas lives in. |
| `config.account_root` | Local account workspace root, if you want the partner report saved alongside other account files. |

The value below is **not** covered by a canonical config key (this skill's canvas isn't the shared `todo_canvas_id`). It is documented here — flagged, not invented — as a required workspace-specific placeholder the installer must set:

| Placeholder | Purpose | Original workspace value (reference only) |
|---|---|---|
| `<PARTNER_INTEL_CANVAS_ID>` | Source canvas for this skill's report template | `<PARTNER_INTEL_CANVAS_ID>` |

The original skill's `metadata.author` also referenced the specific internal Slack channel it was shared in by name. That channel name has been withheld here (it is workspace-specific provenance, not required for the skill to function) rather than mapped to a config key.

## How to Use

1. Open Slackbot DM
2. Provide the account name
3. Skill searches Org62, disambiguates if needed, then pulls 3 years of opportunities

## What It Produces

- Top partners ranked by frequency and total deal value
- Partner roles (Lead SI, Reseller, Referral)
- Most recent opportunity date per partner
- Flagging of recurring vs. one-off partners
- Data gap identification (blank partner fields)

## Output Format

**Top Partners Table:**
| Partner | # Opps | Total Value | Most Recent Opp | Primary Role |

**Opportunity Breakdown Table:**
| Opp Name | Stage | Amount | Close Date | Partner |

**Key Observations:** 3 most actionable insights about partner mix, trends, or gaps.

## Key Principles

- Always confirms account before querying
- Includes both won AND lost opportunities
- Resolves partner name from lookup references
- Limits observations to 3 actionable insights
- Never hardcode a real account, partner, or opportunity name as an example in this skill file — all figures above are placeholders describing the output shape, not real deal data.

## Notes

- No brain references in this skill.

## Source

Slack canvas: `https://salesforce.enterprise.slack.com/docs/{config.slack_team_id}/<PARTNER_INTEL_CANVAS_ID>` — resolves only inside the origin workspace referenced above.
