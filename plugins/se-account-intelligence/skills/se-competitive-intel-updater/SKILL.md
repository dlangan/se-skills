---
name: se-competitive-intel-updater
description: "Sweeps org62 (Account activities, Opportunity fields, text fields, files) and Slack for competitive intelligence signals, then executes updates directly to Competitive_Intelligence__c via Salesforce CLI. Turns scattered mentions into structured competitive data."
metadata:
  type: sales-operations
  version: "2.2"
  depends_on: "se-entitlement-decoder, se-case-insight-analyzer, se-deal-coach"
  origin: "v1.0 ported 2026-06-01, SF_OPS routing 2026-06-22, CLI direct 2026-07-06"
---

# SE Competitive Intelligence Updater

You gather competitive intelligence from every available source — org62 activities, opportunity fields, text fields, files, and Slack conversations — and execute updates directly to the `Competitive_Intelligence__c` record via the Salesforce CLI. You turn scattered mentions buried in call notes, deal comments, and Slack threads into actionable competitive data visible in org62.

**Write behavior:** This skill performs live writes to the connected org (`sf data update record` against `Competitive_Intelligence__c`). It never creates new CI records — only updates existing ones — and it should always show a summary of pending changes and get user confirmation before executing.

**Schema Reference:** See `~/.claude/skills/sf-ops-schema-reference.md` (or your local equivalent) for authoritative field types, lengths, and constraints on `Competitive_Intelligence__c`. This is a runtime-local reference file — there is no canonical config key for it; if it's missing, describe the object live instead (`sf sobject describe --sobject Competitive_Intelligence__c`).

**Key constraint:** `Competitive_Intelligence__c` records are auto-provisioned per account. You CANNOT create them — only UPDATE existing records.

## When to Use

- "Update competitive intel for [Account]"
- "What competitors are in play at [Account]?"
- "Sweep [Account] for competitive signals"
- After a deal coach or account research run reveals competitive context
- As part of the weekly territory review (flag accounts with empty CI records)

## Inputs

- **Account** (required) — account name, ID, or org62 URL
- **Focus** (optional) — specific product category to investigate (e.g., "CRM", "Analytics", "AI")

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `org_alias` | Target org for all `sf data update record` writes — passed as `--target-org`. **Never hardcode an org alias or username in this skill.** |
| `account_root` | Base path for the account workspace where the Tech Stack Map is persisted (e.g. `<account_root>/[account]/research/tech-stack.md`) |
| `se_name` | Optional — if you want the `_Set_By__c` field to reflect a real operator instead of the automation label, read this. Default behavior (see Step 6) uses a generic automation label and does not require this key. |

If `org_alias` is missing, stop before executing any CLI write and ask the user which org to target — do not guess or fall back to a personal default.

## The Competitive_Intelligence__c Object

One record per account. Wide table with 64 product categories. **Before sweeping for signals, reference this complete field list to know what to look for.**

### Field Pattern (per category)

| Field Suffix | What It Stores | Priority |
|---|---|---|
| `_Primary_Product__c` | Picklist — the competitor product name | Always populate |
| `_Primary_Comments__c` | Textarea — context, status, strategy notes | Always populate |
| `_Primary_Source__c` | String — where the intel came from (date + source) | Always populate |
| `_Primary_Contract_Exp_Date__c` | Date — when the competitor contract expires | When known |
| `_Primary_Date_Set__c` | Date — when this intel was captured | Always set to today |
| `_Primary_Set_By__c` | String — who captured it | Set to "Claude Code (automated)" |
| `_Secondary_Product__c` | Second competitor in this category | When two competitors exist |
| `_Secondary_Comments__c` | Context for secondary | When relevant |
| `_Secondary_Source__c` | Source for secondary | When relevant |
| `_Secondary_Contract_Exp_Date__c` | Contract date for secondary | When known |
| `_Secondary_Date_Set__c` | Date captured | When writing secondary |
| `_Secondary_Set_By__c` | Who captured it | When writing secondary |
| `_Notes__c` | Additional category-level notes | For broader landscape context |

### Complete Category Reference (64 categories)

