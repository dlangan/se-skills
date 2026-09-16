---
name: se-co-prime-brief
description: "Produces a co-prime brief for a specific specialist co-prime — surfacing their active opportunities across your territory, assessing deal health from the co-prime's product lens, and recommending concrete next actions. Outputs a Slack canvas written for the co-prime as the audience."
metadata:
  type: sales-operations
  version: "1.0"
---

# SE Co-Prime Brief

You are a Salesforce Solutions Engineering assistant that produces a co-prime brief for a specific specialist co-prime — surfacing their active opportunities across your territory, assessing deal health from the co-prime's product lens, and recommending concrete next actions.

The output is a Slack canvas structured for scanning. The co-prime is the audience, not the AE.

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `specialist_roster` | Your co-prime specialists — name, role, Salesforce User ID, and product filter (family or name-pattern) per person. **Do not hardcode a roster or Salesforce User IDs in this skill.** |
| `ae_roster` | Your aligned AEs — the territory this brief pulls opportunities from. |

If `specialist_roster` is empty or missing, stop and ask the user to identify the co-prime (name, role, Salesforce User ID, and product filter) before proceeding — do not guess or invent a roster.

## When to Use

- User says "co-prime brief for [Co-Prime Name]"
- User says "brief for [Co-Prime Name]"
- User says "[Co-Prime Name] brief"
- User says "co-prime brief" (ask which co-prime if not specified)

---

## Co-Prime Roster

Read the authoritative co-prime roster from `config.specialist_roster` (see Configuration above). Do not hardcode names, roles, or Salesforce User IDs — pull them from config at runtime.

**Role → Product Filter reference** (methodology, not personal data — match each role to a real name via `config.specialist_roster`):

| Role | Filter Type | Filter Value |
|---|---|---|
| Service Cloud AE | Family | `Service Cloud` |
| Marketing Cloud AE | Family | `Marketing Cloud` |
| Slack AE | Family | `Slack` |
| Commerce AE | Family | `Commerce Cloud` |
| Analytics Specialist AE | Family | `Analytics` |
| Platform & Security AE | Family | `Custom Cloud` (disambiguate via Name — see Rules) |
| MuleSoft Consumption Seller | Family | `Mulesoft` |
| Employee Service AE | Family | `Platform` AND Name LIKE `%Employee Productivity%` OR `%IT Service Center%` OR `%Workplace Command Center%` |
| Revenue Cloud AE | Family = `Sales Cloud` AND Name | LIKE `%Revenue Cloud%` OR `%CPQ%` OR `%Salesforce Billing%` OR `%Salesforce Contracts%` |
| SPM AE | Family = `Sales Cloud` AND Name | LIKE `%Sales Planning%` OR `%Salesforce Maps%` |
| Partner Cloud AE | Family = `Sales Cloud` AND Name, OR Family = `Communities` | LIKE `%Partner Ecosystem Management%` |
| Data Cloud AE / Data Foundations AE | Name | LIKE `%Data Cloud%` OR `%Data 360%` |
| Agentforce AE | Name | LIKE `%Agentforce%` OR `%Agent Fabric%` |
| Agentforce + DC Consumption Seller | Name | LIKE `%Agentforce%` OR `%Data Cloud%` OR `%Data 360%` |

Each `config.specialist_roster` entry should carry: `name`, `role`, `user_id`, `filter_type` (`family` or `name`), `filter_value` (string or array of LIKE patterns). Example shape:

```jsonc
"specialist_roster": [
  { "name": "[Name]", "role": "Service Cloud AE", "user_id": "<salesforce_user_id>", "filter_type": "family", "filter_value": "Service Cloud" },
  { "name": "[Name]", "role": "Revenue Cloud AE", "user_id": "<salesforce_user_id>", "filter_type": "name", "filter_value": ["%Revenue Cloud%", "%CPQ%", "%Salesforce Contracts%"] }
]
```

