---
name: se-rvp-territory-summary
description: "Produces a concise RVP-facing territory summary canvas covering all deals closing in the current and next fiscal quarter, with a dedicated ≥$100K section and a risk assessment per deal using C360 stage and deal health."
metadata:
  type: sales-operations
  version: "1.1"
  depends_on: "se-ae-territory-brief"
---

# SE RVP Territory Summary

You produce a concise, manager-ready territory summary for the RVP. The RVP needs to know: what's closing, what's at risk, and what needs attention — with enough C360 context to ask the right questions on the next forecast call.

This skill is a **consumer of the deal health files** produced by `se-ae-territory-brief`. It does not re-query Org62 or Slack. If deal health files don't exist for today, it reads the most recent available run.

## When to Use

- User says "RVP summary for [AE Name]"
- User says "RVP canvas for [AE]"
- User says "manager summary for [AE]"
- After `se-ae-territory-brief` has been run

## Inputs

- **AE Name** (required)
- **RVP Name** (optional — defaults to "RVP" if not specified)

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `ae_roster_path` | Path to your AE roster/alignment file. Falls back to `~/.claude/territory-reviews/ae-alignment.md` if not configured. |
| `fiscal_calendar` | Optional — centralize fiscal quarter boundaries here instead of the hardcoded table below if your fiscal calendar structure ever changes. |

If `ae_roster_path` is missing, use the default path above.

**Unresolved / flagged (no canonical key — do not invent one):** the `~/.claude/territory-reviews/` root used below for deal health source files has no dedicated config key today. It's already portable (no literal username, just `~`), so it's left as a literal path — flag it if you want it centralized under a future config key.

## AE Roster

Read from `config.ae_roster_path` (falls back to `~/.claude/territory-reviews/ae-alignment.md` if not configured).

## Salesforce Fiscal Quarter Calendar

- FQ1: Feb 1 – Apr 30
- FQ2: May 1 – Jul 31
- FQ3: Aug 1 – Oct 31
- FQ4: Nov 1 – Jan 31

Current quarter = the quarter today's date falls in. Next quarter = the following one.

## State Machine — Source Files

Reads from:
```
~/.claude/territory-reviews/
  [ae-full-name]/
    [YYYY-MM-DD]-[initials]-01-org62.md
    [YYYY-MM-DD]-[initials]-03-classification.md
    [YYYY-MM-DD]-[initials]-04-deal-health/
      [YYYY-MM-DD]-[initials]-[account-slug].md
    [YYYY-MM-DD]-[initials]-07-forecast-prep.md
```

Use the most recent dated set if today's files don't exist. Note the file date at the top of the canvas.

---

## Execution Order

### Step 1: Load Source Files

Read the org62, classification, deal health, and forecast prep files for the AE. Build a working list of all opps with:
- Opp name, account, amount, CRM stage, close date
- Tier (EVP/RVP/Active/Dormant)
- Actual Phase (LISTEN/BUILD TRUST/PARTNER) — from deal health file
- Phase Earned? (Yes/No)
- 5Cs Score (X/5) — from deal health file
- Last customer contact date and days since contact
- RAG (🟢/🟡/🔴/⚪) — from deal health file
- SE Active (✅/❌) — from deal health file (Active DC record in Org62)
- BVS (🟢/❌) — from deal health file (Business_Value_Maturity__c populated or DSR/ROI doc in Slack)
- Partner — from deal health file (named partner, "Lead only", or "None")
- Top recommendation from deal health file

---

### Step 2: Scope Filter

Include only:
1. All deals closing in **current fiscal quarter** (regardless of amount)
2. All deals closing in **next fiscal quarter** (regardless of amount)
3. Any EVP deal (≥$100K) closing in the quarter after that — large enough to flag early

Exclude:
- Dormant deals with no close date in scope
- Deals without an amount

---

### Step 3: Assemble Canvas Content

Build the canvas in this order:

#### 3A. EVP Deals (≥$100K) — Own Section

For every EVP deal in scope, produce a deal card:

```
### [Account] — [Opp Name] | $[Amount] | Closes [Mon DD]

**Stage:** [## - Name] | **Actual Phase:** [LISTEN/BUILD TRUST/PARTNER] | **Earned?** [Yes/No]
**5Cs Score:** [X]/5 | **Last Customer Contact:** [date] ([N] days ago)
**RAG:** 🟢/🟡/🔴/⚪ | **SE Active:** ✅/❌ | **BVS:** 🟢/❌ | **Partner:** [Name / "Lead only" / "None"]

**Risk Assessment:**
[One paragraph. Be direct. What is the honest state of this deal? What would make it slip? What would make it close? If phase is not earned, say so explicitly.]

**One Action to Protect/Advance:**
[The single most important thing the team needs to do in the next 7 days. Tied to a specific gap from the deal health file.]
```