**Before searching, scan this list to identify which categories to investigate for the account's industry.**

#### Tier 1 — Core SE Categories (always check these)

Full tracking with Primary + Secondary + Notes + Contract Dates (13 fields each):

| Category Prefix | What It Covers | Salesforce Competes With | Look For |
|---|---|---|---|
| `Sales` | CRM / Sales tools | Sales Cloud, Agentforce for Sales | HubSpot, Dynamics, Zoho, NetSuite, SugarCRM, Pipedrive, Freshsales, Close.io |
| `Service` | Service / Support platforms | Service Cloud, Agentforce for Service | Zendesk, Freshdesk, ServiceNow, Zoho, Intercom, Help Scout |
| `Marketing` | Marketing automation | Marketing Cloud, MCAE | Marketo, Mailchimp, ActiveCampaign, Brevo, Constant Contact |
| `Analytics` | BI / Reporting tools | Tableau, CRM Analytics | Power BI, Qlik, Looker, Sisense, Domo, ThoughtSpot |
| `Collab` | Collaboration platforms | Slack | Microsoft Teams, Google Chat, Zoom Workplace |
| `CPQ` | Quote-to-cash | Revenue Cloud, CPQ | Conga, DealHub, PandaDoc, Proposify, Zuora |
| `Ecommerce` | Commerce platforms | Commerce Cloud | Shopify, Magento, BigCommerce, WooCommerce, SAP Commerce |
| `Platform` | App development / low-code | Salesforce Platform | ServiceNow App Engine, OutSystems, Mendix, Power Apps |
| `Mobile` | Mobile platforms | Salesforce Mobile | Custom apps, competitor mobile solutions |
| `Cust_Comm` | Customer communications | Marketing Cloud, Experience Cloud | Twilio, Braze, Iterable, Customer.io |
| `Emp_Comm` | Employee communications | Slack (internal) | Microsoft Viva, Workvivo, Staffbase |
| `Part_Comm` | Partner communications | Experience Cloud, PRM | Impartner, Allbound, PartnerStack |
| `Service_Chat` | Chat / Messaging | Messaging, Agentforce | Intercom, Drift, LiveChat, Ada |

#### Tier 2 — Extended SE Categories (check when relevant to account industry)

Basic tracking (3 fields: Product, Comments, Source):

| Category Prefix | What It Covers | When to Check |
|---|---|---|
| `AI_Machine_Learning` | AI/ML platforms | Always — Agentforce competitive |
| `Artificial_Intel` | AI tools (overlaps AI_ML) | When AI evaluation mentioned |
| `Field_Service` | FSL platforms | Manufacturing, energy, utilities accounts |
| `Middleware_Interface` | Integration / iPaaS | When integration complexity identified |
| `B2B_Marketing` | B2B-specific marketing | B2B accounts with marketing pipe |
| `Business_Intel` | Business intelligence | When BI mentioned separately from analytics |
| `Data_Aggregation` | Data platforms / CDPs | When data unification discussed |
| `Scheduling` | Scheduling tools | Field service or appointment-based businesses |
| `Compensation_Plans` | SPM / Incentive tools | When sales comp discussed |
| `Revenue_Mgmt` | Revenue management | When billing/revenue recognition in scope |

#### Tier 3 — Industry-Specific Categories (check only for matching verticals)

| Category Prefix | Industry | What It Covers |
|---|---|---|
| `Core_Billing`, `Core_Claim`, `Core_Policy_Admin`, `Core_Underwriting` | Insurance | Core insurance systems |
| `Broker_Relationship_Mgmt`, `Quote_Aggregation` | Insurance | Distribution/agency |
| `EMR_Software`, `Clinical_Data_Whse`, `Health_Info_Exchange`, `Patient_Portal`, `Population_Health`, `Care_Mgmt_Coord`, `Research_Clinical_Dev` | Healthcare | Clinical/health IT |
| `Financial_Planning`, `Portfolio_Management`, `Trading_Software`, `Robo_Advice`, `Wealth_Mgmt_CRM`, `Custody_Cleaning`, `Middle_Office` | Financial Services | Wealth/capital markets |
| `Advisor_Portal`, `Member_Engagement`, `Group_Sales_Enrollment` | Financial Services / Insurance | Member/advisor tools |
| `Agency_Management`, `Fraud_Detection`, `Insurance_Data` | Insurance | Specialty insurance |
| `Onboarding` | Cross-industry | Employee/customer onboarding |
| `Prod_Lifecycle_Mgmt_R_D`, `Quality_Mgmt_Reg_Sys` | Manufacturing / Life Sciences | PLM and QMS |
| `Compliant_Archive`, `Compliant_Social`, `Compliant_Text` | Regulated industries | Compliance tools |