**Product-brain SKU reference files** (load on demand for deal context):
- `~/product-brain/service-cloud/service-ae-skus.md`
- `~/product-brain/marketing-cloud/marketing-ae-skus.md`
- `~/product-brain/slack/slack-ae-skus.md`
- `~/product-brain/commerce-cloud/commerce-ae-skus.md`
- `~/product-brain/analytics/analytics-ae-skus.md`
- `~/product-brain/custom-cloud/platform-security-ae-skus.md`
- `~/product-brain/mulesoft/mulesoft-ae-skus.md`
- `~/product-brain/employee-service/employee-service-ae-skus.md`
- `~/product-brain/revenue-cloud/revenue-cloud-ae-skus.md`
- `~/product-brain/spm/spm-ae-skus.md`
- `~/product-brain/communities/partner-cloud-ae-skus.md`
- `~/product-brain/agentforce/data-cloud-ae-skus.md`
- `~/product-brain/agentforce/agentforce-ae-skus.md`

---

## State Machine — File Structure

Every run writes dated files under:
```
~/.claude/territory-reviews/
  co-primes/
    [co-prime-slug]/
      [YYYY-MM-DD]-01-opportunities.md    ← Org62 opp pull
      [YYYY-MM-DD]-02-classification.md   ← tier + days dark
      [YYYY-MM-DD]-03-deal-notes.md       ← Slack signals per opp
      [YYYY-MM-DD]-04-recommendations.md  ← actions per opp
      [YYYY-MM-DD]-canvas.md              ← assembled canvas
```

**Co-prime slug:** Lowercase hyphenated name (e.g., `jane-doe`, from `config.specialist_roster`).

**Resume logic:** Before executing any step, check if today's dated file exists for that step. If yes — read it and skip the query/work. If no — execute and write, then continue. Every run is resumable from the last successful step.

Create directories as needed before writing files.

---

## Execution Order

### Step 1: Find All Matching Opportunities → `[date]-01-opportunities.md`

The goal: find every open opportunity in your territory where **all three** conditions are true simultaneously:
1. The co-prime is listed on the **OpportunityTeamMember** for that specific opportunity (not just the account), AND
2. That team member record is **active** (no IsDeleted / role not 'Inactive'), AND
3. At least one **OpportunityLineItem** on that opp matches the co-prime's product responsibility

This is a single two-query pattern — do not use AccountTeamMember.

---

**Step 1A — Get qualifying opportunity IDs (co-prime on team + matching product):**

Use the appropriate product filter for the co-prime (from `config.specialist_roster.filter_type`/`filter_value`). Two variants:

**Pattern A — Family filter** (used when `filter_type: "family"`):
```sql
SELECT otm.OpportunityId,
       otm.Opportunity.Name, otm.Opportunity.StageName,
       otm.Opportunity.Amount, otm.Opportunity.CloseDate,
       otm.Opportunity.CurrencyIsoCode, otm.Opportunity.ForecastCategoryName,
       otm.Opportunity.LastActivityDate, otm.Opportunity.NextStep,
       otm.Opportunity.Manager_Forecast_Judgement__c,
       otm.Opportunity.OwnerId, otm.Opportunity.Owner.Name,
       otm.Opportunity.AccountId, otm.Opportunity.Account.Name
FROM OpportunityTeamMember otm
WHERE otm.UserId = '[co-prime-user-id]'
  AND otm.Opportunity.IsClosed = false
  AND otm.Opportunity.Amount > 0
  AND otm.OpportunityId IN (
    SELECT oli.OpportunityId
    FROM OpportunityLineItem oli
    WHERE oli.Product2.Family = '[family]'
  )
ORDER BY otm.Opportunity.Amount DESC
```

**Pattern B — Name filter** (used when `filter_type: "name"`):
```sql
SELECT otm.OpportunityId,
       otm.Opportunity.Name, otm.Opportunity.StageName,
       otm.Opportunity.Amount, otm.Opportunity.CloseDate,
       otm.Opportunity.CurrencyIsoCode, otm.Opportunity.ForecastCategoryName,
       otm.Opportunity.LastActivityDate, otm.Opportunity.NextStep,
       otm.Opportunity.Manager_Forecast_Judgement__c,
       otm.Opportunity.OwnerId, otm.Opportunity.Owner.Name,
       otm.Opportunity.AccountId, otm.Opportunity.Account.Name
FROM OpportunityTeamMember otm
WHERE otm.UserId = '[co-prime-user-id]'
  AND otm.Opportunity.IsClosed = false
  AND otm.Opportunity.Amount > 0
  AND otm.OpportunityId IN (
    SELECT oli.OpportunityId
    FROM OpportunityLineItem oli
    WHERE oli.Product2.Name LIKE '[pattern1]'
       OR oli.Product2.Name LIKE '[pattern2]'
  )
ORDER BY otm.Opportunity.Amount DESC
```

