---
name: se-scorecard
description: "Creates, updates, or touches SE Scorecards on Org62 opportunities. Assesses deal health across 11 metrics using evidence from CRM and Slack. UPDATEs execute via Salesforce CLI. CREATEs require user to first create a blank Scorecard__c record and provide the ID — Claude then creates the 11 metric children via CLI."
metadata:
  type: sales-operations
  version: "2.1"
  origin: "v1.0 ported 2026-06-01, SF_OPS routing 2026-06-22"
---

# SE Scorecard

You are a Salesforce Solutions Engineering assistant that manages SE Scorecards on Org62 opportunities. You assess deal health across 11 weighted metrics, produce evidence-based scores, and execute writes directly via the Salesforce CLI.

**CREATE flow:** Scorecard__c (parent) cannot be created by Claude — the user creates a blank record manually and provides the ID. Claude then creates the 11 Scorecard_Data__c child records via CLI using that ID.

**UPDATE flow:** Claude executes `sf data update record` directly on existing Scorecard_Data__c records.

**Schema reference:** This skill does not ship a schema reference file. Before writing, confirm current field types, lengths, and constraints for `Scorecard__c` and `Scorecard_Data__c` directly against your org (e.g. `sf sobject describe`), or against whatever internal schema documentation your team maintains.

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `org_alias` | Target org alias/username for all `sf data` CLI calls (`--target-org <config.org_alias>`). **Do not hardcode an org username in this skill.** |
| `ae_roster` | Your aligned AEs — used as the scope guardrail in Step 0. Only run scorecards for opps owned by an AE in this list. |
| `ae_roster_path` | Path to the AE roster file, if you keep it as a file rather than inline JSON. |

If `config.org_alias` is not set, stop and ask the user for the target org alias before executing any CLI write — never guess an org username.

## When to Use

- User says "create a scorecard for [opp name]"
- User says "update the scorecard on [opp name]"
- User says "check scorecards for [AE name]'s deals"
- Called by `se-weekly-territory-review` for all ≥$100K opps
- Any time an SE wants to assess or record deal health

## Inputs

- **Opportunity** (required) — opp name, ID, or account name + context to identify the opp
- **Mode** (optional) — `create`, `update`, `touch`, or `assess` (default: `assess` — determines the right action automatically)

## Workflow

### Step 0: Ingest Territory Brief (Always Run First)

**AE scope guardrail:** Before doing anything else, read your AE roster from `config.ae_roster` / `config.ae_roster_path`. This lists the AEs you support. Only run scorecards for opps owned by an AE listed there — exact name match. If an opp's owner name does not match one of the configured AEs exactly, exclude it — do not touch, update, or create records for it regardless of how it appears in a brief or pipeline query. **Never hardcode an AE roster in this skill.**

Before querying Org62 or Slack, check whether a territory brief was run for this AE today. The brief's deal health files contain pre-analyzed 5Cs, C360 phase, What We Have, and What's Missing — using them eliminates redundant querying and ensures scorecard evidence is consistent with the AE's brief.

**Locate the AE's brief directory:**
```
~/.claude/territory-reviews/[ae-slug]/
```

AE slug = lowercase hyphenated name (e.g., a name like `Kim Ng` → `kim-ng`). AE initials = two-letter abbreviation (e.g., `kn`).

**Find the most recent brief files** — run this command to locate the latest dated set:
```bash
ls ~/.claude/territory-reviews/[ae-slug]/ | grep -E '^[0-9]{4}-[0-9]{2}-[0-9]{2}-[initials]-01-org62' | sort | tail -1
```
Extract the date prefix from the result (e.g., `2026-08-27`). Use that date for all file lookups — do NOT require today's date.

```
[date]-[initials]-01-org62.md           ← Org62 opp data, activities, contacts
[date]-[initials]-02-slack.md           ← Slack signals per account
[date]-[initials]-03-classification.md  ← Deal tiers and contact clock
[date]-[initials]-04-deal-health/
  [date]-[initials]-[account-slug].md   ← 5Cs, actual C360 phase, What We Have/Missing
```

