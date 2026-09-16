---
name: se-leadership-deal-review
description: "Produces a deal review canvas for the SE's manager covering all deals >$50K across the full territory, structured as pre-meeting deal review prep with recommended SE actions and one question per deal."
metadata:
  type: sales-operations
  version: "1.1"
  depends_on: "se-ae-territory-brief"
---

# SE Leadership Deal Review

You produce a deal review prep canvas for the SE's manager. This is the view that gets used in a 1:1 or deal review meeting: >$50K deals, SE engagement quality, what the SE recommends, and the one question that surfaces the most risk or unlocks the most value.

This skill is a **consumer of deal health files** from `se-ae-territory-brief`. It does not re-query Org62 or Slack. All assessments come from the deal health files.

## When to Use

- User says "SE leadership summary"
- User says "deal review for my manager"
- User says "leadership deal review"
- User says "prep my deal review"
- Scheduled before a 1:1 with SE management

## Inputs

- **AE Name(s)** (required) — one AE or "all" to run across the full territory
- **Manager Name** (optional — for canvas header; defaults to "SE Leadership")

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `ae_roster_path` | Path to your AE roster/alignment file. Falls back to `~/.claude/territory-reviews/ae-alignment.md` if not configured. |
| `se_name` | Your name, used in the "Prepared by" line on the canvas. |

If `ae_roster_path` is missing, use the default path above. If `se_name` is missing, ask the user for a name to use or omit the "Prepared by" line.

**Unresolved / flagged (no canonical key — do not invent one):** the `~/.claude/territory-reviews/` root used below for deal health source files has no dedicated config key today. It's already portable (no literal username, just `~`), so it's left as a literal path — flag it if you want it centralized under a future config key.

## AE Roster

Read from `config.ae_roster_path` (falls back to `~/.claude/territory-reviews/ae-alignment.md` if not configured).

## State Machine — Source Files

Reads from:
```
~/.claude/territory-reviews/
  [ae-full-name]/
    [YYYY-MM-DD]-[initials]-01-org62.md
    [YYYY-MM-DD]-[initials]-03-classification.md
    [YYYY-MM-DD]-[initials]-04-deal-health/
      [YYYY-MM-DD]-[initials]-[account-slug].md
    [YYYY-MM-DD]-[initials]-05-one-question.md
```

Use the most recent dated set. Note the file date on the canvas.

---

## Scope

Include: all deals with Amount ≥ $50,000 (EVP and RVP tiers).

Exclude: Active and Dormant deals below $50K.

If "all" AEs specified, produce one canvas covering all AEs — sorted by deal size descending, grouped by AE.

---

## Execution Order

### Step 1: Load Source Files

For each qualifying AE, read the org62, classification, deal health, and one-question files. Extract per qualifying opp:

- Opp name, account, AE name, amount, CRM stage, close date
- Actual Phase (LISTEN/BUILD TRUST/PARTNER)
- Phase Earned? (Yes/No)
- 5Cs score (X/5) and which Cs are missing
- SE Engagement quality signals (from deal health recommendations and gaps)
- The One Question (from Step 5 file)
- Top 2 recommendations from deal health file

---

### Step 2: SE Engagement Assessment Per Deal

For each qualifying deal, derive a 3-level SE engagement quality assessment:

| Level | Criteria |
|---|---|
| **Strong** | Phase earned + 5Cs ≥ 3/5 + methodology activities completed for phase + specific SE next steps defined |
| **Developing** | Phase partially earned OR 5Cs 2/5 OR methodology gaps exist but progress is visible |
| **Needs Attention** | Phase not earned + 5Cs ≤ 1/5 OR no logged SE activities + no defined next steps |

This is not a judgement on the AE — it's a signal about what SE work remains and where SE resources should be focused.

---

### Step 3: Recommended SE Actions

For each deal, derive the SE's recommended next actions from the deal health file. Format as:

```
**SE Recommended Actions:**
1. [Specific action — e.g., "Run complexity assessment with IT lead by Aug 1"] (Owner: SE)
2. [Specific action — e.g., "File BVS DSR to quantify ROI before next forecast call"] (Owner: SE + AE)
```

