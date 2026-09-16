---
name: df27-account-profile
description: "Build or refresh a Dreamforce account profile file (account.md) in the Dreamforce planning folder. Queries Org62 for contracts, won line items, and open opportunities; reads Slack ZC channel and account research canvases; runs the SE Entitlement Decoder pattern if no footprint data exists; and writes a structured account.md covering: company description + why DF this year, strategic imperatives, Salesforce footprint, contract health, and open pipeline summary."
metadata:
  type: event-planning
  version: "1.1"
  origin: "Built 2026-07-09"
  depends_on: "mcp__salesforce-org62__run_soql_query, mcp__slack__slack_read_channel, mcp__slack__slack_read_canvas, mcp__slack__slack_search_channels, se-entitlement-decoder"
---

# DF27 Account Profile Builder

You build or refresh the `account.md` file for a Dreamforce-registered account. This file is the single source of truth for the "About [Company] at Dreamforce" section used in every attendee recommendation doc. It also captures strategic imperatives, Salesforce footprint, contract health, and pipeline — so the recommendation doc skill doesn't need to re-derive these.

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `org_alias` | Org62 target org / username — supplies the `usernameOrAlias` parameter for `mcp__salesforce-org62__run_soql_query` calls. **Never hardcode a username or email in this skill.** |

**Not yet a canonical config key (placeholder only):**
- The Dreamforce planning workspace root (used below as `<dreamforce_root>`, e.g. `~/dreamforce-2027/`) has no dedicated config key today. Use your own path until one is added — do not invent a key name in the skill itself.
- The `directory` parameter required by the Org62 SOQL MCP tool is the SE's home directory. Resolve it at runtime rather than hardcoding a path.

## When to Use

- A new account has registered contacts for DF27 and no `account.md` exists yet
- "Build the account profile for [Company]"
- "Refresh the account profile for [Company]"
- Before generating recommendation docs — always check `account.md` exists first

## Required Inputs

- **Account name** and **AE name**
- **Account ID** (from registrations file or contact profile)

## Step 1 — Check for Existing Data

Before querying, check if an account research canvas or product map already exists in the Slack ZC channel. This saves significant query time.

1. Use `mcp__slack__slack_search_channels` to find the ZC channel (`#ZC:...:AccountName`)
2. Read the channel with `mcp__slack__slack_read_channel` — look for canvas files (product maps, account research, call notes)
3. If a product map canvas exists: read it with `mcp__slack__slack_read_canvas` — it contains the full entitlement breakdown
4. If an account research canvas exists: read it — it contains strategic imperatives and stakeholder context

## Step 2 — Org62 Queries (run in parallel)

### Active Contracts
```soql
SELECT Id, ContractNumber, StartDate, EndDate, Status
FROM Contract
WHERE AccountId = '[AccountId]'
AND Status = 'Activated'
ORDER BY EndDate ASC
```
Flag any contract expiring within 90 days as ⚠️, within 30 days as 🚨.

### Won Line Items (Footprint)
```soql
SELECT Product2.Family, Product2.Name, SUM(Quantity) totalQty
FROM OpportunityLineItem
WHERE Opportunity.AccountId = '[AccountId]'
AND Opportunity.IsWon = true
AND Opportunity.CloseDate >= 2022-01-01
GROUP BY Product2.Family, Product2.Name
ORDER BY Product2.Family, Product2.Name
```

### Open Opportunities
```soql
SELECT Id, Name, StageName, Amount, CloseDate, Type, Owner.Name,
  (SELECT Contact.Name, Contact.Title FROM OpportunityContactRoles WHERE IsPrimary = true LIMIT 1)
FROM Opportunity
WHERE AccountId = '[AccountId]'
AND IsClosed = false
ORDER BY CloseDate ASC
```

## Step 3 — Entitlement Decoder (if no product map canvas)

If no product map canvas was found in Step 1, run the SE Entitlement Decoder pattern to resolve SKU names to actual entitlements. Follow the workflow in the `se-entitlement-decoder` skill.

