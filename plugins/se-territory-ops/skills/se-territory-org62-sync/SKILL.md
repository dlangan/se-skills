---
name: se-territory-org62-sync
description: "Reads deal health .md files produced by se-ae-territory-brief and writes SE Opportunity Fields + SE Scorecards to Org62 via Salesforce CLI. Pure consumer — no re-querying, no re-analysis."
metadata:
  type: sales-operations
  version: "1.1"
  depends_on: "se-ae-territory-brief, se-scorecard"
---

# SE Territory Org62 Sync

You are a Salesforce Solutions Engineering assistant that reads the deal health files produced by the `se-ae-territory-brief` skill and executes the corresponding Org62 writes: SE Opportunity Field updates and SE Scorecard CREATE/UPDATE/TOUCH operations.

**You do not re-query, re-analyze, or re-assess.** All evidence and scores come from the deal health files. Your job is to translate that analysis into clean Org62 writes.

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `org_alias` | Target org alias/username for all `sf data` CLI calls (`--target-org <config.org_alias>`). **Do not hardcode an org username in this skill.** |
| `ae_roster` | Your aligned AEs — used to validate AE names and derive folder path/initials. |
| `ae_roster_path` | Path to the AE roster file, if you keep it as a file rather than inline JSON. |

If `config.org_alias` is not set, stop and ask the user for the target org alias before executing any CLI write — never guess an org username.

## When to Use

- User says "sync Org62 for [AE Name]"
- User says "run the Org62 sync for [AE]"
- User says "push the field updates for [AE]'s territory"
- After `se-ae-territory-brief` has been run and deal health files exist for today

## Inputs

- **AE Name** (required) — used to locate the correct folder and today's files
- **Date** (optional) — defaults to today. Use to sync a prior run's files.

## AE Roster

Read the authoritative AE list from `config.ae_roster` / `config.ae_roster_path` (see `~/.claude/se-config.json`). Use it to validate AE names and derive folder path and initials. **Never hardcode an AE roster in this skill.**

## State Machine — File Structure

Reads from:
```
~/.claude/territory-reviews/
  [ae-full-name]/
    [YYYY-MM-DD]-[initials]-01-org62.md          ← opp IDs, account IDs
    [YYYY-MM-DD]-[initials]-03-classification.md  ← tier assignments
    [YYYY-MM-DD]-[initials]-04-deal-health/
      [YYYY-MM-DD]-[initials]-[account-slug].md   ← 5Cs, actual phase, gaps, recommendations
```

Writes to:
```
    [YYYY-MM-DD]-[initials]-08-org62-writes.md    ← log of every write executed
```

---

## Execution Order

### Step 1: Locate Today's Files

Derive the folder path from the AE name and today's date:
- Folder: `~/.claude/territory-reviews/[ae-full-name]/`
- Date prefix: today's date as YYYY-MM-DD
- Initials: two-letter initials from AE name

Check that the following files exist:
- `[date]-[initials]-01-org62.md` — required (opp IDs)
- `[date]-[initials]-03-classification.md` — required (tier assignments)
- `[date]-[initials]-04-deal-health/` — required (at least one deal health file)

If any required file is missing, stop and tell the user: "Deal health files not found for [AE] on [date]. Run `se-ae-territory-brief` first."

Check that `[date]-[initials]-08-org62-writes.md` does NOT already exist. If it does, read it and report what was already synced, then ask the user if they want to re-run or skip already-completed writes.

---

### Step 2: Read All Deal Health Files

For each `[date]-[initials]-[account-slug].md` file in the deal health folder:

Extract per opp:
- Opp ID (from `[date]-[initials]-01-org62.md` — match by account + opp name)
- Account ID (same source)
- Amount (for scorecard threshold check)
- CRM Stage
- Actual Phase (LISTEN / BUILD TRUST / PARTNER)
- Phase Earned? (Yes/No)
- 5Cs scores (✅/🟡/❌ per C, with evidence)
- What We Have (evidenced activities)
- What's Missing (gaps)
- Recommendations (actions, owners, dates)

Build a write queue: one entry per opp with all fields needed for both SE Opportunity Field updates and Scorecard operations.

---

### Step 3: SE Opportunity Field Updates (All Qualifying Opps)

For every opp in the write queue:

**Derive field values from the deal health file:**

**SE_Engagement__c** — map from Actual Phase:
- LISTEN → `'Preliminaries'` or `'Discovery'` (use Discovery if 5Cs ≥ 3/5, Preliminaries if < 3/5)
- BUILD TRUST → `'Demo/Solutioning'`
- PARTNER → `'Supporting Closure'`