Max 2 actions per deal. These are SE-owned or SE-led — not AE commercial actions. If there are no SE-specific next steps, note: "No SE-blocking gaps — monitoring."

---

### Step 4: Assemble Canvas Content

Build one deal block per qualifying opp, sorted: EVP deals first (largest to smallest), then RVP deals. Group by AE when running "all."

**Deal block structure:**

```markdown
### [AE Name] | [Account] — [Opp Name] | $[Amount] | Stage [##] | Closes [Mon DD]

**Actual Phase:** [LISTEN/BUILD TRUST/PARTNER] | **Earned?** [Yes / No — reason in one clause]
**5Cs:** [X]/5 | [list missing Cs by name] | **SE Engagement:** [Strong/Developing/Needs Attention]

**SE Recommended Actions:**
1. [Action] (Owner: [SE/AE/BDR]) — by [date]
2. [Action] (Owner: [SE/AE/BDR]) — by [date]

**The One Question:**
"[The question — written the way the SE would say it to the AE or the customer]"
→ What it surfaces: [good answer = X / bad answer = Y]

---
```

If Phase Earned = No, add a one-sentence explicit callout:
> *Stage [##] not earned: [what's been accomplished vs. what the stage requires — e.g., "CRM shows Stage 05 but no Connected Vision delivered and no vendor-of-choice signal received."]*

---

### Step 5: Territory Health Rollup (when "all" AEs)

When running across all AEs, add a summary section at the top:

```markdown
## Territory Health — SE View | [Mon DD, YYYY]

| AE | EVP Pipeline | RVP Pipeline | Strong Engagement | Needs Attention |
| --- | --- | --- | --- | --- |
| [name] | $[X] | $[X] | [N deals] | [N deals] |
| ... | | | | |
| **Total** | **$[X]** | **$[X]** | **[N]** | **[N]** |

**Highest-Risk Deals:**
- [Account/Opp] — [1-line risk reason]
- [Account/Opp] — [1-line risk reason]

**Highest-Opportunity Deals (SE investment would move the needle):**
- [Account/Opp] — [1-line rationale]
- [Account/Opp] — [1-line rationale]
```

---

### Step 6: Publish Canvas

Create in Slack with `slack_create_canvas`:

**Title:** `SE Deal Review — [AE Name or "Full Territory"] | [Mon DD, YYYY]`

**Canvas structure:**

```markdown
# SE Deal Review — [AE Name or "Full Territory"] | [Mon DD, YYYY]

### *This canvas was generated using AI. Review for accuracy before using in management discussions.*

**Prepared by:** [config.se_name] | **Source files:** [date of deal health files]
**Scope:** Deals ≥$50K | [AE name(s)]

[Territory Health Rollup — if "all" run]

---

[Deal blocks — sorted EVP → RVP, largest to smallest]
```

Share the canvas link in the conversation.

---

## Rules

- **SE lens, not AE lens.** This canvas is about SE engagement quality, SE next steps, and SE resource allocation — not commercial progression. The RVP canvas handles forecast. This one handles SE work.
- **Honest engagement assessment.** "Needs Attention" is not a negative judgment — it's a flag for where SE effort should go next. Write it as directional, not critical.
- **The One Question is the most valuable output.** It's the question that either accelerates the deal or surfaces a fatal flaw fast. Ground it in the biggest 5Cs gap or missing methodology activity.
- **Two actions maximum.** The manager doesn't want a brainstorm. They want to know what the SE is doing next on each deal.
- **Cite file dates.** If deal health files are from a prior run, say so. Stale data is worse than acknowledged stale data.
- **Do not re-score or re-derive.** All phase, 5Cs, and engagement evidence comes from deal health files. Do not second-guess the prior assessment.

## Conversation Starters

- "SE leadership summary for all AEs"
- "Deal review prep for my manager"
- "Leadership deal review for [AE Name 1] and [AE Name 2]"
- "Prep my SE deal review canvas"
