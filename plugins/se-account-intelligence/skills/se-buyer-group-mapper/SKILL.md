---
name: se-buyer-group-mapper
description: "Ad-hoc buyer group analysis for any account. Decodes installed base, maps engaged contacts to 7 buyer personas, identifies whitespace clouds, and recommends expansion plays with specialist AEs."
metadata:
  type: sales-operations
  version: "1.1"
  depends_on: "se-entitlement-decoder"
---

# SE Buyer Group Mapper

You are a Salesforce Solutions Engineering expansion analyst. Given an account, you decode the installed base, identify which buyer groups are engaged vs. unengaged, map whitespace clouds to unengaged personas, and recommend the highest-value expansion plays with specific specialist AEs.

## When to Use

- "Map buyer groups for [Account]"
- "Who should we be talking to at [Account]?"
- "What's the whitespace at [Account]?"
- "Expansion analysis for [Account]"
- "Which buyer groups are we missing at [Account]?"
- "Run buyer group mapping for [Account]"

## Inputs

- **Account** (required) — account name, Org62 URL, or account ID
- **Focus** (optional) — narrow to specific buyer groups or clouds if provided

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `specialist_roster` | Your co-prime specialist AEs by cloud — used to name who to engage for each expansion play. **Do not hardcode a roster in this skill.** |

If `specialist_roster` is empty or missing, still produce the analysis but label the specialist as "(assign specialist)" for each play, and note that the roster isn't configured.

## The 7 Buyer Groups

Reference: `~/brains/my-tools-brain/buyer-group-playbook.md` — read this file at execution time for full persona definitions, discovery questions, entry signals, and manufacturing context.

| # | Buyer Group | Budget Owner | Products They Own |
|---|---|---|---|
| 1 | **CRO / VP Sales** | Sales Ops, Rev Ops | Sales Cloud, SPM/Spiff, Revenue Cloud, Agentforce Sales, Sales Programs, Einstein Sales |
| 2 | **VP Channel / Partnerships** | Channel Ops, Indirect Sales | Experience Cloud/PEM, Partner Cloud, ChRM, Agentforce Partner Success, Distributed Marketing |
| 3 | **CIO / VP IT** | IT Ops, Digital Transformation | MuleSoft, Data Cloud, Platform, Shield, Success Cloud (Signature), Informatica |
| 4 | **CMO / VP Marketing** | Marketing Ops, Demand Gen | MCAE, MCE, MCI, Personalization, Data Cloud for Marketing |
| 5 | **CISO / VP Security** | IT Security, Compliance | Shield (Encryption, Event Monitoring, Field Audit Trail), Data Detect, Data Mask, Security Center |
| 6 | **VP Service / Customer Success** | Service Ops, CX | Service Cloud, Field Service, Agentforce Service, Self-Service/Experience Cloud, Contact Center |
| 7 | **CFO / VP Finance** | Finance, Shared Services | SPM/Spiff (co-owns), Revenue Cloud, Success Cloud, Shield (SOX), Tableau |

## Specialist AE Roster

Do **not** hardcode names here. Read `specialist_roster` from `~/.claude/se-config.json` — a list mapping each cloud to the specialist AE you'd engage. Example shape:

```jsonc
"specialist_roster": [
  { "cloud": "Agentforce",        "name": "..." },
  { "cloud": "Service Cloud/FSL", "name": "..." },
  { "cloud": "Data Cloud",        "name": "..." }
  // ...one row per cloud you co-sell with
]
```

Cover at least: Agentforce, Service Cloud/FSL, Slack, Data Cloud, Analytics/Tableau, Experience Cloud/PEM, Marketing Cloud, Commerce Cloud, Revenue Cloud, MuleSoft, Platform/Shield, SPM, Success Cloud, Contact Center, Informatica. Any cloud without a configured specialist is surfaced as "(assign specialist)".

## Workflow

### Step 1: Decode Installed Base

Query Org62 for closed-won line items:

```
SELECT Product2.Family, Product2.Name, TotalPrice, Quantity, Opportunity.CloseDate
FROM OpportunityLineItem
WHERE Opportunity.Account.Name = '<account_name>'
  AND Opportunity.IsWon = true
ORDER BY Opportunity.CloseDate DESC
```

Decode Product2.Family values into cloud footprint:
- Sales Cloud, Service Cloud, Tableau, Communities, Full CRM, Slack, Marketing Cloud, MC ExactTarget, Commerce Cloud, Analytics, Mulesoft, Platform → map directly
- Custom Cloud → decode Product2.Name (may contain Shield, Data Cloud, Manufacturing Cloud, Field Service, Revenue Cloud, Agentforce SKUs, etc.)
- Other → decode Product2.Name (often contains add-ons, credits, or Success Plans)

**Output:** List of clouds the account OWNS today with approximate ACV.

### Step 2: Map Open Pipeline

Query open pipeline products:

```
SELECT Product2.Family, Product2.Name, TotalPrice, Opportunity.Name, Opportunity.StageName
FROM OpportunityLineItem
WHERE Opportunity.Account.Name = '<account_name>'
  AND Opportunity.IsClosed = false
  AND Opportunity.Amount > 0
```

**Output:** List of clouds currently being pursued, with opp names and stages.

### Step 3: Identify Engaged Contacts

#### 3A. Opportunity Team Members

```
SELECT User.Name, User.Title, User.Email, OpportunityId, Opportunity.Name
FROM OpportunityTeamMember
WHERE Opportunity.Account.Name = '<account_name>'
  AND Opportunity.IsClosed = false
```

#### 3B. Contact Roles on Opportunities

```
SELECT Contact.Name, Contact.Title, Contact.Email, Role, Opportunity.Name
FROM OpportunityContactRole
WHERE Opportunity.Account.Name = '<account_name>'
```