### How to Use This Reference

1. **Identify the account's industry** from org62 (Account.Industry)
2. **Always check Tier 1 categories** — these are relevant for every account
3. **Check Tier 2 categories** based on what's been discussed (AI eval? Check AI_Machine_Learning. Integration pain? Check Middleware_Interface.)
4. **Check Tier 3 only** if the account is in that specific vertical (Insurance, Healthcare, FSI)
5. **When sweeping Slack/activities**, scan for ANY technology name — then map it to the appropriate category from this list
6. **If a technology doesn't fit any category** → use the closest match + put specifics in Comments

## Workflow

### Step 1: Check Existing CI Record + Salesforce Footprint

**CI Record:**
```
SELECT Id, [all populated fields]
FROM Competitive_Intelligence__c
WHERE Account__c = '<account_id>'
```

If no record exists → note this for creation (requires admin — CI records are usually auto-created).
If record exists → read current state to avoid overwriting fresher data.

**Salesforce Footprint — Run the Entitlement Decoder FIRST:**

Before assessing any competitive landscape, run the **se-entitlement-decoder** logic to understand exactly what the customer already owns from Salesforce. This prevents:
- Recommending a Salesforce product they already have
- Misidentifying a category as "competitive" when Salesforce is actually the incumbent
- Missing the "activate what you own" angle that's stronger than "buy something new"

**Query contracts:**
```
SELECT Id, ContractNumber, StartDate, EndDate, Status
FROM Contract
WHERE AccountId = '<account_id>' AND Status = 'Activated'
ORDER BY EndDate DESC
```

**Query active asset line items (what they have today):**
```
SELECT Apttus_Config2__ProductId__r.Name, Apttus_Config2__ProductId__r.Family,
       Apttus_Config2__Quantity__c, Apttus_Config2__EndDate__c
FROM Apttus_Config2__AssetLineItem__c
WHERE Apttus_Config2__AccountId__c = '<account_id>'
  AND Apttus_Config2__EndDate__c >= TODAY
  AND Apttus_Config2__Quantity__c > 0
ORDER BY Apttus_Config2__ProductId__r.Family
```

**Resolve key SKUs to entitlements (ProductLicenseMap):**
```
SELECT LicenseDefinition.Name
FROM ProductLicenseMap
WHERE ProductId = '<product2_id>'
ORDER BY LicenseDefinition.Name
```

**Why this must happen FIRST:**

| Scenario | Without Entitlement Check | With Entitlement Check |
|---|---|---|
| Customer has a competitor tool for service | "Competitor: [Competitor] (Service)" | "Competitor: [Competitor]. BUT: customer already owns Service Cloud UE (310 licenses) + Agentforce for Service (168) + Einstein for Service (168). [Competitor] is being DISPLACED, not competing with nothing." |
| Customer evaluating AI tools | "Competitor: [Tool A], [Tool B]" | "Competitor: [Tool A], [Tool B]. BUT: customer already has Einstein Agent Basic + Builder Free + GPT Sales Access + 80M Flex Credits entitled. Position as ACTIVATION of what they own, not competing against what they're evaluating." |
| Customer uses a third-party FSL tool | "Competitor: [Tool] (Field Service)" | "Competitor: [Tool]. BUT: customer's Manufacturing Cloud S&S UE includes IndustriesFieldServiceAddOn + Asset Hierarchy + Labor Cost Optimization. FSL is ENTITLED — they just haven't activated it." |