> **⚠️ DO NOT filter by AE name, OwnerId, or territory.** The co-prime supports multiple AE territories. Pull ALL open opportunities where the co-prime is on the team with a matching product — regardless of which AE owns the account. Filtering to your AEs would hide the co-prime's full book.

---

**Step 1B — Pull line items for each qualifying opp:**

```sql
SELECT Product2.Name, Product2.Family, Quantity, TotalPrice
FROM OpportunityLineItem
WHERE OpportunityId = '[opp-id]'
ORDER BY TotalPrice DESC
```

**Line item batching:** Run this query in batches of up to 50 opp IDs using `WHERE OpportunityId IN (...)` to avoid N+1 queries. For large pulls (>50 opps), split into 2–3 batches and merge results.

**What to record in `01-opportunities.md`:** Under each opp, list ALL line items from this query so the file is complete. Filter to only the co-prime's product family when computing the SKU column for canvas tables (see Step 4A).

**SKU compact format** for canvas tables: show the top 1–2 line items by TotalPrice, using short labels (e.g. `SC-EE ×45 + AfS-UE ×37`). Skip provisioning/adjustment lines with TotalPrice ≤ 0. Show `—` if no matching co-prime SKU with value > 0. If more than 2 positive items, append `(+N)`.

---

**Step 1C — Pull last activity for each qualifying opp:**

```sql
SELECT Id, Subject, ActivityDate, Type, WhoId, Who.Name
FROM Task
WHERE WhatId = '[opp-id]'
  AND ActivityDate >= LAST_N_DAYS:90
ORDER BY ActivityDate DESC
LIMIT 5
```

---

**Post-query exclusions — apply after results are in:**

- Exclude opps whose name pattern matches your org's known non-deal motions (e.g. webstore/online-sales renewals, courtesy credits) — define this exclusion list locally; it's org-specific and not a canonical config key.
- Exclude opps where Amount is null or 0 (belt-and-suspenders — the WHERE clause covers most)

**Bundle contamination check:**
Many multi-cloud bundles (e.g. an "all-clouds" or "Einstein 1"-style bundle) include co-prime SKUs as minor line items. After pulling line items, check:

- If the co-prime's SKU is NOT the top line item by TotalPrice AND the opp name clearly belongs to another motion (e.g. a different cloud's named bundle) → tag it **Supporting (bundle)** and note "verify with AE — [product] motion may not be real." Do NOT surface in Recommended Actions.
- Exception: a contact-center-flavored Agentforce SKU (Family = `Service Cloud`) is always valid for the Service Cloud co-prime even inside a bundle — it IS a Service motion.

---

Write results to `[date]-01-opportunities.md`. One section per account, one sub-section per qualifying opp. Include:
- Opportunity metadata: name (linked), amount + currency, stage, forecast category, close date, AE (owner), MFJ, last activity date
- All co-prime-relevant line items: product name, quantity, total price
- Last activity (from Step 1C) if available

This file is the source of truth for SKU data used in canvas tables. Do not omit line items — the canvas update pass reads from this file.

---

### Step 2: Classification → `[date]-02-classification.md`

For every qualifying opp, classify by tier and days dark:

**Tiers:**

| Tier | Criteria |
|---|---|
| **Key** | Amount ≥ $100K |
| **Significant** | Amount $50K–$99K |
| **Active** | Amount < $50K AND last activity ≤ 60 days |
| **Dark** | Last activity > 60 days (any amount — Key/Significant opps also get a dark flag) |

**Days dark** = days since last Task or Event with a WhoId (customer contact present). AE-internal tasks don't count.

**SKU match quality** — for each opp, note whether the co-prime's product is:
- **Primary** — the co-prime's product is the largest line item or is the clear reason for the opp
- **Supporting** — the co-prime's product is a secondary add-on to a larger Sales/Service Cloud deal
- **Potential** — the opp has no line items yet but the account has the co-prime on the ATM, suggesting a future conversation

Write classification table to `[date]-02-classification.md`:

| Account | AE | Opp | Amount | Stage | Tier | SKU Match | Last Contact | Days Dark | Dark Flag |
|---|---|---|---|---|---|---|---|---|---|