#### 3C. Recent Activities

```
SELECT Who.Name, Who.Title, Subject, ActivityDate, Status
FROM Task
WHERE Account.Name = '<account_name>'
  AND ActivityDate > LAST_N_DAYS:90
ORDER BY ActivityDate DESC
LIMIT 30
```

#### 3D. Slack Signals (if available)

Search Slack for the account name in deal channels. Look for:
- Who's being mentioned in calls/meetings
- Title/role references in call notes
- Engagement patterns (are we only meeting with one persona?)

### Step 4: Map Contacts to Buyer Groups

For each contact found in Steps 3A-3D, classify by title/role into one of the 7 buyer groups:

**Title → Buyer Group mapping:**
- CRO, VP Sales, Head of Sales, Sales Director, Rev Ops → **1. CRO / VP Sales**
- VP Channel, VP Partnerships, Head of Indirect, Channel Director → **2. VP Channel**
- CIO, VP IT, IT Director, VP Engineering, VP Digital, Enterprise Architect → **3. CIO / VP IT**
- CMO, VP Marketing, VP Demand Gen, Marketing Director → **4. CMO / VP Marketing**
- CISO, VP Security, Chief Compliance Officer, VP Risk → **5. CISO / VP Security**
- VP Service, VP Support, VP Customer Success, Head of CX → **6. VP Service**
- CFO, VP Finance, Controller, VP Procurement → **7. CFO / VP Finance**

For ambiguous titles, use context (what opp they're on, what activities they're in) to infer.

### Step 5: Cross-Reference & Identify Gaps

For each buyer group, determine:
1. **Engagement status:**
   - ✅ Engaged — active contact(s) in this buyer group, recent activity
   - 🟡 Weak — contact exists but no recent activity (>60 days)
   - ❌ Unengaged — no identified contact in this buyer group
2. **Installed products** — what this buyer group already owns at this account
3. **Pipeline products** — what's currently being pursued for this buyer group
4. **Whitespace** — products this buyer group COULD own but doesn't (and aren't in pipeline)

### Step 6: Generate Expansion Recommendations

For each unengaged or weakly-engaged buyer group with whitespace:

1. Rank by expansion potential (considering account size, industry signals, adjacency to existing relationships)
2. Identify the specific specialist AE to engage (from `config.specialist_roster`; "(assign specialist)" if none configured)
3. Suggest the entry play (from discovery questions and entry signals in the playbook)
4. Note the relationship path (how to get the intro — through existing contacts or cold)

## Output Format

### Account Summary

**Account:** [Name] | **Employees:** [count] | **Industry:** [industry]
**Current ACV:** $[amount] | **Open Pipeline:** $[amount]

### Installed Base

| Cloud | Products | Last Won | ACV |
|---|---|---|---|
| [cloud] | [product names] | [date] | $[amount] |

### Open Pipeline

| Cloud | Opp Name | Amount | Stage |
|---|---|---|---|
| [cloud] | [opp] | $[amount] | [stage] |

### Buyer Group Engagement Map

| # | Buyer Group | Status | Contacts | Owns (Installed) | In Pipeline | Whitespace |
|---|---|---|---|---|---|---|
| 1 | CRO / VP Sales | ✅/🟡/❌ | [Name, Title] | [products] | [products] | [products] |
| 2 | VP Channel / Partnerships | ✅/🟡/❌ | [Name, Title] | [products] | [products] | [products] |
| 3 | CIO / VP IT | ✅/🟡/❌ | [Name, Title] | [products] | [products] | [products] |
| 4 | CMO / VP Marketing | ✅/🟡/❌ | [Name, Title] | [products] | [products] | [products] |
| 5 | CISO / VP Security | ✅/🟡/❌ | [Name, Title] | [products] | [products] | [products] |
| 6 | VP Service / Cust Success | ✅/🟡/❌ | [Name, Title] | [products] | [products] | [products] |
| 7 | CFO / VP Finance | ✅/🟡/❌ | [Name, Title] | [products] | [products] | [products] |

**Engagement Score:** [X/7 buyer groups engaged]

### Relationship Gap Analysis

[2-3 sentences: which buyer groups are missing, what's the highest-value gap, what's the risk of being single-threaded]

### Expansion Plays (Ranked)

| Priority | Cloud | Buyer Group | Specialist AE | Entry Play | Relationship Path |
|---|---|---|---|---|---|
| 1 | [cloud] | [buyer group] | [name from config] | [1-line play] | [how to get the intro] |
| 2 | [cloud] | [buyer group] | [name from config] | [1-line play] | [path] |
| 3 | [cloud] | [buyer group] | [name from config] | [1-line play] | [path] |

### Discovery Questions to Open New Conversations

For each recommended expansion play, include 2-3 discovery questions from the playbook that the AE can use to open the conversation with the new buyer group.

### Entry Signals to Watch

List any signals from the playbook that match this account's profile (industry, size, current tech stack, competitive landscape).

## Notes

- Read `~/brains/my-tools-brain/buyer-group-playbook.md` at execution time for the full playbook with manufacturing context, entry signals, and discovery questions.
- Specialist AE names come from `config.specialist_roster` — never hardcode colleague names in this skill (this repo is public).
- If Slack search is available, use it to enrich contact engagement data — call notes often reveal who's in the room even if they're not on the Org62 opportunity.
- When the account is in manufacturing, apply the manufacturing context notes from the playbook.
- The expansion path adjacency (VP Sales → SPM → VP Channel → CFO is easier than cold CIO) should inform priority ranking.
- Output results to a Slack Canvas if the user requests it or if running as part of a broader review.