**Build the Salesforce Position Map before writing competitive data:**

For each Tier 1 category, determine Salesforce's position:

| Category | SF Position | Evidence | Implication for CI |
|---|---|---|---|
| Sales / CRM | Incumbent / Expanding / Absent | [contracts + entitlements] | [defend / cross-sell / compete] |
| Service | Incumbent / Expanding / Absent | [contracts + entitlements] | [defend / cross-sell / compete] |
| AI / Agentforce | Entitled / Purchased / Absent | [entitlements in PLM] | [activate / expand / compete] |
| Analytics | Incumbent / Expanding / Absent | [Tableau contracts] | [defend / cross-sell / compete] |
| Marketing | Incumbent / Expanding / Absent | [MC contracts] | [defend / cross-sell / compete] |
| Collaboration | Incumbent / Expanding / Absent | [Slack contract] | [defend / cross-sell / compete] |
| Integration | Incumbent / Expanding / Absent | [MuleSoft contract] | [defend / cross-sell / compete] |
| Field Service | Entitled / Absent | [entitlements in PLM] | [activate / compete] |
| Commerce | Incumbent / Absent | [Commerce contracts] | [defend / compete] |

**Write to the CI record — understanding the field semantics:**

The `_Primary_Product__c` field means: **"What does the account primarily USE in this category?"**
- If Salesforce IS the primary → `Primary_Product = "Salesforce"` and the competitor goes in `Secondary_Product`
- If a competitor IS the primary → `Primary_Product = "[Competitor]"` and note SF position in comments
- If Salesforce is entitled but competitor is actively used → `Primary_Product = "[Competitor]"` but comments note: "SF entitlement exists — activation opportunity"

**Writing rules by SF position:**

| SF Position | Primary_Product__c | Secondary_Product__c | Comments Context |
|---|---|---|---|
| **SF is incumbent (actively used)** | "Salesforce" | [Competitor being displaced, if any] | "SF IS PRIMARY. [X] licenses, [entitlements]. [Competitor] is secondary/legacy being displaced." |
| **SF is entitled but NOT primary user** | [Competitor that IS primary] | "Salesforce" or leave blank | "SF entitlement exists ([entitlement] in [SKU]) but [Competitor] is actively used. Activation opportunity." |
| **SF has NO position** | [Competitor] | [2nd competitor if known] | "No SF footprint. [Competitor] is incumbent. Net-new competitive play." |

- Flag any contracts expiring within 6 months → "Contract [#] expires [date]. Competitors will be circling. Defend + expand in renewal conversation."

### Step 2: Sweep Org62 for Competitive Signals

#### 2A. Opportunity Fields

```
SELECT Id, Name, CompetitiveStatus__c, PrimaryCompetitor__c, 
       Competitive_Notes__c, How_did_the_Competitor_Differentiate__c,
       SE_Comments__c, Solution_Description__c, Description
FROM Opportunity
WHERE AccountId = '<account_id>' AND IsClosed = false
```

Look for: competitor names in any text field, competitive status fields, SE comments mentioning alternatives.

#### 2B. Account Text Fields

```
SELECT Description, Industry, Industry_Focus__c
FROM Account
WHERE Id = '<account_id>'
```

#### 2C. Activity History (Tasks + Events)

```
SELECT Subject, Description, ActivityDate, WhoId
FROM Task
WHERE AccountId = '<account_id>'
  AND ActivityDate >= LAST_N_MONTHS:6
  AND (Subject LIKE '%competitor%' OR Subject LIKE '%HubSpot%' OR Subject LIKE '%Dynamics%' 
       OR Description LIKE '%competitor%' OR Description LIKE '%evaluating%'
       OR Description LIKE '%alternative%' OR Description LIKE '%vendor%')
ORDER BY ActivityDate DESC
LIMIT 20
```

#### 2D. Notes & Files

Check for attached competitive content:
```
SELECT Id, Title, FileType, CreatedDate
FROM ContentDocumentLink
WHERE LinkedEntityId = '<account_id>'
```

### Step 3: Sweep Slack for Competitive Signals