Risk language should be honest and specific. "Stage 05 but no business value quantified and no mutual close plan" is a real risk statement. "Deal looks good" is not.

#### 3B. RVP Deals ($50K–$99K)

Same card structure as EVP, but shorter risk paragraph (2-3 sentences).

#### 3C. Active Deals Closing This Quarter (<$50K)

Table format only — no individual cards:

| Account | Opp | Amount | Stage | Actual Phase | Earned? | RAG | SE Active | BVS | Close Date | Risk |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| [name] | [name] | $[X] | [##] | [phase] | [Y/N] | 🟢/🟡/🔴/⚪ | ✅/❌ | 🟢/❌ | [date] | [1-line: e.g., "No compelling event, may push"] |

#### 3D. Next Quarter Pipeline

One table for all next-quarter deals (all tiers):

| Account | Opp | Amount | Tier | Stage | Actual Phase | RAG | SE Active | BVS | Close Date | Watch |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| [name] | [name] | $[X] | EVP/RVP/Active | [##] | [phase] | 🟢/🟡/🔴/⚪ | ✅/❌ | 🟢/❌ | [date] | [1-line signal or flag] |

"Watch" column: flag anything that looks wrong — e.g., Stage 02 with a next-quarter close, no customer contact in 60 days, 5Cs score ≤2/5.

---

### Step 4: Publish Canvas

Create in Slack with `slack_create_canvas`:

**Title:** `[AE Name] — RVP Territory Summary | [Mon DD, YYYY]`

**Canvas structure:**

```markdown
# [AE Name] — RVP Territory Summary | [Mon DD, YYYY]

### *This canvas was generated using AI. Review for accuracy before using in forecast discussions.*

**AE:** [Name] | **Period:** [Current FQ] close + [Next FQ] pipeline | **Source files:** [date of deal health files]

---
**Legend**
**C360 Phase:** LISTEN = qualifying/discovery (Stages 01–02) · BUILD TRUST = demo/POV/value proof (Stages 03–05) · PARTNER = mutual close plan/commercial (Stages 06–08)
**RAG:** 🟢 Contact ≤14 days + stage earned · 🟡 Contact 15–45 days or phase not earned · 🔴 Contact >45 days or no customer contact · ⚪ Deliberately parked
**5Cs Score:** 1 pt each for documented evidence of — C-level Priorities · Challenges · Competitive Threats · Compelling Events · Potential Crises
**SE Active:** ✅ = Active Deal Contribution record in Org62 · ❌ = No active DC logged
**BVS:** 🟢 = Business Value maturity logged on DC or DSR/ROI doc shared in Slack · ❌ = No BVS evidence
**Partner:** Named partner co-selling · "Lead only" = partner sourced lead, not co-selling · "None" = no partner
---

**Current FQ Commit:** $[sum of EVP+RVP in current FQ at Stage 06+]
**Current FQ Best Case:** $[sum of EVP+RVP in current FQ at Stage 04-05]
**Pipeline at Risk:** $[sum of deals with phase-not-earned or 5Cs ≤ 2/5]

---

# :rotating_light: EVP Deals — ≥$100K

[Deal cards — Step 3A]

---

# :large_orange_circle: RVP Deals — $50K–$99K

[Deal cards — Step 3B]

---

# :small_blue_diamond: Active Deals Closing This Quarter

[Table — Step 3C]

---

# :telescope: Next Quarter Pipeline

[Table — Step 3D]
```

Share the canvas link in the conversation.

---

## Rules

- **RVP time is short.** Lead with what needs attention, not what's healthy. Healthy deals get less space.
- **No jargon the RVP doesn't use.** C360 phase names are fine — they're shared language. SE methodology deep dives are not for this audience.
- **Phase earned = the most important signal.** If a deal hasn't earned its stage, the RVP needs to know. It's the difference between a forecast call that's honest and one that surfaces a surprise at the end of the quarter.
- **One action per deal.** The RVP doesn't want a list. They want to know what the team is doing next.
- **Cite the deal health file date.** If files are from a prior run, note the gap at the top of the canvas.
- **Do not re-score.** All assessments come from the deal health files. Do not re-derive phase or 5Cs from raw CRM data.

## Conversation Starters

- "RVP summary for [AE Name 1]"
- "Manager canvas for [AE Name 2]"
- "RVP territory summary for [AE Name 3]"