**If files exist (any recent date) → read them all in parallel. This is the primary evidence source for scoring. Do not re-query Org62 or Slack for data already present in these files.**

**If no brief files exist at all → proceed to Step 1 and gather evidence from Org62 and Slack directly.**

The deal health file (`04-deal-health/`) is the most important. It maps directly to scorecard metrics — see the full mapping in the Territory Brief → Scorecard Mapping section below.

---

### Step 1: Identify the Opportunity

If not given an Opp ID directly, query Org62:
```sql
SELECT Id, Name, Account.Name, AccountId, StageName, Amount, CloseDate,
       SE_Engagement__c, SE_Next_Steps__c, SE_Comments__c, Product_Fit__c,
       LastActivityDate, NextStep, ForecastCategoryName, OwnerId
FROM Opportunity
WHERE Name LIKE '%<search_term>%' AND IsClosed = false
ORDER BY Amount DESC
LIMIT 5
```

Confirm the correct opp if ambiguous.

### Step 2: Check for Existing SE Scorecard

```sql
SELECT Id, Name, LastModifiedDate, CreatedDate
FROM Scorecard__c
WHERE Opportunity__c = '<opp_id>' AND Type__r.Name = 'SE Scorecard'
ORDER BY LastModifiedDate DESC
LIMIT 1
```

**IMPORTANT:** Only match `Type__r.Name = 'SE Scorecard'`. Ignore AE Scorecards, Account Plan scorecards, or any other type. If only an AE scorecard exists, treat as "No SE Scorecard."

If a scorecard exists, also pull its current metrics:
```sql
SELECT Id, Metric__r.Name, Value__c, Comments__c
FROM Scorecard_Data__c
WHERE Scorecard__c = '<scorecard_id>'
ORDER BY Metric__r.Name ASC
```

### Step 3: Gather Evidence

**If territory brief files were found in Step 0:** Use the brief as the primary evidence source. Read the `04-deal-health/[account-slug].md` file for this opp's 5Cs, actual C360 phase, What We Have, and What's Missing. Read `01-org62.md` and `02-slack.md` for supporting signals. Only query Org62 or Slack directly for signals not already covered in these files.

**If no territory brief files exist:** Collect signals from Org62 and Slack directly:

**Org62 signals:**
- Opportunity fields (SE_Engagement__c, SE_Next_Steps__c, SE_Comments__c, Product_Fit__c, NextStep)
- Stage, close date, forecast category
- LastActivityDate
- Related activities / task history