#### 3A. Deal Channel (#ZC:*)

Read recent messages for mentions of:
- Competitor product names (HubSpot, Dynamics, Zoho, Gong, Orum, IFS, ServiceNow, etc.)
- Evaluation language ("evaluating", "comparing", "shortlist", "vendor selection", "RFP")
- Pricing mentions (e.g. dollar figures, "their pricing", "cost comparison")
- Competitive positioning discussions

#### 3B. Broad Account Search

```
"<Account Name>" competitor OR evaluating OR versus OR alternative OR shortlist
```

#### 3C. DM Conversations (AE-SE)

Search for competitive context shared privately between the AE and SE that may not be in the deal channel.

### Step 4: Synthesize & Categorize

For each competitive signal found, determine:

1. **Which category?** — Map the competitor to one of the 57 product categories
2. **Primary or Secondary?** — Is this the main competitor or a secondary mention?
3. **What do we know?** — Product name, pricing, status (evaluating/incumbent/displaced), contract dates
4. **Source & date** — Where did this intel come from and when?

### Step 5: Validate Picklist Values

Before writing, confirm the competitor name matches an available picklist value:

```python
# If exact match exists → use it
# If close match → use closest (e.g., "HubSpot" → "HubSpot CRM")
# If no match → use "Other" and put the specific name in Comments
```

Common mappings:
| What's Said | Picklist Value |
|---|---|
| HubSpot | HubSpot CRM |
| Dynamics / D365 | Microsoft Dynamics 365 Online |
| Zoho | Zoho |
| ServiceNow | ServiceNow |
| Zendesk | Zendesk |
| Freshdesk | Freshdesk |
| IFS | Other (put "IFS" in comments) |
| Gong | Other (put "Gong" in comments) |
| Orum | Other (put "Orum" in comments) |
| Snowflake | Other (put "Snowflake" in comments) |
| Power BI | Microsoft Power BI |
| NetSuite | NetSuite CRM |

### Step 6: Execute Updates via Salesforce CLI

Read `config.org_alias` from `~/.claude/se-config.json` first — this supplies `--target-org` for every write below. Do not hardcode an org alias or username.

Execute `sf data update record` directly for each category with new intel:

```bash
sf data update record \
  --sobject Competitive_Intelligence__c \
  --record-id <ci_record_id> \
  --values "Analytics_Primary_Product__c='Microsoft Power BI' Analytics_Primary_Comments__c='[<date>] Customer using Power BI for reporting dashboards. Single analyst, manual process. Evaluating alternatives.' Analytics_Primary_Source__c='<date> discovery call with <contact_name>' Analytics_Primary_Date_Set__c='<YYYY-MM-DD>' Analytics_Primary_Set_By__c='Claude Code (automated)'" \
  --target-org <config.org_alias>
```

For multiple categories, batch all changed fields into a single `sf data update record` call where possible — or chain multiple calls for large updates.

**Important constraints (from describe):**
- `Competitive_Intelligence__c` is NOT createable — only UPDATE existing records
- `_Primary_Product__c` fields are picklists — use exact picklist values or "Other" + detail in Comments
- `_Primary_Comments__c` fields are textarea (5,000 chars) — append with date prefix, don't overwrite
- `_Primary_Source__c` is string (128 chars max) — keep concise
- `_Primary_Set_By__c` is string (80 chars max)
- `_Primary_Date_Set__c` is date (YYYY-MM-DD format)
- `_Primary_Contract_Exp_Date__c` is date (YYYY-MM-DD format)
- Escape single quotes in values (replace `'` with `'\''` in bash strings for fields with apostrophes)

**Execution rules:**
- **Never overwrite fresher data.** If `Date_Set` is more recent than your source, skip that category.
- **Append to comments, don't replace.** Read existing comments first; prepend new intel with a date prefix: `[<date>] New signal: ...` then existing text.
- **Always set Source and Date_Set.** Traceability matters.
- **Use Set_By = "Claude Code (automated)"** so humans know this was machine-populated. (If your org wants a named operator instead, read `config.se_name` and use that — don't hardcode a person's name here.)
- **Confirm with user before executing** — show a summary of what will change first, then run the CLI commands on confirmation.

