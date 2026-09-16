---
name: se-avp-deal-update
description: "Generates a deal update in AVP format for a specific opportunity when the AVP or RVP asks for a status update. Pulls Org62 + Slack signals and formats as: Deal Name + $ + Tracking/Not Tracking, What happened, Next steps, Support needed, Risk, with the AVP and RVP tagged."
metadata:
  type: sales-operations
  version: "1.1"
---

# SE AVP Deal Update

Generate a concise deal update in the format your AVP expects, for a specific named deal.

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `ae_roster` | Your aligned AEs. Each entry's `manager` field is used as the RVP to tag for that AE's deals — see Step 6. |
| `ae_roster_path` | Path to the AE roster file, if you keep it as a file rather than inline JSON. |
| `se_name` | Your name — used for "Next steps" rows where you are the owner (e.g. joint demo prep). |

**No canonical config key exists for AVP identity, VP-above-RVP identity, or Slack user IDs for @-mentions** — these are flagged, not invented. Ask the user (or keep as bracket placeholders) for:
- `[AVP Name]` / `[AVP Slack ID]` — who this update format is written for
- `[VP Name]` / `[VP Slack ID]` — the VP above the RVPs, copied on larger deals
- Per-AE RVP names come from `config.ae_roster[].manager` — do not hardcode an AE→RVP table in this skill
- If `config.ae_roster` has no `manager` set for the AE in question, ask the user who the RVP is rather than guessing or defaulting to a specific name

## When to Use

- "AVP deal update for [Account]"
- "The AVP wants a deal update on [Account]"
- "Give me the update format for [Opp]"
- "What do I tell the AVP about [Account]?"
- "[Account] update for the AVP"

## Standard Format

```
[Deal Name linked to Org62 URL] + $[Amount] + Tracking / Not Tracking

1. What happened today?
[1-2 sentences or bullets on most recent activity]

2. Next steps?
• [Action] — [Owner], [Date]
• [Action] — [Owner], [Date]

3. Support needed?
[Specific ask, or "None at the moment"]

4. Risk?
[Honest one-liner — or "None at the moment" if clean]

@[AVP Name] @[RVP Name] (+ @[VP Name] on larger deals)
```

**If nothing happened:** Post "NO update" or copy/paste yesterday's update — many AVPs explicitly allow this. Confirm your own AVP's preference if unsure.

## Instructions

### Step 1: Look up the opportunity in Org62

Run a SOQL query to find the opportunity:
```sql
SELECT Id, Name, Amount, StageName, CloseDate, Probability, 
       Owner.Name, Account.Name
FROM Opportunity
WHERE Account.Name LIKE '%[account name]%'
  AND IsClosed = false
ORDER BY CloseDate ASC
LIMIT 5
```

Also pull recent activity:
```sql
SELECT Subject, ActivityDate, Description, Who.Name, Type
FROM ActivityHistory
WHERE What.Name LIKE '%[account name]%'
ORDER BY ActivityDate DESC
LIMIT 10
```

And SE fields:
```sql
SELECT SE_Engagement__c, Solution_Description__c, Next_Steps__c,
       SE_Comments__c, Product_Fit__c
FROM Opportunity
WHERE Account.Name LIKE '%[account name]%'
  AND IsClosed = false
LIMIT 1
```

### Step 2: Search Slack for recent signals

Search the account's ZC channel or any recent mentions:
- Search for the account name in Slack
- Look for the most recent thread with deal activity, call notes, or blockers

### Step 3: Determine Tracking / Not Tracking

**Tracking ✅** if:
- Close date is within the current or next fiscal quarter AND stage is ≥ Proposal/Pricing
- Or deal is progressing (stage advanced recently, next steps defined, active engagement)

**Not Tracking ❌** if:
- Close date has passed or is imminent but stage is early (Discovery/Qualification)
- No recent activity in 2+ weeks
- Known blocker with no resolution path

### Step 4: Fill the format

- **What happened today** — most recent activity from Org62 activity history or Slack. If nothing happened literally today, use "most recently" framing.
- **Next steps** — from SE Next Steps field in Org62, or inferred from last activity. Always include a date and owner.
- **Support needed** — check SE Comments or Slack for any outstanding asks. Default to "None at this time" if clean.
- **Risk** — honest one-liner. Pull from deal coach signals if available: no champion, single-threaded, stalled stage, missing technical win, etc.

### Step 5: Output — Canvas only

Create a Slack Canvas with the formatted update and return the link to the user. **Do not post to any channel.** The AE owns their channel presence — give them the canvas link so they can review, adapt, and post themselves.

### Step 6: Tag correctly

Standard tags for the update:
- AVP: **@[AVP Name]** — no canonical config key; confirm with the user or keep as a placeholder
- RVP depends on AE: look up `config.ae_roster` for the AE on this deal and use their `manager` field as the RVP. **Do not hardcode an AE→RVP table in this skill.**
- Also tag the VP above the RVPs on deals ≥$100K or when escalating, if your org has one — no canonical config key; confirm with the user or keep as `@[VP Name]`.

If the RVP can't be resolved from `config.ae_roster`, ask the user who the RVP is rather than defaulting to any specific name.

## Example Output

```
[[Account] - Agentforce for Service](https://org62.lightning.force.com/...) + $100K + Tracking

1. What happened today?
Completed technical discovery with [Account]'s IT lead — confirmed legacy Service Cloud EE, 
no current Einstein adoption. Strong interest in Agentforce for case deflection.

2. Next steps?
• Demo — [AE Name] + [SE Name], Sep 3
• [AE Name] to confirm exec sponsor attendance by Aug 28

3. Support needed?
None at the moment.

4. Risk?
Single-threaded at IT level — no confirmed business sponsor yet.

@[AVP Name] @[RVP Name] @[VP Name]
```

## Notes

- Keep it tight — AVP format is meant to be read in 30 seconds
- "What happened today" doesn't have to be literally today — use the most recent meaningful activity
- If there's genuinely nothing to report, say so honestly rather than padding
- If the deal needs AVP intervention (exec call, pricing relief, etc.), make the Support Needed field specific and direct
- **No hardcoded people.** AVP, RVP, and VP identities are not stored in this skill — RVP comes from `config.ae_roster[].manager`; AVP/VP have no canonical key and must be confirmed with the user or kept as placeholders (this repo is public).