The decoded output goes into the Salesforce Footprint section of `account.md` and should also be saved as a standalone canvas or file in the account folder if substantial.

## Step 4 — Write account.md

Write to: `<dreamforce_root>/accounts/{account-slug}/account.md`

Use this template:

```markdown
# [Account Name] — Account Profile
**AE:** [AE Name]
**Industry:** [Industry]
**HQ:** [City, Province/State, Country]
**Account ID:** [ID]
**Parent:** [If applicable]
**Last Refreshed:** [Date]

---

## About the Company

[2-3 sentences: what they do, ownership/structure, scale. Written plainly — this appears verbatim in customer-facing recommendation docs. No Salesforce jargon here.]

**Why DF this year:** [2-4 sentences. What's the inflection point for this account RIGHT NOW? What makes DF relevant for them specifically this year — not generically. Reference active deals, recent conversations, new hires, expiring contracts, competitive threats, or strategic decisions that need to be made.]

---

## Strategic Imperatives

| # | Imperative | Signal |
|---|---|---|
| 1 | **[Plain-language imperative]** | [What's the evidence — deal, Slack signal, call note, canvas] |
| 2 | ... | ... |

Aim for 3–5 imperatives. These should be *their* priorities, not Salesforce's pitch. Phrase them from the customer's perspective.

---

## Salesforce Footprint

| Product | Qty / Detail | Status |
|---|---|---|
| [Product] | [Qty or detail] | Active / Entitled / Pending / Not yet |

Include a **Key insight** note below the table if the footprint has something notable (dormant entitlements, Einstein 1 = full Agentforce, FSL deployment scale, etc.).

---

## Contract Health

| Contract # | Expires | Risk | Notes |
|---|---|---|---|
| [number] | [date] | 🚨 / ⚠️ / OK | [What's at risk] |

Only include if contracts are expiring within 12 months or if there's a lapse risk. Skip if all contracts are healthy.

---

## Open Pipeline Summary

| Opportunity | Stage | Amount | Close | Owner |
|---|---|---|---|---|
| [Name] | [Stage] | $[Amount] | [Date] | [Owner] |

Include a **Total active pipeline** line at the bottom (excluding far-future renewals).

---

## Key Contacts at DF

| Name | Title | Archetype | File |
|---|---|---|---|
| [Name] | [Title] | [Archetype] | `[file].md` |

Note any important contacts who are **NOT** attending DF — especially if they're the economic buyer or primary champion.

---

## Key References (optional)

- [Canvas name]: `[canvas_id]` ([channel])
- [Any other relevant files]
```

## After Writing

1. Update `INDEX.md` if the account row doesn't already point to the account folder
2. Confirm the file path to the user
3. If contract expiry risks were found, flag them explicitly — these need immediate action from the AE, not just a note in a file

## Key Conventions

- **"About the Company" and "Why DF this year"** are written to potentially appear in customer-facing docs — keep them clean, no internal deal language
- **Strategic Imperatives** are the customer's agenda, not Salesforce's pitch. If the customer wouldn't recognize their own imperative in your phrasing, rewrite it.
- **Footprint** should distinguish between Active (deployed and used), Entitled (licensed but dormant), Pending (open opp), and Not yet (not purchased)
- **Contract health** is actionable — anything expiring in 30 days needs AE escalation, not just a table row
- **Org62 queries** always require `usernameOrAlias: config.org_alias` and the SE's home `directory`

## Example Account Profile Entry (illustrative)

| Account | Slug | AE | File | Notes |
|---|---|---|---|---|
| [Account Name] | `<account-slug>` | [AE Name] | `accounts/<account-slug>/account.md` | e.g. "Full canvas data from a recent account-research canvas" |

This table is a running log of accounts profiled to date — keep entries generic in this shared skill file; do not record real customer names or AE names here. Track the real, per-run history in your own account workspace instead.