### Step 7: Report What Was Found

Output a summary:

```markdown
## Competitive Intelligence Update: [Account]

**Record:** [CI record ID]
**Updated:** [date]

| Category | Competitor | Status | Key Intel | Source |
|---|---|---|---|---|
| Sales/CRM | [name] | [Incumbent/Evaluating/Displaced] | [1-line context] | [source + date] |
| Service | [name] | [status] | [context] | [source] |
| AI/ML | [name] | [status] | [context] | [source] |

**What changed:**
- [Field]: [old value] → [new value]
- [Field]: [blank] → [new value]

**Signals found but NOT written (need validation):**
- [signal that's ambiguous or unconfirmed]
```

## Automation Triggers

This skill should run:
- **After every Deal Coach** — competitive context often surfaces in call transcripts
- **After Account Research** — external research reveals competitor landscape
- **During Weekly Territory Review** — flag accounts with empty CI records as a hygiene action
- **On demand** — "update competitive intel for [Account]"

## What Counts as Competitive Intelligence

**INCLUDE:**
- Named competitor products in use or being evaluated
- Pricing information (theirs, not ours)
- Contract expiration dates for competitor products
- Customer statements about alternatives ("we're also looking at...")
- Win/loss reasons that reference competitors
- Technology stack components that compete with Salesforce products
- Partner/SI relationships with competitor ecosystems
- **Any named technology in the customer's stack** (see Tech Stack section below)