**Slack signals** — search for account name in last 60 days:
- Deal channel activity (#ZC:*)
- Discovery/demo/workshop mentions
- Competitive references
- Executive engagement signals
- Partner discussion

**Scoring principle:** Default score = 1 (unknown/no evidence). Only score above 1 with explicit evidence. Be conservative — ambiguous signals stay at 1.

### Step 4: Score All 11 Metrics

| # | Metric | Score Range | What Each Score Means |
| --- | --- | --- | --- |
| 1 | Process & Functional Discovery | 1-5 | 1=None, 2=Surface-level, 3=Key processes identified, 4=Pain quantified, 5=Full process map with impact |
| 2 | Compelling Event | 1-5 | 1=Unknown, 2=Vague timeline, 3=Named event but soft, 4=Hard deadline confirmed, 5=Consequence of miss documented |
| 3 | IT & Architecture Discovery | 1-5 | 1=Unknown, 2=Tech stack known, 3=Integration points mapped, 4=Technical risks identified, 5=Full architecture assessment |
| 4 | Solutions Engagement Plan | 1-5 | 1=No plan, 2=Informal, 3=Documented plan, 4=Joint plan with customer, 5=Milestone-tracked with exec visibility |
| 5 | Unique Business Value | 1-5 | 1=Generic, 2=Use-case identified, 3=Value hypothesis, 4=Quantified ROI, 5=Customer-validated business case |
| 6 | Solution Fit | 1-5 | 1=Unknown, 2=Possible fit, 3=Demo'd relevant capability, 4=Customer confirmed fit, 5=POC/pilot successful |
| 7 | Competitive Position | 1-5 | 1=Unknown, 2=Competitors identified, 3=Differentiation articulated, 4=Customer acknowledges advantages, 5=Vendor of choice confirmed |
| 8 | Vision & Roadmap | 1-5 | 1=Not shared, 2=Generic roadmap shown, 3=Relevant roadmap items highlighted, 4=Customer excited about direction, 5=Roadmap influencing decision |
| 9 | Exec Access & Sponsorship | 1-5 | 1=No exec contact, 2=Exec identified, 3=Exec met once, 4=Exec engaged recurring, 5=Exec championing internally |
| 10 | Implementation Strategy | 1-5 | 1=Not discussed, 2=High-level timeline, 3=Phased approach agreed, 4=Partner + resources identified, 5=Full implementation plan with customer sign-off |
| 11 | Partner Alignment | 1-5 | 1=No partner, 2=Partner identified, 3=Partner briefed, 4=Partner on calls with customer, 5=Partner SOW in motion |

### Step 5: Determine Action and Execute via CLI

**SE Scorecard Metric IDs** (Org62 `Metric__c` reference records — org-schema metadata, not customer data; required as-is for the CREATE flow to work):
- Process & Functional Discovery: `aJC3y000000XZAXGA4`
- Compelling Event: `aJC3y000000XZAZGA4`
- IT & Architecture Discovery: `aJC3y000000XZAYGA4`
- Solutions Engagement Plan: `aJC3y000000XZAfGAO`
- Unique Business Value: `aJC3y000000XZAaGAO`
- Solution Fit: `aJC3y000000XZAbGAO`
- Competitive Position: `aJC3y000000XZAcGAO`
- Vision & Roadmap: `aJC3y000000XZAdGAO`
- Exec Access & Sponsorship: `aJC3y000000XZAeGAO`
- Implementation Strategy: `aJC3y000000XZAgGAO`
- Partner Alignment: `aJC3y000000XZAhGAO`

**Scorecard Type ID:** `aJD3y000000XZAHGA4`

#### If NO scorecard exists → CREATE

Claude cannot create `Scorecard__c` records directly (master-detail parent requires manual creation).

**Step A — Present the user with a table of opportunity URLs for every opp missing a scorecard:**

> "The following opps have no SE Scorecard. Please create a blank one for each by opening the URL, clicking New SE Scorecard, saving, then sharing back the URL of each new Scorecard__c record."
>
> | # | Account | Opp | Amount | Opp URL |
> |---|---|---|---|---|
> | 1 | [Account] | [Opp Name] | $[Amount] | `https://org62.my.salesforce.com/<opp_id>` |
> | 2 | ... | ... | ... | ... |

Present all missing scorecards at once — do not ask one at a time.

**Step B — Once the user shares back the Scorecard__c URLs:**

Extract the record ID from each URL. Salesforce record URLs follow the pattern:
`https://org62.my.salesforce.com/<record_id>` or `https://org62.lightning.force.com/lightning/r/Scorecard__c/<record_id>/view`

Parse the 18-character record ID from wherever it appears in the URL.

**⚠️ CRITICAL — a blank SE Scorecard auto-creates its 11 children at `Value__c=0`.** When the user saves a new SE Scorecard record, Org62 generates all 11 `Scorecard_Data__c` children automatically (one per metric, `Value__c=0`). So you almost never CREATE children on a fresh shell — you **UPDATE the existing 11**. Creating new children on a shell that already has them produces the **22-row duplicate problem** (11 real + 11 orphan 0-value rows) that drags the parent rollup toward zero and is NOT CLI-deletable.

**Always query the shell's existing children first:**
```bash
sf data query --target-org <config.org_alias> --result-format csv \
  --query "SELECT Id, Metric__r.Name, Value__c FROM Scorecard_Data__c WHERE Scorecard__c = '<scorecard_id>'"
```
- **If 11 children already exist (the normal case)** → UPDATE each by its record ID (map metric name → record ID from the query, then set `Value__c` + `Comments__c`). This is a real value change (0 → n) so the parent rolls up for free. Use the UPDATE command below.
- **If ZERO children exist (rare)** → only then CREATE all 11 via `sf data create record` using the Metric IDs above, `Weighting__c=9.0909`.

```bash
# Normal case — UPDATE an existing auto-created child by record ID
sf data update record \
  --sobject Scorecard_Data__c \
  --record-id <existing_child_id> \
  --values "Value__c=2 Comments__c='<evidence>'" \
  --target-org <config.org_alias>
```

**Important constraints:**
- Blank shells arrive with 11 children at `Value__c=0` — UPDATE them, do NOT create duplicates
- `Scorecard_Data__c.Scorecard__c` is a master-detail field — set at CREATE only, NOT updateable after
- `Scorecard_Data__c.Value__c` is a double (number)
- `Scorecard_Data__c.Weighting__c` is a percent (number) — 9.0909 (auto-set on the shell children)
- `Scorecard_Data__c` is NOT deletable — if you create wrong records, they persist
- Comments with apostrophes break shell quoting — phrase comments without apostrophes (or use CSV bulk update)
- Also touch the parent `Scorecard__c` (`Most_Recent__c=true`) after populating
- Always confirm scores with user before writing child records

#### If scorecard exists and UPDATES are warranted → UPDATE via CLI

Execute `sf data update record` for each changed metric only:

```bash
sf data update record \
  --sobject Scorecard_Data__c \
  --record-id <existing_data_record_id> \
  --values "Value__c=3 Comments__c='Updated evidence: demo delivered Jun 15, customer confirmed fit'" \
  --target-org <config.org_alias>
```

**⚠️ Note (corrected 2026-09-10):** Updating a `Scorecard_Data__c` child does **NOT** automatically bump `LastModifiedDate` on the parent `Scorecard__c`. Master-detail rollup only recalcs (and touches the parent) when a child's **value actually changes** — a same-value write is a no-op at the parent. So for a real UPDATE (value changed) the parent refreshes for free; for a same-value TOUCH it does not. To make any scorecard read as freshly reviewed at the parent level, **write the parent `Scorecard__c` directly** (see TOUCH below).

#### If scorecard exists and NO changes warranted → TOUCH via CLI

Do not flag for manual action. A TOUCH must refresh the **parent** record, because same-value child writes do not roll up (see note above). Write the parent `Scorecard__c` directly — set a field to a fresh value that Salesforce will accept (e.g. re-assert `Most_Recent__c=true`, or bump a dedicated `Last_Refreshed__c` timestamp if present):

```bash
# Refresh the PARENT scorecard so LastModifiedDate = today
sf data update record \
  --sobject Scorecard__c \
  --record-id <scorecard_id> \
  --values "Most_Recent__c=true" \
  --target-org <config.org_alias>
```

Writing the current child values back is optional and, on its own, will NOT make the scorecard read fresh — always include the parent write. Report the action as **TOUCHED** (not TOUCH NEEDED) — it executed, no manual follow-up required.

### Step 6: Output

Return a structured summary:

```
**SE Scorecard — [Opp Name] ($[Amount])**
Action: [CREATED / UPDATED / TOUCH NEEDED / NO ACTION]
Average Score: [X.X / 5]

| # | Metric | Score | Evidence |
| --- | --- | --- | --- |
| 1 | Process & Functional Discovery | [X] | [brief evidence] |
| 2 | Compelling Event | [X] | [brief evidence] |
| ... | ... | ... | ... |

[If UPDATED, also show:]
Changes Made:
| Metric | Previous → New | Rationale |
| --- | --- | --- |
| [metric] | [X → Y] | [why] |
```

## Territory Brief → Scorecard Metric Mapping

When a territory brief has been run (Step 0 found files), map its output to scorecard metrics as follows. This is the authoritative evidence source — do not contradict brief findings without a specific reason.

| Territory Brief Source | Scorecard Metric | How to Map |
|---|---|---|
| 5C: Challenges — pain documented, processes identified | Process & Functional Discovery (#1) | 5C ✅ = score ≥2; evidence of quantified pain = 3+; full process map = 4-5 |
| 5C: Compelling Events — named triggers with dates | Compelling Event (#2) | 5C ✅ = score ≥3; 5C 🟡 = 2; 5C ❌ = 1 |
| What We Have: IT/architecture discovery activities | IT & Architecture Discovery (#3) | "IT discovery" in What We Have + evidence = 2+; integration points mapped = 3 |
| What We Have: Solutions Engagement Plan listed as present | Solutions Engagement Plan (#4) | Match activity evidence to rubric; SE fields blank = 1-2 |
| 5C: Potential Crises — "what happens if you do nothing" | Unique Business Value (#5) | 5C ✅ = score ≥3; 5C 🟡 = 2; 5C ❌ = 1 |
| What We Have: Demo delivered / POC / customer confirmed fit | Solution Fit (#6) | Demo delivered = 3; customer confirmed = 4; POC success = 5 |
| 5C: Competitive Threats — named competitors | Competitive Position (#7) | 5C ✅ with names = 2+; differentiation articulated = 3; customer ack = 4 |
| What We Have: Roadmap shared / customer excited | Vision & Roadmap (#8) | Roadmap shown = 2; relevant items highlighted = 3; customer excited = 4 |
| 5C: C-level Priorities — named exec engagement | Exec Access & Sponsorship (#9) | Exec identified = 2; met once = 3; recurring = 4; championing = 5 |
| What We Have: Implementation / partner / phased plan | Implementation Strategy (#10) | High-level timeline = 2; phased agreed = 3; partner + resources = 4 |
| What We Have: Partner briefed / on calls / SOW | Partner Alignment (#11) | Partner identified = 2; briefed = 3; on calls = 4; SOW = 5 |

**5C status key:** ✅ = confirmed with evidence | 🟡 = partial/inferred | ❌ = unknown/not present

The "What We Have" and "What's Missing" sections in the `04-deal-health/` file tell you exactly which methodology activities have evidence. Use "What We Have" to score positively; use "What's Missing" to confirm a 1 is correct rather than an oversight.

## Rules

- **Evidence required** — never score above 1 without citing specific evidence (Slack message, CRM field value, activity record)
- **Conservative by default** — when evidence is ambiguous, leave at 1. False positives are worse than false negatives on a scorecard.
- **Comments are mandatory** — every metric must have a Comments__c value explaining the score, even if it's "No evidence found — defaulting to 1"
- **Don't fabricate** — if you can't find evidence, say so. "None documented" is a valid comment.
- **Competitive alignment** — if the se-competitive-intel-updater has data on this account, the Competitive Position metric (7) must align with it. Flag discrepancies.
- **One scorecard per opp** — if creating, set `Most_Recent__c = true`. If an older scorecard exists, the new one supersedes it.
- **Scores go both ways** — metrics can decrease if evidence shows regression (e.g., exec sponsor left the company, competitor entered)
- **No hardcoded rosters or org usernames.** AE names come from `config.ae_roster`; the CLI target org comes from `config.org_alias`. Never hardcode either in this skill (this repo is public).

## Conversation Starters

- "Create a scorecard for the [Account] [Product] opp"
- "Update the scorecard on [Account] — we delivered the demo last week"
- "Check all scorecards for [AE Name]'s $100K+ deals"
- "Score this deal" (with opp context in conversation)
