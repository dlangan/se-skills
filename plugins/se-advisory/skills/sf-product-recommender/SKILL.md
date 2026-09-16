---
name: sf-product-recommender
description: "Given a capability description or RFP requirement, reads ~/product-brain/ to identify which Salesforce product(s) cover it, which specific PLM entitlements justify the score, and produces a score (0–3) + 250-word comment. Works for single requirements, bulk RFP tabs, or open-ended capability questions. Product Brain required at ~/product-brain/."
metadata:
  type: sales-operations
  version: "1.1"
  depends_on: "product-brain (~/product-brain/)"
  origin: "Built 2026-07-30 during an RFP scoring session for a named account. Companion to se-entitlement-decoder."
---

# SF Product Recommender

You answer the question every SE, AE, and RFP responder needs: **"Which Salesforce product covers this capability, and what specific entitlement proves it?"**

Salesforce has 364 base products across 20 families. This skill navigates the Product Brain (`~/product-brain/`) to find the right product(s), cite the actual PLM entitlement(s) that justify the recommendation, and produce a scored, defensible answer — no guessing, no fabrication.

---

## Configuration

This skill **hard-requires** a bundled Product Brain at `~/product-brain/` (or `~/brains/product-brain/`) — a local knowledge base of Salesforce product families and PLM entitlements. It is not optional: without it, this skill cannot score requirements or cite entitlements. Treat the brain path as a runtime-local reference (not a repo asset); if neither path exists, tell the user the Product Brain needs to be built/synced before this skill can run.

This skill does not read `~/.claude/se-config.json` — it has no account-owner, roster, or Slack identity to externalize. Per-deal supporting files (e.g. pre-GA entitlement notes for a specific account) live under the account's own workspace folder (`config.account_root` in the skills that manage that folder), not inside this skill.

---

## When to Use

- "Which Salesforce product covers [capability]?"
- "What's the score for requirement [X.X] and why?"
- "Fill in the blank requirements in this RFP tab"
- "Does Salesforce have a native entitlement for [feature]?"
- "What product do I demo for this use case?"
- "Is this capability bundled or a separate SKU?"
- Whitespace analysis: "What does this customer not have that their use case needs?"
- Competitive displacement: "Which entitlement beats the competitor's claim here?"

---

## Inputs

- **Requirement** (required) — free text capability description, an RFP requirement number + text, or a list of requirements
- **Product stack** (optional) — the products already in scope for this deal (e.g. "Sales A1E, Marketing Cloud Engagement, MuleSoft, Maps, Shield, Slack"). Constrains recommendations to what's purchasable for this customer.
- **Mode** (optional) — `single` (one requirement), `bulk` (list of requirements), `rfp-tab` (full XLS tab), `open` (capability question without a score)

---

## Core Workflow

### Step 1: Parse the Requirement

Extract:
- The **core capability** being asked about (strip RFP boilerplate)
- The **scoring context**: 0 = not available, 1 = custom config, 2 = third-party, 3 = OOTB
- Any **conditional modifiers** (e.g. "bi-directional", "automated", "regulatory-grade")

### Step 2: Navigate the Product Brain

**Always start with the master index:**
```
Read ~/product-brain/INDEX.md
```

Identify the 1–3 most likely product families. Then read the family INDEX.md(s):
```
Read ~/product-brain/[family-slug]/INDEX.md
```

Then read the specific product file(s):
```
Read ~/product-brain/[family-slug]/[product-slug].md
```

**Family routing guide** (use as starting heuristic — always verify):

| Capability Theme | Start Here |
|---|---|
| CRM, pipeline, leads, accounts, contacts, forecasting | `sales-cloud/` |
| Service, cases, support, entitlements | `service-cloud/` |
| Portals, communities, self-service, external users | `communities/` |
| Email, SMS, journeys, campaign automation | `mc-exacttarget/` or `marketing-cloud/` |
| Analytics, dashboards, reports, BI | `analytics/` or `tableau/` |
| Integration, API, middleware, connectivity | `mulesoft/` or `mulesoft-anypoint-platform/` |
| Identity resolution, unified profiles, segmentation | `data-cloud/` or `custom-cloud/` |
| Territory management, geographic routing, maps | `sales-cloud/` (Maps products) |
| Collaboration, notifications, messaging | `slack/` |
| Security, encryption, audit, compliance | Look across `sales-cloud/`, `service-cloud/`, `full-crm/` for Shield products |
| Success, support plans, TAM | `success-cloud/` |
| Commerce, storefronts, ordering | `commerce-cloud/` |
| Industry-specific (manufacturing, health, FSC, etc.) | `full-crm/` |
| AI, Einstein, Agentforce | `sales-cloud/`, `service-cloud/`, or `custom-cloud/` |
| Platform, automation, Flow, low-code | `platform/` or `sales-cloud/` |

### Step 3: Match Capability to Entitlement

For each candidate product file, scan the entitlement table for the specific entitlement(s) that cover the requirement. Look for:

1. **Direct name match** — entitlement name contains a relevant keyword
2. **Functional match** — entitlement description covers the capability
3. **Bundled match** — capability is implied by a broader entitlement (e.g. `Sales Cloud Einstein` covers lead scoring, call summaries, activity capture)

If no entitlement in the product files clearly covers the capability:
- Check if it's a **platform primitive** (standard Salesforce functionality not represented in PLM — e.g. Reports, Dashboards, Flows, standard objects)
- Check if it requires a **third-party ISV** (DocuSign, Smartsheet, etc.)
- Check if it's **roadmap / pre-GA** — score accordingly

### Step 4: Determine Score

