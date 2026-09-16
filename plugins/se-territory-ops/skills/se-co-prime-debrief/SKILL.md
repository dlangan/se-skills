---
name: se-co-prime-debrief
description: "Post-meeting debrief after a co-prime sync. Reads the pre-meeting co-prime brief files as baseline, layers in meeting notes, and produces a delta canvas: what changed, what was committed to, what's now unblocked or at risk."
metadata:
  type: sales-operations
  version: "1.1"
  depends_on: "se-co-prime-brief"
---

# SE Co-Prime Debrief

You capture what actually happened in a co-prime sync and turn it into a structured delta against the pre-meeting brief. The output is a Slack canvas the co-prime and AE can both act from.

## Configuration

This skill has no direct config keys of its own — it reads the dated baseline files written by `se-co-prime-brief`, which already externalizes the co-prime roster via `config.specialist_roster` (see `~/.claude/se-config.json` and the `se-co-prime-brief` Configuration section). No roster, name, or ID should be hardcoded here either.

## When to Use

- "Debrief from my sync with [Co-Prime Name]"
- "Post-meeting notes for [Co-Prime Name] — here's what we covered"
- "Update the co-prime brief for [Name] after our call"
- After any co-prime 1:1, team call, or pipeline review where deal status was discussed

---

## Inputs

- **Co-prime name** (required) — who the sync was with
- **Meeting notes** (required) — paste raw notes, a transcript, or a bullet summary. Can be messy.
- **Date** (optional) — defaults to today's actual date at execution time

---

## Step 1: Load the Pre-Meeting Baseline

Resolve the co-prime slug (lowercase hyphenated name, matching the slug used by `se-co-prime-brief`) and read the most recent dated files from:

```
~/.claude/territory-reviews/co-primes/[co-prime-slug]/
```

Files to read (most recent date prefix):
- `[date]-01-opportunities.md` — the opportunity list
- `[date]-02-classification.md` — tier / days dark table
- `[date]-04-recommendations.md` (or `04-deal-health.md`) — pre-meeting recommendations

If no prior brief files exist, note "No prior brief on file — debrief will stand alone without a baseline comparison."

---

## Step 2: Parse the Meeting Notes

Extract from the raw notes:

**Deals discussed** — for each deal mentioned:
- Account name and opp (match to baseline list)
- What was said: status update, blocker surfaced, decision made, action committed to
- Who said it (you / co-prime / AE if present)

**New deals or accounts** — any account or opp mentioned that wasn't in the baseline brief

**Commitments** — anything with a named owner and implied or explicit deadline

**Blockers surfaced** — things that came up as new risks or blockers not previously flagged

**Co-sell signals** — accounts where the co-prime said they want to get involved or were asked about

---

## Step 3: Produce the Delta

For each deal from the baseline brief that was discussed, classify the change:

| Status | Meaning |
|---|---|
| ✅ **Unblocked** | A gap from the brief was addressed in the meeting |
| 🔼 **Advanced** | Deal moved forward — stage progressed, meeting booked, commitment secured |
| ⚠️ **New Risk** | A blocker or risk surfaced that wasn't in the brief |
| 🔄 **Updated** | Status changed but direction unclear yet |
| 🔇 **Not Discussed** | Was in the brief but didn't come up — flag if it was a Key or Significant deal |

---

## Step 4: Write the Debrief Canvas → `[date]-debrief.md` + Slack

Write the file to:
```
~/.claude/territory-reviews/co-primes/[co-prime-slug]/[date]-debrief.md
```

Then create in Slack.

---

## Canvas Structure

**Title:** `[Co-Prime Name] — Post-Sync Debrief | [Mon DD, YYYY]`

```markdown
# [Co-Prime Name] — Post-Sync Debrief | [Mon DD, YYYY]
**Synced with:** [Co-prime name + role] | **Duration:** [X min if known]
**Pre-meeting brief:** [date of the brief this debrief is based on]

### *This canvas was generated using AI, which can produce inaccurate or harmful responses. Review for accuracy and safety before using.*

---

# :white_check_mark: Commitments Made

Concrete actions with an owner — follow up if not completed by the implied date.

| Deal | Commitment | Owner | By |
| --- | --- | --- | --- |
| [Account — Opp] | [Specific action committed to — not vague, e.g. "Send pricing deck to [Contact Name]"] | [Co-prime / You / AE / BDR] | [Date or "this week"] |

---

# :arrows_counterclockwise: Deal Updates

What changed versus the pre-meeting brief.

## Unblocked / Advanced

| Deal | Was | Now | What Changed |
| --- | --- | --- | --- |
| [Account — Opp ($Amount)] | [Prior state from brief] | [New state] | [What happened in the meeting that changed it] |

## New Risks Surfaced

| Deal | Risk | Impact | Next Move |
| --- | --- | --- | --- |
| [Account — Opp ($Amount)] | [New blocker or risk that came up] | [What's at stake] | [Who does what next] |

## Status Updates (no direction change yet)

| Deal | Update | Note |
| --- | --- | --- |
| [Account — Opp] | [What was said] | [Context] |

---

# :no_bell: Not Discussed

Key or Significant deals from the pre-meeting brief that didn't come up. Flag for follow-up.

| Deal | Tier | Why It Matters |
| --- | --- | --- |
| [Account — Opp ($Amount)] | [Key / Significant] | [Why this is a gap — e.g. "120 days dark, was flagged for re-engage"] |

---

# :bulb: New Signals

Accounts or opportunities that came up in the meeting but weren't in the pre-meeting brief.

| Account | Signal | Suggested Next Step |
| --- | --- | --- |
| [Account] | [What was said — e.g. "Co-prime mentioned they're being asked about Field Service by the AE"] | [What to do with it] |

---

# :spiral_notepad: Raw Notes (Reference)

[Paste or lightly cleaned version of the meeting notes here for future reference]

---
```

---

## Rules

- **Delta first.** The canvas exists to surface what changed, not to recap what was already in the brief. If nothing changed on a deal, it doesn't need a row — just the "Not Discussed" flag.
- **Commitments are the product.** A sync without commitments is a conversation, not a meeting. If no commitments were made, flag it explicitly: "No commitments captured — follow up to confirm next steps."
- **Link every opp.** Every opportunity reference must hyperlink: `[Opp Name](https://salesforce.lightning.force.com/lightning/r/Opportunity/[OppID]/view)` — use IDs from the baseline brief files.
- **Not Discussed is signal.** If a Key deal didn't come up in a sync with the relevant co-prime, that's worth flagging — either the deal isn't real to them, or it got dropped.
- **Raw notes go at the bottom.** Always append the source notes. This preserves the original record and lets anyone verify what was actually said.
- **File write always happens.** Even if the meeting was light, write the debrief file and the canvas. The blank "Not Discussed" section alone has value.
- **No hardcoded names.** Co-prime, AE, and contact names in canvas rows are drawn from the meeting notes and baseline brief files at runtime — never hardcode an example person into this skill's logic (this repo is public).

## Relationship to the Brief Skill

| se-co-prime-brief | se-co-prime-debrief |
|---|---|
| Runs before the sync | Runs after the sync |
| Input: Org62 + Slack | Input: meeting notes |
| Output: what should be discussed | Output: what actually happened |
| Audience: you + co-prime prep | Audience: you + AE + co-prime follow-up |

The debrief file (`[date]-debrief.md`) sits alongside the brief files in the same co-prime directory. The next brief run will read both to build a richer baseline.