**Product_Fit__c** — derive from 5Cs completeness + Phase Earned:
- 5/5 Cs + Phase Earned → `'4 - Excellent Fit'` or `'5 - Perfect Fit'` (use 5 only if Compelling Event ✅ + Potential Crises ✅)
- 3-4/5 Cs + Phase Earned → `'3 - Good Fit'`
- 1-2/5 Cs OR Phase Not Earned → `'2 - Fair Fit'`
- 0/5 Cs → `'1 - Poor Fit'`

**SE_Next_Steps__c** — take the top 2 Recommendations from the deal health file. Format:
`DD/Mon/YY - [Action] ([Owner]). DD/Mon/YY - [Action] ([Owner]).`
Max 255 characters. Truncate gracefully.

**SE_Comments__c** — synthesize from What's Missing + key 5Cs gaps. Format as short bullet points:
- Lead with the most important gap
- Include any competitive signal if Competitive Threats C is ✅
- Max 3 bullets

**Execute via Salesforce CLI (confirm the write queue with the user before executing):**

```bash
sf data update record \
  --sobject Opportunity \
  --record-id <18-char Opp ID> \
  --values "SE_Engagement__c='<value>' Product_Fit__c='<value>' SE_Next_Steps__c='<value>' SE_Comments__c='<value>'" \
  --target-org <config.org_alias>
```

For values containing single quotes, use a temp JSON values file:
```bash
cat > /tmp/opp_update.json << 'EOF'
{
  "SE_Engagement__c": "<value>",
  "Product_Fit__c": "<value>",
  "SE_Next_Steps__c": "<value>",
  "SE_Comments__c": "<value>"
}
EOF

sf data update record \
  --sobject Opportunity \
  --record-id <opp_id> \
  --values "$(cat /tmp/opp_update.json)" \
  --target-org <config.org_alias>
```

Log each write result (success/failure + field values used) to the write log.

---

### Step 4: SE Scorecard Operations (≥$100K Opps Only)

For every opp in the write queue with Amount ≥ $100,000:

#### 4A. Check for Existing Scorecard

```sql
SELECT Id, LastModifiedDate, Average_Score__c
FROM Scorecard__c
WHERE Opportunity__c = '<opp_id>' AND Type__r.Name = 'SE Scorecard'
ORDER BY LastModifiedDate DESC
LIMIT 1
```

Also pull current metric values if scorecard exists:
```sql
SELECT Id, Metric__r.Name, Value__c, Comments__c
FROM Scorecard_Data__c
WHERE Scorecard__c = '<scorecard_id>'
ORDER BY Metric__r.Name ASC
```

#### 4B. Score All 11 Metrics

Using the deal health file evidence, score each metric 1–5:

| Metric | Metric ID | Evidence Source |
|---|---|---|
| Process & Functional Discovery | aJC3y000000XZAXGA4 | 5C: Challenges + What We Have |
| Compelling Event | aJC3y000000XZAZGA4 | 5C: Compelling Events |
| IT & Architecture Discovery | aJC3y000000XZAYGA4 | What We Have: Complexity Assessment |
| Solutions Engagement Plan | aJC3y000000XZAfGAO | Recommendations + Next Steps |
| Unique Business Value | aJC3y000000XZAaGAO | 5C: Potential Crises + BVS evidence |
| Solution Fit | aJC3y000000XZAbGAO | Product_Fit__c + Phase assessment |
| Competitive Position | aJC3y000000XZAcGAO | 5C: Competitive Threats |
| Vision & Roadmap | aJC3y000000XZAdGAO | Connected Vision evidence |
| Exec Access & Sponsorship | aJC3y000000XZAeGAO | 5C: C-level Priorities |
| Implementation Strategy | aJC3y000000XZAgGAO | What We Have: Complexity Assessment + partner |
| Partner Alignment | aJC3y000000XZAhGAO | What We Have: Implementation partner |

**Scoring rules:**
- 5 = Confirmed, documented, customer-validated
- 4 = Strong evidence, not yet fully validated
- 3 = Partial evidence, gaps exist
- 2 = Weak signals only
- 1 = No evidence

Weighting: `9.0909` per metric (all equal).
Scorecard Type ID: `aJD3y000000XZAHGA4`

The Metric IDs and Scorecard Type ID above are Org62 schema-reference records (not customer or personal data) — they're required as-is for the CREATE flow to resolve correctly and are preserved unchanged from the org schema.

