---
name: afo-use-case-fit-triage
description: "Bullseye triage for AFO use case requests. Scores accounts against 5 criteria (SF footprint, OOTB fit, integration complexity, ACV threshold, process definition) and produces a BULLSEYE/BORDERLINE/NOT A PRIORITY verdict. Includes Path to Bullseye coaching for non-Bullseye verdicts. Source: Slack canvas (see Configuration)."
metadata:
  type: slack-skill
  version: "1.1"
  source_canvas: "<AFO_TRIAGE_CANVAS_ID>"
  author: "AFO Enablement Team"
---

# AFO (Regrello) Use Case Fit Triage

This is a Slack Skill (run via Slackbot DM). It's a bandwidth protection tool — rapidly assesses whether an account/use case is a Bullseye fit for AFO.

> **Workspace-scoped skill.** This skill only functions inside the original Slack workspace where its canvas and reference PDFs (AFO Use Case Matrix, AFO FINS Use Case Deck) live. If you install this plugin into a different Slack workspace, recreate the equivalent canvas and re-attach the reference documents there.

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `config.slack_team_id` | Slack workspace (team) ID this skill's canvas lives in. |

The value below is **not** covered by a canonical config key (this skill's canvas isn't the shared `todo_canvas_id`). It is documented here — flagged, not invented — as a required workspace-specific placeholder the installer must set:

| Placeholder | Purpose | Original workspace value (reference only) |
|---|---|---|
| `<AFO_TRIAGE_CANVAS_ID>` | Source canvas for this skill's triage template | `<AFO_TRIAGE_CANVAS_ID>` |

## How to Use

1. Open Slackbot DM
2. Add this skill (canvas `<AFO_TRIAGE_CANVAS_ID>`)
3. Provide: Account name + Proposed use case description
4. Optionally provide: RFP, meeting transcript, PDF, or POV canvas

## Bullseye Scoring Criteria

| # | Criteria | Met | Partial | Not Met |
|---|----------|-----|---------|---------|
| 1 | Existing SF Footprint | Meaningful deployment | Some SF | None |
| 2 | OOTB Use Case Fit | Matrix match + Pre-Built=Yes | Matrix match, Pre-Built=No | No matrix match |
| 3 | No Complex Integrations | Achievable without custom dev | Some integration needed | Unique/high-effort required |
| 4 | Deal Size / ACV | $1M+ | $500K-$999K | <$500K or unknown |
| 5 | Process/Workflow Definition | Steps, stakeholders, handoffs clear | High-level only | No meaningful definition |

These ACV bands are generic triage thresholds (methodology), not a real deal's dollar amount — apply them to whatever the actual opportunity size is.

## Verdicts

- **BULLSEYE:** All 5 scored, OOTB + Integration both Met, ACV $1M+
- **BORDERLINE:** 3-4 Met/Partial, or OOTB/Integration is Partial, or ACV unknown
- **NOT A PRIORITY:** 2 or fewer Met, OR any hard gate triggered (OOTB Not Met, Integration Not Met, Process Not Met, ACV confirmed <$500K)

## Key Feature: Path to Bullseye

For BORDERLINE/NOT A PRIORITY verdicts, the skill provides:
- Discovery questions to ask the customer (tied to weak criteria)
- Use case pivots to consider (closest OOTB analogs)
- Conditions that would change the verdict

## Authoritative OOTB References

1. **AFO Use Case Matrix** (PDF in Slack) — master list of all documented use cases
2. **AFO FINS Use Case Deck** — for Financial Services accounts

## Notes

- No brain references in this skill.
- Score the account presented to you at runtime only — never hardcode a real account name or deal size as an example in this skill file.

## Source

Slack canvas: `https://salesforce.enterprise.slack.com/docs/{config.slack_team_id}/<AFO_TRIAGE_CANVAS_ID>` — resolves only inside the origin workspace referenced above.