---

### Step 3: Signals — Slack + Org62 Activity → `[date]-03-deal-notes.md`

Once the opportunity set is in hand, pull all available context for each qualifying opp. Run all three sources in parallel across Key and Significant tiers; include Active opps if the count is manageable.

#### 3A — Slack

For each qualifying account:

1. Find the `#ZC` deal channel for the account (search `ZC [Account Name]`). Read the last 30 days.
2. Search `"[Account Name]" "[co-prime product area]"` across all public and private channels — last 60 days.
3. Search for the co-prime's name + the account name together (from `config.specialist_roster`) to surface any DMs or threads between you and the co-prime about this account.
4. Look for call transcript threads — call-recording posts often land in deal channels or `#call-notes` style channels. Pull any that reference the co-prime's product area.

Capture: call outcomes, customer objections, competitive mentions, co-sell coordination, any commitments the co-prime made to the AE or customer.

#### 3B — Org62 Activity

For each qualifying opp, query the last 90 days of Tasks:

```sql
SELECT Id, Subject, ActivityDate, Type, Status, WhoId, Who.Name, Description
FROM Task
WHERE WhatId = '[opp-id]'
  AND ActivityDate >= LAST_N_DAYS:90
  AND IsClosed = true
ORDER BY ActivityDate DESC
LIMIT 10
```

Also pull open Tasks (next steps / commitments logged):

```sql
SELECT Id, Subject, ActivityDate, Type, Status, WhoId, Who.Name, Description
FROM Task
WHERE WhatId = '[opp-id]'
  AND IsClosed = false
ORDER BY ActivityDate ASC
LIMIT 5
```

Flag any task where the co-prime is the WhoId owner, or where the subject references the co-prime's product area — these are the strongest signal that the product motion is real.

#### 3C — Synthesis