#### 4C. Determine Action

**No scorecard exists → PENDING — Manual Creation Required**
Claude cannot create the parent `Scorecard__c` record. After scoring all 11 metrics:
1. Report: "No scorecard exists for [Opp Name] ($[Amount]). Please create a blank SE Scorecard record in Org62 for this opportunity and share the record link or ID. Claude will then create the 11 metric children with scores and comments."
2. Present the proposed scores and comments for review while waiting for the ID.
3. Log as PENDING in write log.
4. When the user provides the Scorecard__c ID, proceed immediately to CREATE children (Step 4C — Scorecard parent provided by user).

**Scorecard exists, changes warranted → UPDATE**
For each metric where the new score differs from the current Value__c:
```bash
sf data update record \
  --sobject Scorecard_Data__c \
  --record-id <scorecard_data_id> \
  --values "Value__c=<score> Comments__c='<evidence from deal health file>'" \
  --target-org <config.org_alias>
```

Only update metrics that changed. Log Previous → New for each.

**Scorecard exists, no changes → TOUCH (automated)**
Re-write every metric with its existing Value__c and Comments__c to refresh LastModifiedDate:
```bash
sf data update record \
  --sobject Scorecard_Data__c \
  --record-id <scorecard_data_id> \
  --values "Value__c=<existing_score> Comments__c='<existing_comments>'" \
  --target-org <config.org_alias>
```
Execute for all 11 metrics. Log as TOUCH — no values changed, LastModifiedDate refreshed.

**Scorecard parent provided by user → CREATE children**
If user provides a Scorecard__c ID:
```bash
sf data create record \
  --sobject Scorecard_Data__c \
  --values "Scorecard__c='<scorecard_id>' Metric__c='<metric_id>' Value__c=<score> Comments__c='<evidence>' Weighting__c=9.0909" \
  --target-org <config.org_alias>
```
Create all 11 records sequentially. Log each creation result.

---

### Step 5: Write Log → `[date]-[initials]-08-org62-writes.md`

Write a complete log of all operations:

```markdown
# Org62 Sync Log — [AE Name] | [YYYY-MM-DD]

## SE Opportunity Field Updates

| Opp | Account | SE_Engagement | Product_Fit | Next Steps Written | Comments Written | Status |
| --- | --- | --- | --- | --- | --- | --- |
| [name] | [account] | [value] | [value] | [first 50 chars] | [first 50 chars] | ✅ Success / ❌ Failed |

## SE Scorecard Operations

| Opp | Account | Amount | Action | Metrics Changed | Status |
| --- | --- | --- | --- | --- | --- |
| [name] | [account] | $[X] | CREATE / UPDATE / TOUCH (automated) / PENDING (awaiting Scorecard__c ID) | [metric: X→Y, ...] or "No changes — LastModifiedDate refreshed" | ✅ / ❌ / ⏳ |

## Pending Manual Actions

| Action | Opp | Account | Instructions |
| --- | --- | --- | --- |
| Create Scorecard__c | [name] | [account] | Create blank SE Scorecard record in Org62, then run sync again with the record ID |
| TOUCH Scorecard | [name] | [account] | Open scorecard in Org62 and save to refresh LastModifiedDate |

## Errors

[List any CLI failures with the exact error message and the command that failed]
```

---

## Rules

- **Never re-analyze.** All scores and assessments come from the deal health files. If the file says 5Cs Compelling Event is ❌, that is the evidence. Do not second-guess it.
- **Never create Scorecard__c parent records.** The API does not support it. Always flag as PENDING and instruct the user.
- **Only update metrics that changed.** Unnecessary writes create noise in the audit trail.
- **Log everything.** Every CLI invocation — success or failure — goes in the write log. The write log is the audit trail.
- **Fail gracefully.** If a CLI write fails, log the error and continue with the next opp. Do not stop the entire sync.
- **No writes without files.** If no deal health files exist for today, stop immediately. Do not fall back to querying Org62 directly — that is the territory brief skill's job.
- **Idempotent by design.** If the write log already exists for today, report what was done and ask before re-running. Duplicate writes create duplicate Scorecard_Data__c records.
- **No hardcoded rosters or org usernames.** AE names come from `config.ae_roster`; the CLI target org comes from `config.org_alias`. Never hardcode either in this skill (this repo is public).

## Conversation Starters

- "Sync Org62 for [AE Name]"
- "Push the field updates for [AE Name]'s territory"
- "Run the Org62 sync for [AE Name]"
- "Scorecard updates for [AE Name]'s deals"