| Score | Condition |
|---|---|
| **3** | Covered by a named entitlement in a Product Brain file, OR by a Salesforce platform primitive (standard out-of-box functionality included with any license) |
| **2** | Covered by a listed third-party ISV on Salesforce AppExchange or as a certified integration partner (DocuSign, Smartsheet, Google Workspace, etc.) |
| **1** | Requires custom configuration with Flow, Apex, or custom objects — no native entitlement, but achievable within the platform |
| **0** | Not currently available — either pre-GA, not on roadmap, or genuinely out of scope |

**Score 3 defensibility rules:**
- Must cite the product name AND at least one specific entitlement OR name the platform primitive
- Pre-GA products score 3 only if GA is within the current evaluation window — add "GA [Month Year]" in the comment
- Do not score 3 for a capability that requires ISV integration as the primary delivery mechanism

### Step 5: Draft the Comment

**Format:** 250 words max (hard limit established for a specific customer RFP; apply to all RFP contexts unless overridden)

**Structure:**
1. **Lead sentence:** State what Salesforce delivers natively — name the product
2. **Entitlement citation:** Name the specific entitlement(s) that prove it
3. **How it works:** 2–3 sentences on the mechanism — concrete, not marketing
4. **Recipe/customer context** (if known): 1 sentence connecting to their specific scenario
5. **If ISV involved:** name the partner, describe the integration depth
6. **If expansion required:** flag it clearly but don't lead with it

**Tone rules:**
- No competitor names
- No "SI" — use "implementation partner"
- No "Salesforce Sign" — deprecated
- No "CRM Analytics" — use "Tableau Next"
- "Available as an add-on" for Shield
- "Expansion capability" for Data Cloud if customer doesn't own it
- "GA [Month Year]" for pre-GA products — never "next quarter"
- No URLs except where RFP explicitly requests them

---

## Output Format

### Single Requirement Mode

```
## Requirement [X.X] — [Short capability label]

**Score:** [0–3]
**Y/N:** [Y / N / Partial]
**Product(s):** [SKU name(s)]
**Entitlement(s):** [Specific entitlement name(s) from Product Brain]

**Comment:**
[250-word max answer]

**Source:** ~/product-brain/[family]/[product].md
```

---

### Bulk Mode (multiple requirements)

Output a table first for scanability, then full comments:

```
## Score Summary

| Req # | Capability (short) | Score | Product(s) | Key Entitlement |
|---|---|---|---|---|
| 1.18 | Cross-brand candidate profile | 3 | Sales A1E | Account Hierarchy + Data Cloud Starter |
| 1.25 | Rule-based lead assignment | 3 | Sales A1E | Assignment Rules (platform primitive) |
| ... | | | | |

---

## Full Comments

### 1.18 — Cross-brand candidate profile
[Full comment...]

### 1.25 — Rule-based lead assignment
[Full comment...]
```

---

### Open Mode (capability question, no score needed)

```
## [Capability Question]

**Best Product Match:** [Product name] — [family]
**Entitlement:** [Name] — [what it means]
**Also Consider:** [any adjacent products]
**Bundled or Separate SKU:** [bundled in X / separate SKU / add-on]

**How It Works:**
[3–5 sentences]
```

---

## RFP Tab Mode

When given a full tab of requirements to complete:

1. Read all requirements, identify which already have scores vs. which are blank
2. Group blank requirements by capability theme (routing guide above)
3. Read Product Brain family indexes in parallel for all relevant families
4. For each requirement: score + comment using the workflow above
5. Output: score summary table + full comments, ready to paste into XLS

**Do not re-score requirements that already have scores** unless explicitly asked.

---

## Product Brain Navigation Rules

1. **Always read INDEX.md first** — never guess a file path
2. **Read the family INDEX before individual files** — the index sorts by entitlement count, which tells you which products are richest
3. **Cross-family requirements are common** — a single requirement may touch `sales-cloud/` (base CRM) + `mc-exacttarget/` (email) + `slack/` (notification). Read all relevant families.
4. **Platform primitives are real** — Reports, Dashboards, Flows, Assignment Rules, Approval Processes, Record Types, Sharing Rules, Field-Level Security — these are standard Salesforce functionality included with any license. They don't appear in PLM files because they're not add-ons. Score them 3 and name them explicitly.
5. **If the product file has 0 entitlements** — the product is likely a standalone or legacy SKU. The capability may still be real; describe it from the product name and notes section.
6. **Product Brain was built 30 Jul 2026** — pre-GA products (e.g. Agentforce for Marketing, GA Oct 2026) may not have Product2 records. Check `<config.account_root>/<account_name>/agentforce-marketing-entitlements.md` or similar per-deal files for pre-GA documentation.

---

## Integration with Other Skills

| Skill | How Product Recommender Feeds It |
|---|---|
| **se-entitlement-decoder** | Decoder tells you what a customer HAS; Recommender tells you what they NEED — together they close the whitespace gap |
| **sf-dse** | DSE uses Recommender output to build cross-cloud architecture narratives with specific entitlement citations |
| **se-deal-coach** | Recommender identifies which products to position and which entitlements to demo |
| **bvs-executive-value-map** | Recommender maps capabilities to products; Value Map maps products to business outcomes |
| **se-360-builder** | Footprint + whitespace sections get precise product recommendations with entitlement-level justification |
| **se-agentforce-use-case-advisor** | Recommender confirms which AI entitlements exist to support the agent use cases identified |

---

## Conversation Starters

- "Score requirement 1.37 from the [Account] RFP"
- "Which product covers FDD cooling-off period enforcement?"
- "Fill in all blank scores on Tab 2 of the Recipe XLS"
- "Does Salesforce have a native entitlement for territory visualization?"
- "What SKU covers bi-directional DocuSign integration?"
- "Is automated call transcription bundled or a separate add-on?"
- "Which product covers PIPEDA-compliant data retention?"
- "Recommend products for a franchise development CRM build"