Write signals to `[date]-03-deal-notes.md` organized by account. For each account, note:
- **Source** (Slack channel / Org62 Task / call transcript)
- **Signal** (what was said or logged)
- **Owner** (AE / Co-prime / You / Customer)
- **Date**
- **Implication** (what this means for the co-prime's product motion — advancing, stalled, at risk, or unknown)

---

### Step 4: Deal Health + Outreach + One Question → `[date]-04-recommendations.md`

This step produces all canvas content. Write to `[date]-04-recommendations.md`.

---

#### C360 Sales Methodology — Phase Definitions

Use these phases to assess deal state from the co-prime's product lens. Assess ACTUAL phase vs. CRM stage independently.

| Phase | Signal | Evidence Required |
|---|---|---|
| **LISTEN** | Active qualification; customer agreed to engage; discovery in progress | Named discovery questions asked about the co-prime's product area; customer responding |
| **BUILD TRUST** | Discovery complete for this product; demo delivered; POV/Connected Vision shared; customer validating | Demo or POV delivered; customer has articulated a specific use case for this product |
| **PARTNER** | Mutual plan in motion; commercial terms being discussed; procurement engaged | Pricing shared; eval criteria agreed; exec alignment both sides on this product motion |

**Actual Phase Assessment rule:** Assign the phase based on evidence — not CRM stage. A Stage 04 deal where no product-specific discovery has occurred is still LISTEN. A deal where the co-prime has delivered a POV and the customer is doing internal eval is BUILD TRUST even if still Stage 02.

**5Cs scoring** — score each C from the co-prime's product lens:
- ✅ Confirmed with evidence
- 🟡 Partial — implied or partially explored  
- ❌ Unknown or not addressed for this product area

---

#### 4A. Forecast Call Prep table

For every MFJ-set opp (IN / UP+ / UP-): write a one-row forecast prep entry. The primary axis is **the co-prime's role and gap** — not the AE name. AE appears only as context. Answer: what would be asked about the co-prime's product involvement on a forecast call, and what needs to happen?

| Opportunity | [Product] SKUs | C360 Phase | Gate / Next Action | Account Owner | [Co-Prime]'s Co-Prime Role | Gap | Ask | Action |
|---|---|---|---|---|---|---|---|---|
| [Account — Opp Name ($Amount, Stage ##) — linked] | [compact SKU string from 01-opportunities.md] | [LISTEN / BUILD TRUST / PARTNER] + ⚠️ if CRM stage is ahead of evidence | [One line: what this product motion needs to advance from current phase to next] | [Account Owner = Opportunity Owner = AE name] | [specific co-prime role: POV author / re-engage architect / not yet engaged / blocked] | [the specific gap blocking the co-prime's product motion] | [The question to ask on the forecast call] | [Specific time-bound action — co-prime or AE — with date] |

*Substitute the co-prime's actual first name (from `config.specialist_roster`) for `[Co-Prime]` in the column header when generating the canvas, and the co-prime's product area (e.g. "Service", "Marketing", "FSL") for `[Product]`.*

**SKU column rules:** Use the compact format defined in Step 1B. Pull from the `01-opportunities.md` line items for this opp. The column name should match the co-prime's product area. Show `—` if no qualifying line items.

**C360 Phase column rules:**
- Assign phase based on evidence only — never CRM stage. A Stage 04 deal with no product-specific discovery is still LISTEN.
- Append ⚠️ if the CRM stage implies further progress than the evidence supports (stage not earned).
- For IN/UP+ tables: show both `C360 Phase` and `Gate / Next Action` as separate columns.
- For UP- table: show a single `C360` column (phase + ⚠️ only) — no gate narrative column (table is already wide).
- Gate / Next Action: one sentence — the specific gate condition required to advance from the current phase to the next, anchored to the co-prime's product motion on this specific deal.

---

#### 4B. This Week's Outreach table

For every MFJ-set opp with customer contact > 30 days AND every null/Pipeline opp with customer contact > 60 days: one outreach row. The co-prime is often the right owner because they can re-engage on a product angle when the AE's relationship angle has gone cold.

The "[Co-Prime]'s Co-Prime Role" column replaces AE-name-as-primary-axis. AE name appears only in the Via or Owner column as needed. Account Owner = Opportunity Owner = AE name (they are the same in this territory).

| Contact / Account | Account Owner | [Co-Prime]'s Co-Prime Role | Opportunity | Hook | Via | Owner |
|---|---|---|---|---|---|---|
| [Named person or AE DM / account name] | [Account Owner = AE name] | [specific co-prime role on this deal] | [Opp name + amount — linked] | [Specific hook — not "checking in"] | [Email / Slack DM / Call] | [Co-prime / AE / Co-prime proposes, AE sends] |

---

#### 4C. The One Question table

For every MFJ-set opp (IN / UP+ / UP-): derive the one question that either advances the co-prime's product motion or surfaces a fatal flaw fast.

Use C360 gate notation for the Product Gate column:
`[current C360 phase] → [next C360 phase]`
`Requires: [specific gate condition for the co-prime's product]`

| Deal | Account Owner | Product Gate | The Ask | Advances If | Stalls If |
|---|---|---|---|---|---|
| [Account — Opp Name] | [Account Owner = AE name] | [LISTEN → BUILD TRUST]<br>Requires: [specific condition] | [One sentence — not a script] | [What it means if they get it] | [What it means if they don't] |

---

#### 4D. Per-opp deal health — MFJ-grouped

Full C360 treatment applies to IN / UP+ / UP- opps. Pipeline/null opps get a table view only.

**Tone by MFJ group:**
- **IN** — Manager committed. What does the co-prime need to do to close?
- **UP+** — Manager is more bullish than AE. What does the co-prime need to do to justify the manager's confidence?
- **UP-** — Manager is skeptical. Why is the manager downgrading, and what's the evidence gap the co-prime needs to close?

**Full C360 block structure (IN / UP+ / UP-):**

**[Account Name] — [Opp Name] | $[Amount] | Stage [##] | MFJ: [IN/UP+/UP-]**

**AE:** [AE Name] | **Co-Prime's SKUs:** [product names] | **SKU Match:** [Primary / Supporting]
**Last Customer Contact:** [date] ([N] days ago)

**Actual Phase:** [LISTEN / BUILD TRUST / PARTNER] | **Earned?** [Yes / No + one-sentence rationale]

[One-sentence headline on where the co-prime's product motion stands given the MFJ tone.]

**5Cs | [X]/5** (from the co-prime's product lens)

| C | Status | Evidence | Source |
|---|---|---|---|
| C-level Priorities | ✅/🟡/❌ | [relevant to this product area] | [Org62 / Slack date] |
| Challenges | ✅/🟡/❌ | [operational pain specific to the co-prime's product] | |
| Competitive Threats | ✅/🟡/❌ | [known competitors in this product area] | |
| Compelling Events | ✅/🟡/❌ | [time-bound trigger for this product specifically] | |
| Potential Crises | ✅/🟡/❌ | [risks if co-prime's product motion stalls] | |

**What We Have**

| Activity | Evidence | Source |
|---|---|---|
| [e.g. "POV delivered"] | [Slack date] | [Slack] |

**What's Missing**

| Activity | Why It Matters |
|---|---|
| [e.g. "No named IT buyer"] | [why this specifically blocks the motion] |

**Recommendations**

| Gap | Action | Owner | By |
|---|---|---|---|
| [gap] | [specific action] | [Co-prime / AE / SE / BDR] | [date] |

---

**Pipeline / null table (no MFJ set):**

Group all null/Pipeline opps in a stage-sorted table. No C360 depth. Flag any with close date within 90 days or >60 days dark.

| Account | AE | Opp | Amount | CCY | Stage | Dark | Close | ★ |
|---|---|---|---|---|---|---|---|---|
| [account] | [AE] | [opp (linked)] | $[X]K | USD/CAD | 02 | [N]d | [date] | ★ if this is one of your AEs |

---

### Step 5: Assemble Canvas → `[date]-canvas.md` + Slack

Read all step files and assemble the canvas in presentation order. Then create in Slack.

---

## Canvas Structure

**Title:** `[Co-Prime Name] — Co-Prime Brief | [Mon DD, YYYY]`

The canvas is co-prime-lens first throughout. AE names appear only as context, never as the organizing axis. The primary structure is Manager Forecast Judgement (MFJ). Full C360 deal health only for opps where MFJ is set (IN / UP+ / UP-). All other opps get a stage-sorted table view with the co-prime's role noted.

Every column header, outreach row, and deal health block answers: **what is the co-prime's role here, and what should they do next?** — not "what is the AE doing?"

```markdown
# [Co-Prime Name] — Co-Prime Brief | [Mon DD, YYYY]
**Co-prime:** [Name] ([Role]) | **Territory:** [AE1 / AE2 / AE3 / ...]  | **Generated:** [Mon DD, YYYY]

### *This canvas was generated using AI, which can produce inaccurate or harmful responses. Review for accuracy and safety before using.*

---

# :bar_chart: Pipeline Snapshot

| Metric | Value |
| --- | --- |
| Total qualifying opps (clean) | [N] |
| Total pipeline (approx USD equiv) | ~$[X]M |
| Key tier (≥$100K) | [N] opps |
| Significant tier ($50K–$99K) | [N] opps |
| Active (<60 days dark) | [N] opps |
| Dark (>60 days) | [N] opps |
| No CRM activity recorded | [N] opps |
| Closing in <90 days | [list opp names + dates] |
| Proactive pipeline signal | [any co-prime-generated pipeline events, e.g. field events] |

---

# :mag: Deal Groups — Manager Forecast Judgement

| Group | Opps | Approx Value | Treatment |
| --- | --- | --- | --- |
| MFJ = IN (manager committed) | [N] | ~$[X]M | Full C360 5Cs assessment |
| MFJ = UP+ (manager more bullish than AE) | [N] | ~$[X]M | Full C360 5Cs assessment |
| MFJ = UP- (manager skeptical / downgrading) | [N] | ~$[X]M | Full C360 5Cs assessment |
| MFJ = OUT (manager pulled from forecast) | [N] | ~$[X]M | Pulse check table — flag each with reason if known |
| No MFJ (not yet manager-reviewed) | [N] | ~$[X]M | Pulse check table |

---

# :crystal_ball: Forecast Call Prep — MFJ Deals

Only MFJ-set opps. These are the deals the manager has an opinion on — co-prime's product motion must be ready to answer for.

| Opportunity | [Product] SKUs | C360 Phase | Gate / Next Action | Account Owner | [Co-Prime]'s Co-Prime Role | Gap | Ask | Action |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| [Account — Opp Name ($Amount, Stage ##) — linked] | [compact SKU string] | [LISTEN / BUILD TRUST / PARTNER] ⚠️ if stage not earned | [what this product motion needs to advance to next phase] | [Account Owner = AE name] | [specific co-prime role on this deal: POV author / re-engage architect / not yet engaged / blocked / etc.] | [the specific gap that blocks the co-prime's product motion] | [The question to ask in the forecast call] | [Specific time-bound action — owner + date] |

> **C360 column guidance for canvas tables:**
> - IN / UP+ tables: two columns — `C360 Phase` (phase + ⚠️) and `Gate / Next Action` (one-line gate condition)
> - UP- table: single `C360` column (phase + ⚠️ only) — table is already wide; no gate column

---

# :eyes: No-MFJ Deals — Pulse Check

All opps where MFJ is null. Stage-sorted, highest amount first. No C360 depth. Flag: 🔴 = close date within 90 days or >60 days dark.

| Opportunity | [Product] SKUs | Account Owner | Amount | [Co-Prime]'s Co-Prime Role | Last Touch | Note |
| --- | --- | --- | --- | --- | --- | --- |
| [Account — Opp Name (linked)] | [compact SKU string] | [Account Owner = AE name] | $[X]K [CCY] | [specific role or "No visibility"] | [date (N days)] | [one-line note on status or next step] |

---

# :phone: This Week's Outreach

MFJ opps dark >30 days + No-MFJ opps dark >60 days. The co-prime is often better positioned to re-engage on a product angle than the AE on a relationship angle.

| Contact / Account | Account Owner | [Co-Prime]'s Co-Prime Role | Opportunity | Hook | Via | Owner |
| --- | --- | --- | --- | --- | --- | --- |
| [Named person or AE DM] | [Account Owner = AE name] | [specific co-prime role] | [Opp name + amount — linked] | [specific hook — not "checking in"] | [Email / Slack DM / Call] | [Co-prime / AE / Co-prime proposes, AE sends] |

---

# :question: The One Question — MFJ Deals

One question per MFJ opp that either advances the co-prime's product motion or surfaces a fatal flaw fast.

| Deal | Account Owner | Gate | The Question | Advances If | Stalls If |
| --- | --- | --- | --- | --- | --- |
| [Account — Opp Name (linked)] | [Account Owner = AE name] | [what must be true for this product motion to progress] | [one sentence — not a script] | [what it means if they get it] | [what it means if they don't] |

---

# :stethoscope: Deal Health — Manager IN

> Manager has committed these. What does the co-prime need to do to close from their product angle?

---

### [Account Name] — [Opp Name] | $[Amount] | Stage [##] | MFJ: IN

**[Co-Prime]'s co-prime role:** [specific role — POV author / discovery co-lead / not yet engaged / etc.] | **Account Owner:** [AE name] | **Last customer contact:** [date] ([N] days)

[One headline: where the co-prime's product motion stands and the key blocker.]

**5Cs Assessment:**

| C | Status | Evidence |
| --- | --- | --- |
| C-level Priorities | ✅/🟡/❌ [label] | [evidence specific to this product area] |
| Challenges | ✅/🟡/❌ [label] | |
| Competitive Threats | ✅/🟡/❌ [label] | |
| Compelling Events | ✅/🟡/❌ [label] | |
| Potential Crises | ✅/🟡/❌ [label] | |

- **What We Have:** [bullet list]
- **What's Missing:** [bullet list]
- **Actions:** [bullet list — specific, time-bound, owner named]

---

[Repeat per IN opp]

---

# :chart_with_upwards_trend: Deal Health — Manager UP+

> Manager is more bullish than the AE. What does the co-prime need to do to justify the manager's confidence?

[Same full block structure as IN section — one block per UP+ opp]

---

# :chart_with_downwards_trend: Deal Health — Manager UP-

> Manager is skeptical / downgrading. Why? What's the evidence gap the co-prime needs to close?

[Same full block structure as IN section — one block per UP- opp]

---

# :rotating_light: Co-Sell Signals

Accounts where the co-prime's product conversation hasn't started yet but signals suggest it should. Warm inbound, AE flags, account activity without co-prime footprint.

[One entry per signal — account, source, signal, action]

---

# :no_entry_sign: Excluded (Bundle / Renewal)

| Account | Account Owner | Opportunity | Amount | Reason |
| --- | --- | --- | --- | --- |
| [account] | [Account Owner = AE name] | [opp (linked)] | $[X]K | [Bundle contamination / Renewal — one line] |

---

*After your sync, run `/se-co-prime-debrief` to capture what changed and close the loop.*

---
```

---

## Post-Sync

After meeting with the co-prime, run `se-co-prime-debrief` with your meeting notes. It reads the files written by this skill as baseline and produces a delta canvas: commitments made, deals advanced or at risk, and anything that didn't come up that should have.

---

## Rules

- **Never filter by AE or territory.** The Org62 query pulls ALL open opportunities where the co-prime is on the OpportunityTeamMember with a matching product — no AE name filter, no OwnerId filter, no territory scope. Your AEs may appear prominently in the output, but the query must not be pre-filtered to them.
- **Co-prime lens is the organizing axis — not AE names.** Every column header, row label, and deal health block answers: what is the co-prime's role on this deal, and what should they do next? AE names appear only as supporting context (who to DM, who to loop in). Never structure a table with AE as the primary grouping or as a primary column.
- **MFJ grouping is the primary deal structure.** Canvas sections: Pipeline Snapshot → Deal Groups table → Forecast Call Prep (MFJ only) → No-MFJ/OUT Pulse Check table → This Week's Outreach → The One Question (MFJ only) → Deal Health IN → Deal Health UP+ → Deal Health UP- → Co-Sell Signals → Excluded table. Full C360 depth only for IN/UP+/UP- opps. OUT and null opps go in the pulse check table — no C360 blocks. OUT opps should note the manager's pull reason if available. Valid MFJ picklist values: `IN`, `UP+`, `UP -`, `OUT`. Query Org62 for the `Manager_Forecast_Judgement__c` field on all qualifying opps before writing any canvas section.
- **Link every opp.** Every opportunity reference must hyperlink: `[Opp Name](https://salesforce.lightning.force.com/lightning/r/Opportunity/[OppID]/view)`
- **Co-prime lens only.** This brief is written for the co-prime to act from. The 5Cs, What We Have, and What's Missing are assessed from the product angle — not the full deal. A green C-level Priorities means the customer's exec priorities connect to this specific product area.
- **SKU match quality matters.** A co-prime on an ATM with no line items is a Potential signal, not an active deal. Don't conflate the two.
- **Dark is dark.** If a co-prime's product is on an opp that's been dark for 60+ days, surface it in both the Dark Opps table AND This Week's Outreach. The co-prime is often better positioned to re-engage on a product angle than the AE on a relationship angle.
- **Co-sell Signals are forward-looking.** The final section surfaces accounts where the co-prime's product conversation hasn't started yet but signals suggest it should. This is the most proactive section.
- **One action per opp — the highest-leverage one.** If there are two, pick the one that either unblocks the deal or surfaces a fatal flaw fast.
- **Platform family disambiguation.** Family = `Platform` covers both the Employee Service co-prime AND the Platform/Security co-prime. When running a brief for the Employee Service co-prime, filter to Name LIKE `%Employee Productivity%`, `%IT Service Center%`, `%Workplace Command Center%`. When running for the Platform/Security co-prime, filter to Name LIKE `%Shield%`, `%Security Center%`, `%Platform Encryption%`, `%Backup%`, `%Privacy Center%`, `%Data Mask%`, `%Salesforce Connect%`, `%Identity for%`.
- **Sales Cloud family disambiguation.** The Revenue Cloud co-prime, the SPM co-prime, and the Partner Cloud co-prime all have Family = `Sales Cloud`. Always use the Name LIKE patterns from `config.specialist_roster.filter_value` — never filter by family alone for these three.
- **Bundle contamination check (critical).** Multi-cloud bundles include multiple product families as line items. An opp named after a different cloud's bundle is NOT this co-prime's deal even if their product appears as a line item — it belongs to whichever co-prime or core AE owns that named motion. Before surfacing any opp in a brief: check that the opp name or the top line item actually corresponds to the co-prime's product area. If the co-prime's SKU is a subordinate component in someone else's named motion, exclude the opp or tag it Supporting (bundle) with a verification note. Never surface it in Recommended Actions.
- **No hardcoded names, roles, or Salesforce User IDs.** Everything about who the co-primes are comes from `config.specialist_roster` — never hardcode a roster in this skill (this repo is public).

## Conversation Starters

- "Co-prime brief for [Co-Prime Name]"
- "Brief for [Co-Prime Name]"
- "[Co-Prime Name] brief"
- "Co-prime brief" (ask which co-prime if not specified)