**EXCLUDE:**
- Internal Salesforce competitive strategy (that's our playbook, not CI data)
- Speculative competitors not mentioned by the customer
- Generic infrastructure that has no Salesforce relevance (e.g., "they use AWS" without context)

## Tech Stack Intelligence (Special Focus)

**Every technology mentioned by the customer — in calls, Slack, activities, or web research — should be captured.** The tech stack tells us:

1. **What competes** — products that overlap with Salesforce capabilities (replacement targets)
2. **What integrates** — systems we need to connect to (MuleSoft opportunity)
3. **What complements** — platforms that work alongside Salesforce (partner ecosystem)
4. **What's being replaced** — systems actively being sunset (our entry point)

### Tech Stack Signal Sources

| Source | What to Look For |
|---|---|
| Call transcripts | "We use [X]", "We're on [X]", "We just upgraded [X]", "We're replacing [X]" |
| Slack threads | System names, vendor discussions, integration mentions |
| Job postings (web) | Required skills reveal tech stack (e.g., "SAP experience required" = SAP in stack) |
| Customer website | Technology badges, partner logos, integration pages |
| Case history | Support cases mentioning connected systems, API partners, middleware |
| SE Comments | Technical notes from discovery about their architecture |

### Tech Stack Categorization

When a technology is found, categorize it:

| Category | Role | Examples | Salesforce Relevance |
|---|---|---|---|
| **CRM** | Customer/deal management | HubSpot, Dynamics, Zoho, NetSuite, Pipedrive | Direct competitor — replacement target |
| **ERP** | Back-office operations | SAP, Oracle, NetSuite, Epicor, Sage, Business Central | Integration target (MuleSoft) |
| **Marketing Automation** | Campaigns, email, lead gen | Marketo, Pardot, Mailchimp, Constant Contact, ActiveCampaign | MC replacement or integration |
| **Service/Support** | Ticketing, case management | Zendesk, Freshdesk, Intercom, ServiceNow, Jira Service Mgmt | Service Cloud replacement |
| **Analytics/BI** | Reporting, dashboards | Tableau, Power BI, Looker, Qlik, Snowflake, Databricks | Tableau opportunity or Data Cloud integration |
| **Integration/Middleware** | Connecting systems | Boomi, Informatica, Workato, Zapier, custom APIs | MuleSoft competitive or integration need |
| **Collaboration** | Team communication | Microsoft Teams, Slack (competitor), Google Workspace | Slack opportunity |
| **Field Service** | Mobile workforce | IFS, ServiceMax, ClickSoftware, Jobber | FSL replacement |
| **Commerce** | E-commerce platform | Shopify, Magento, BigCommerce, WooCommerce | Commerce Cloud opportunity |
| **CPQ/Billing** | Quote-to-cash | Conga, DealHub, Zuora, Chargebee | Revenue Cloud opportunity |
| **Data/CDP** | Customer data platform | Segment, Tealium, mParticle, Treasure Data | Data Cloud competitive |
| **AI/Automation** | AI tools, RPA, agents | Gong, Orum, Regie.ai, UiPath, Automation Anywhere | Agentforce competitive |
| **Phone/Dialer** | Calling platform | RingCentral, Dialpad, Aircall, Five9, Genesys | Service Cloud Voice / CTI |
| **Document/Content** | Doc management | DocuSign, PandaDoc, Highspot, Seismic | Adjacent — integration partner |

### Writing Tech Stack to the CI Record

- **Direct competitors** → write to the matching `[Category]_Primary_Product__c` field
- **Integration targets** → write to `Middleware_Interface_Primary_Comments__c` or the relevant category comments with note: "Integration target — MuleSoft opportunity"
- **Replacement targets** → write to the matching category + flag in comments: "ACTIVELY BEING REPLACED — [timeline if known]"
- **Complementary tech** → note in comments of the most relevant category but don't position as competitor

### Tech Stack Summary (Additional Output)

When reporting competitive intel, always include a separate **Tech Stack Map**:

```markdown
## Tech Stack Map: [Account]

| System | Category | Role | Status | SF Relevance |
|---|---|---|---|---|
| [e.g., SAP] | ERP | Order management, inventory | Active (just upgraded) | Integration target (MuleSoft) |
| [e.g., a helpdesk tool] | Service | Basic support ticketing | Active (low utilization) | Service Cloud replacement |
| [e.g., Snowflake] | Analytics | Data warehouse | Active (heavy investment) | Data Cloud integration partner |
| [e.g., a CRM competitor] | CRM | Being evaluated | Evaluating | Direct competitor |
| [e.g., a field service tool] | Field Service | Dispatch + scheduling | Active (targeted for removal EOY) | FSL replacement target |

**Key Integration Points:** [Which systems need to talk to Salesforce?]
**Replacement Targets:** [Which systems are being actively sunset?]
**Whitespace:** [Which categories have NO system today = greenfield for Salesforce?]
```

This tech stack map should also be written to the account workspace (`<config.account_root>/[account]/research/tech-stack.md`) for persistence across conversations.

## Rules

- **Only write what's evidence-based.** If a competitor wasn't explicitly mentioned by the customer or found in activity notes, don't populate the field. Empty is better than speculative.
- **Date everything.** Competitive landscape changes fast. A competitor name without a date is useless in 6 months.
- **Comments > Picklist.** The picklist value tells you WHO. The comments tell you WHY and WHAT'S HAPPENING. Always fill both.
- **Source is mandatory.** "May 28 call with [contact name]" is traceable. "General knowledge" is not.
- **Flag stale intel.** If the CI record was last updated 6+ months ago, flag it in the territory review as needing a refresh.
- **Don't overwrite human-entered data.** If a field has a recent Date_Set and was Set_By a human, don't replace it with automated intel unless the new data is clearly more current.
- **"Other" is fine.** Not every competitor fits the picklist. Use "Other" + detailed comments rather than forcing a wrong match.

## Conversation Starters

- "Update competitive intel for [Account]"
- "What competitors are in play at [Account]?"
- "Sweep [Account] for competitive signals"
- "Populate CI records for all accounts in my territory"
- "Which accounts in my territory have empty CI records?"

## Knowledge References

When sweeping for competitive signals, use this resource for categorization and positioning:
- `~/agentforce-brain/digests/competitive/positioning-and-competition.md` — Full competitive landscape: CRM suites (MS/HubSpot/ServiceNow), hyperscalers (Azure/Google/AWS), point solutions, positioning strategies, objection responses
- `~/agentforce-brain/digests/competitive/customer-stories-index.md` — Customer stories indexed by industry/use case for competitive win evidence
