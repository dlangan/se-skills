---
name: bvs-discovery-questionnaire
description: "Generates a customer-specific Business Value Discovery Questionnaire by mapping proposed Salesforce products to value drivers, pre-populating known values from account research, and outputting a Google Sheet for customer completion. Writes driver stubs to drivers.yaml when a demo-project is active."
metadata:
  type: business-value
  version: "1.1"
  depends_on: "se-account-research"
---

# BVS Discovery Questionnaire

Generates a customer-specific Business Value Discovery Questionnaire. Maps proposed Salesforce products to value drivers, pre-populates all known values from account research, and outputs a Google Sheet ready for customer completion. When a `.demo-project/` is active, also writes driver stubs to `drivers.yaml`.

## When to Use

- "Build a BVS discovery questionnaire for [Account]"
- "What questions do we need to ask [Customer] to build the business case?"
- "Pre-populate the BVD questionnaire for [Account]"
- Before or after a discovery call — to frame the value conversation
- As a precursor to the BVS Executive Value Map

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `account_root` | Base directory for account workspace folders — used to read `account.md`, `account-research.md`, `northstar-pov.md`, `pccwop.md`, and any existing `drivers.yaml` in Phase 2, and to write driver stubs back to `drivers.yaml` in Phase 7. **Do not hardcode a personal path in this skill.** |

If `account_root` is missing, ask the user for the account folder path (or default to the current working directory) before reading or writing files.

**Unresolved — no matching canonical key:** the Q-Identity Solutions Workspace org ID (`00D8c000003ECzNEAW`, referenced immediately below) has no corresponding key in the canonical config list. It identifies Salesforce's own internal Solutions Workspace org — not a customer org or PII — so it is kept as a literal reference rather than externalized. Flagging here per the de-personalization rules rather than inventing a new config key.

## Q-Identity Status

**Q-Identity (Solutions Workspace `00D8c000003ECzNEAW`) is not yet connected.**

When connected, value drivers will be sourced exclusively from `ROIDataGrouping__c` records filtered by product and industry — the BVS system of record.

Until connected: value drivers are derived from account research (Org62, Slack, account folder, web) using the product-to-driver mapping table below. All drivers are marked `🤔 Research-Derived` and should be validated against Q-Identity before a formal BVS engagement.

**Q-Identity plug-in point** — when the connector is available, replace Phase 3 with:
```
FIND {[product name]}
IN ALL FIELDS
RETURNING ROIDataGrouping__c(
    Id, Name, Type__c, Category__c, Description__c,
    MinBenefit__c, MaxBenefit__c
    WHERE ROICalculator__r.Stage__c = 'Active Template'
)
SELECT ROIDataPoint__r.Name, ROIDataPoint__r.Description__c,
    ROIDataPoint__r.DataPointType__c, ROIDataPoint__r.Formula__c,
    ROIDataPoint__r.UniqueName__c
FROM ROIDataPointGroupItem__c
WHERE ROIDataGrouping__c = '{dataGroupId}'
ORDER BY SortOrder__c ASC NULLS LAST
```
Filter: `Type__c IN ('Cost Savings', 'Revenue Uplift')` and `BVD3Duplicate__c != TRUE`.
Classify data points: Input (Formula__c null) vs Calculated (Formula__c populated). Only ask Inputs.

---

## Execution Flow

### Phase 1 — Identify Account & Opportunity

Parse account name and/or opportunity from the request.

1. Query Org62 for the account:
```soql
SELECT Id, Name, Industry, AnnualRevenue, NumberOfEmployees,
       BillingCity, BillingCountry, Description,
       Challenges__c, Customer_Use_Case__c,
       Primary_Customer_Objective__c, Targeted_Clouds__c
FROM Account
WHERE Name LIKE '%[account name]%'
LIMIT 5
```

2. Query open opportunities:
```soql
SELECT Id, Name, StageName, Amount, CloseDate,
       (SELECT Product2.Name, Product2.Family FROM OpportunityLineItems)
FROM Opportunity
WHERE AccountId = '[AccountId]'
AND IsClosed = false
ORDER BY Amount DESC NULLS LAST
```

3. Confirm with user:
- Account name, industry, revenue, employee count
- Proposed Salesforce products (from opportunity line items or user input)
- If products are ambiguous, ask before proceeding

### Phase 2 — Gather Intelligence (run in parallel)

**Account folder:** Check for `<account_root>/<slug>/`. If exists, read:
- `account.md` — strategic context, stakeholders, discovery signals
- `account-research.md` — business model, pain points, frameworks
- `northstar-pov.md` — strategic objectives, obstacles
- `pccwop.md` — pain, cost, what-if narrative
- `drivers.yaml` (if exists) — any previously identified drivers

**Slack:** Search for account name in recent messages and files. Look for: discovery notes, call transcripts, pain points, QBR summaries, existing BVCs or ROI analyses.

**Web:** Company overview, recent news, strategic priorities from website and press releases.

**Org62 activities:** Recent tasks and events for context on pain points discussed.

### Phase 3 — Map Value Drivers (Research-Derived)

Select 5–8 most relevant value drivers based on the customer's industry, pain points, and proposed products. Use the mapping table below as a starting point, filtered to what's most relevant for THIS customer based on Phase 2 intelligence.

**Product-to-Driver Mapping (Research-Derived — pending Q-Identity)**

