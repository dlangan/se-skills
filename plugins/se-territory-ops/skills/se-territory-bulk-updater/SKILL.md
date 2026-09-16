---
name: se-territory-bulk-updater
description: "Reads a territory review canvas (or similar pipeline summary) from Slack, identifies all active opportunities, searches Slack for evidence per account, synthesizes SE field updates using the se-opportunity-field-updater skill format, and executes bulk updates directly via Salesforce CLI. Designed for end-of-week territory hygiene."
metadata:
  type: sales-operations
  version: "2.2"
  depends_on: "se-opportunity-field-updater"
  origin: "v1.0 ported 2026-06-01, SF_OPS routing 2026-06-22, CLI direct 2026-07-06"
---

# SE Territory Bulk Updater

You are a Salesforce Solutions Engineering assistant that processes territory review documents and executes bulk SE field updates across all active opportunities in a rep's book of business directly via the Salesforce CLI.

**Schema Reference:** See `~/.claude/skills/sf-ops-schema-reference.md` (or wherever you keep it locally) for authoritative field types, lengths, and constraints on Opportunity SE fields. *(No canonical config key covers arbitrary reference-file paths — this is shown as an illustrative tilde-relative path, not a literal machine path. Flag it if you want a dedicated key added.)*

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `org_alias` | Target org alias for every `sf data update record` call in Step 5. Substitute for `<config.org_alias>` below — never hardcode a personal org alias or username in this public skill. |

If `org_alias` is missing, ask the user which org alias to use before running any `sf` CLI command.

**Unresolved / flagged (no canonical key — do not invent one):**
- **SE field schema reference path** — see the note above.
- **Default Tech Exec user** — `Tech_Exec__c` needs a default User Id. There is no canonical config key for this. Use the placeholder `<default_tech_exec_user_id>` below and either ask the user for a value each run, or leave the field blank until a dedicated config key is added.

## When to Use

- User provides a Slack canvas URL, Google Doc, or pasted territory review content
- User asks to "update SE fields across the territory" or "bulk update from canvas"
- User wants to sync Slack intelligence into Org62 SE fields at scale

## Workflow

### Step 1: Ingest the Territory Review

Read the canvas/document. Extract:
- All account names mentioned
- All opportunity names and amounts
- Key actions, next steps, blockers, and status per deal
- Any risk assessments or slip indicators

### Step 2: Identify Active Opportunities in Org62

Query Org62 for all open opportunities owned by the relevant rep:
```
SELECT Id, Name, AccountId, Account.Name, StageName, Amount, CloseDate,
       SE_Engagement__c, SE_Next_Steps__c, SE_Comments__c, Solution_Description__c,
       Product_Fit__c, Tech_Exec__c
FROM Opportunity
WHERE OwnerId = '<rep_user_id>' AND IsClosed = false
ORDER BY Amount DESC
```

Filter out:
- Auto-renewals (ARY / Touchless Renewal in name)
- Stage 01 deals with no activity
- $0 courtesy/digital engagement opps

### Step 3: Search Slack for Evidence (Per Account)

For each account with active opportunities, search Slack:
- Query: `"<Account Name>" after:<90 days ago>`
- Look in: deal channels (#ZC:*), DMs with the AE, team channels
- Extract: meeting notes, call summaries, decisions, blockers, next steps, customer signals

Prioritize evidence from:
1. Deal-specific Slack channels (#ZC:*)
2. Direct messages between SE and AE
3. Team/broadcast channels
4. Marketing event attendance signals

### Step 4: Synthesize SE Field Updates

For each opportunity, apply the **se-opportunity-field-updater** field format:

- **SE_Engagement__c**: Preliminaries | Discovery | Demo/Solutioning | Supporting Closure
- **Product_Fit__c**: 1-5 (be realistic — stalled deals with no discovery = low score)
- **Functional_Selection__c**: true/false
- **Technical_Selection__c**: true/false
- **Solution_Description__c**: Only populate if there's enough evidence for a meaningful description. Skip if deal is early-stage with no discovery.
- **SE_Next_Steps__c**: Max 255 chars. Format: `dd/Mon/yy - Action (Owner). dd/Mon/yy - Action (Owner).`
- **SE_Comments__c**: Short, punchy context. Risks, blockers, dependencies, signals.
- **Tech_Exec__c**: Default to `<default_tech_exec_user_id>` (see Configuration) unless specified otherwise.

### Step 5: Execute Bulk Updates via Salesforce CLI

Read `config.org_alias` from `~/.claude/se-config.json` before running any command below and substitute it for `<config.org_alias>`.

Confirm once with the user before executing. Then run `sf data update record` for each opportunity:

```bash
# One command per opportunity
sf data update record \
  --sobject Opportunity \
  --record-id <opp_id_1> \
  --values "SE_Engagement__c='Discovery' Product_Fit__c='3 - Good Fit' Functional_Selection__c=false Technical_Selection__c=false SE_Next_Steps__c='01/Jul/26 - Demo prep (SE). 08/Jul/26 - Workshop (Team).' SE_Comments__c='Active eval. Competing with a legacy incumbent vendor.' Tech_Exec__c='<default_tech_exec_user_id>'" \
  --target-org <config.org_alias>

sf data update record \
  --sobject Opportunity \
  --record-id <opp_id_2> \
  --values "..." \
  --target-org <config.org_alias>
```

**Field constraints:**
- `Functional_Selection__c` and `Technical_Selection__c` are BOOLEAN — `true`/`false` only
- `Product_Fit__c` requires FULL picklist label: "1 - Poor Fit", "2 - Fair Fit", "3 - Good Fit", "4 - Excellent Fit", "5 - Perfect Fit"
- `SE_Next_Steps__c` max 255 chars (period-separated, abbreviated)
- `SE_Comments__c` max 255 chars
- `Issues__c` (label: "Tech Exec Comments") is NOT updateable — skip
- `SE_Comments_History__c` is NOT updateable — skip
- Escape single quotes in values (replace `'` with `'\''` in bash strings)

**Execution rules:**
- Skip `SE_Comments_History__c` (read-only via API)
- Skip `Solution_Description__c` if no meaningful content to add
- Confirm once with user before running the batch — present a summary table of all planned changes first
- Run commands sequentially; report success/failure per opp as you go

### Step 6: Report Results

After all updates, provide a summary table:
- Account | Opps Updated | Key Intelligence Source
- Call out any failures or skipped opps with reason

## Rules

- **Do NOT ask for confirmation on each individual opp** — this is a bulk operation. Confirm once before starting the push, then run through all.
- **Ground every update in evidence** — either from the canvas, Slack, or both. Never fabricate context.
- **Be conservative with Product Fit** — stalled deals with placeholder dates and no activity get low scores (1-2).
- **Flag sequencing violations** — if an opp is blocked by another deal that hasn't progressed, note this in SE_Comments.
- **Skip opps with no actionable intelligence** — if there's nothing meaningful to say beyond "no activity", leave it blank rather than adding noise.
- **Downstream deals get lower engagement levels** — if a deal depends on another deal closing first, its SE_Engagement should reflect that (usually Preliminaries).

## Conversation Starters

- "Here's [AE Name]'s territory review canvas — bulk update all SE fields."
- "Update SE fields across this rep's book from Slack evidence."
- "Run the territory bulk updater on this canvas: [URL]"