| Salesforce Product | Value Driver Candidates |
|---|---|
| **Field Service (FSL)** | Decrease technician admin time, Increase first-time fix rate, Improve technician utilization, Reduce truck rolls, Decrease time-to-dispatch, Improve customer SLA compliance |
| **Service Cloud** | Decrease case opening time, Decrease case handle time, Increase agent productivity, Reduce case escalations, Improve CSAT/NPS |
| **Sales Cloud** | Increase win rate, Reduce sales cycle length, Increase pipeline visibility, Improve forecast accuracy, Reduce rep ramp time |
| **Agentforce** | Reduce inbound inquiry volume (deflection), Decrease case resolution time, Increase self-service adoption, Reduce after-hours escalations |
| **Experience Cloud** | Increase self-service deflection, Reduce customer status inquiry calls, Improve customer portal adoption |
| **Data Cloud** | Improve cross-sell/upsell identification, Reduce duplicate data and manual reconciliation, Improve marketing ROI |
| **Revenue Cloud** | Reduce quote cycle time, Decrease pricing errors, Improve contract compliance, Accelerate time-to-revenue |

For each selected driver, classify as:
- **Revenue Uplift** — improves top-line (win rate, upsell, retention, faster revenue)
- **Cost Reduction** — reduces cost or admin burden (handle time, rework, truck rolls)

Aim for balance: at least 2 revenue uplift and at least 3 cost/efficiency drivers.

### Phase 4 — Pre-Populate Known Values

For each value driver and its input data points, attempt to pre-populate from Phase 2 research:

| Input Type | Pre-Population Source |
|---|---|
| Headcount (e.g. # technicians, # agents) | Org62 account fields, discovery call notes, account.md |
| Current metric (e.g. current utilization rate, handle time) | Discovery call notes, pccwop.md, account-research.md |
| Cost rates (e.g. fully-loaded hourly rate) | Discovery notes, industry benchmark if flagged as assumption |
| Volume metrics (e.g. jobs/month, cases/week) | Discovery call notes, account-research.md |

Tag every pre-populated value with its source. Append " (pre-filled from [source] — please validate)" to the description.

If a value cannot be sourced, leave the Value cell blank for the customer to fill in.

### Phase 5 — Validation

Before generating output, verify:
- At least 4 value drivers included
- Every driver has at least 2 input rows
- No calculated values included as questions (inputs only)
- Driver names are in plain customer language (no internal jargon)
- All inputs are traceable to a source or clearly marked as blank

### Phase 6 — Generate Google Sheet

Create a Google Sheet:
- **Title:** `[Account Name] — Salesforce Business Value Discovery Questionnaire`
- **1 tab only** — no intro tab, no instructions tab
- **3 columns:** `Input Name` | `Value` | `Description`
- Bold column header row as first row

For each value driver:
- Section header row: driver name in first cell, **bold, ALL CAPS**
- One row per input with:
  - `Input Name`: plain customer-friendly label
  - `Value`: pre-populated if known, blank if not
  - `Description`: what to enter + "(pre-filled from [source] — please validate)" if pre-populated
- Blank row after last input of each driver (before next driver header)

### Phase 7 — Write driver stubs to drivers.yaml (if demo-project active)

If `.demo-project/drivers.yaml` exists in the current directory:

For each value driver identified, append a stub if no matching driver exists:

```yaml
- id: DRV-[NNN]    # next sequential ID
  title: "[driver title]"
  objective_ref: null    # TODO: link to OBJ-### after objectives are defined

  user_story:
    as_a: "[role from account research]"
    i_want: "[capability]"
    so_that: "[business outcome]"

  bvs_impacts:
    - lever_type: "[revenue|cost]"
      dimension: "[driver category]"
      baseline: ""              # TODO: fill from questionnaire responses
      baseline_source: "🤔 Assumption — pending customer validation"
      improvement_formula: ""   # TODO: fill after value map
      improvement_pct: ""
      improvement_source: "🤔 Assumption — pending benchmarking"
      profit_impact_pa: ""      # TODO: fill after value map
      cs_metrics:
        label: ""
        p25: null
        average: null
        p75: null
        source: ""
      customer_references: []
      q_identity_ref: null      # TODO: populate when Solutions Workspace connected

  stakeholders_affected:
    - name: ""
      role: ""
      impact: ""

  salesforce_products:
    - product: ""
      capability: ""

  priority: high
  status: draft    # draft | active | deferred | out-of-scope
  notes: "Auto-generated stub from BVS Discovery Questionnaire — validate and complete"
```

### Phase 8 — Deliver

Post a brief response with:
1. Link to the Google Sheet
2. Summary: proposed products, number of drivers included, what's pre-filled vs blank
3. What the account team should confirm before sending to the customer (any placeholder assumptions)
4. Offer to run the BVS Executive Value Map once inputs are collected

---

## Key Principles

- **Inputs only** — never include a calculated output as a question
- **Pre-populate everything possible** — every blank is work we're asking the customer to do; every pre-filled value signals we've done our homework
- **Customer language** — rewrite field names into plain business language; no Salesforce jargon, no internal acronyms
- **Source every pre-populated value** — tag with origin so the customer knows where it came from
- **Q-Identity first when available** — research-derived drivers are a fallback, not the goal; mark all as `🤔 Research-Derived`
- **Fail loudly** — if no drivers can be mapped to a product, say so explicitly rather than inventing content
- **Driver stubs are drafts** — entries written to drivers.yaml need human review before being promoted to `active`
- **Connected Vision voice** — discovery output (driver labels, customer language captured) feeds CV artifact; see `plugins/se-account-intelligence/skills/se-connected-vision/connected-vision-voice.md` for how that language should be applied downstream
