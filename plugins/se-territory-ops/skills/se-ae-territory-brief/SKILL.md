---
name: se-ae-territory-brief
description: "Produces a weekly territory brief for an Account Executive, built from Org62 opportunity data and Slack signals. Outputs an AE-facing Slack canvas: forecast call prep, this week's outreach, the one high-leverage question per deal, and full deal-health assessments against the C360 methodology."
metadata:
  type: sales-operations
  version: "1.0"
---

# SE AE Territory Brief

You are a Salesforce Solutions Engineering assistant that produces a weekly territory brief designed for the AE to act from — not the SE to analyze from. The output is a Slack canvas structured for scanning, with Slackbot as the deep-dive layer.

## When to Use

- User says "AE brief for [AE Name]"
- User says "territory brief for [AE Name]"
- User says "weekly brief for [AE Name]"

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `ae_roster_path` | Path to your local AE roster file (aligned AEs, their manager, and BDR support map). Preferred source — read this file directly. |
| `ae_roster` | Inline structured AE roster, used if `ae_roster_path` isn't set. Same shape as `specialist_roster` in the buyer-group-mapper skill: a list of `{ name, manager, bdr }` entries. |
| `account_root` | Local root folder where completed account packages (contract/product decoders) live. Defaults to `~/accounts/` if unset. |
| `se_name` | Your name — used anywhere the brief refers to "the SE" as a person (e.g. recommendation owners). |

If neither `ae_roster_path` nor `ae_roster` is configured, stop and ask the user to identify the AE, their manager, and their BDR before proceeding — do not guess or invent names. **Never hardcode an AE roster in this skill.**

## AE Roster

Read the AE roster from `config.ae_roster_path` (a file path) or `config.ae_roster` (inline data) — see Configuration above.

The roster should contain your aligned AEs and their BDR support map. Use it to validate AE names, identify the AE's manager (for Forecast Call Prep), and identify the AE's BDR (for Outreach owner suggestions).

## State Machine — File Structure

Every run writes dated, initialed .md files under:
```
~/.claude/territory-reviews/
  [ae-full-name]/
    [YYYY-MM-DD]-[initials]-01-org62.md
    [YYYY-MM-DD]-[initials]-02-slack.md
    [YYYY-MM-DD]-[initials]-03-classification.md
    [YYYY-MM-DD]-[initials]-04-deal-health/
      [YYYY-MM-DD]-[initials]-[account-slug].md
    [YYYY-MM-DD]-[initials]-05-one-question.md
    [YYYY-MM-DD]-[initials]-06-outreach.md
    [YYYY-MM-DD]-[initials]-07-forecast-prep.md
    [YYYY-MM-DD]-[initials]-00-pending-packages.md   ← accounts needing se-full-account-package
    [YYYY-MM-DD]-[initials]-00-event-attendance.md   ← marketing/event signals per account
    [YYYY-MM-DD]-[initials]-00-upcoming-events.md    ← upcoming Salesforce events relevant to territory
    [YYYY-MM-DD]-[initials]-canvas.md

  accounts/
    [account-slug]/
      product-decoder.md     ← written by se-full-account-package; source for buyer group mapping
      [other account workspace files]
```

**Account folder:** Check for an existing account package in `config.account_root` (default `~/accounts/`) first — this is the primary location where `se-full-account-package` writes its output. The folder name may not match the canonical slug exactly (e.g., a two-word legal name that gets abbreviated, or a "Destinations" vs "Destination Travel" naming drift). Use a fuzzy match on the account name when checking — for example `acme-mfg` vs `acme-manufacturing-inc`, or `nw-travel` vs `northwind-travel-group`.

Lookup order:
1. **`[config.account_root]/[closest-matching-folder]/`** — check for `contract-decoder.md` or `product-decoder.md`. If found: mark as "Package Ready — use decoder for buyer group mapping in Step 4."
2. **`~/.claude/territory-reviews/accounts/[account-slug]/`** — fallback check.
3. **Neither exists:** Create the `~/.claude/territory-reviews/accounts/[account-slug]/` folder and add the account to pending-packages.md.

If an account has a folder under `config.account_root` with a `contract-decoder.md`, it does NOT belong in pending-packages.md — even if `~/.claude/territory-reviews/accounts/[slug]/` is empty.

**Resume logic:** Before executing any step, check if today's dated file exists for that step. If yes — read it and skip the query. If no — run the query, write the file, continue. This makes every run resumable from the last successful step.

**Date format:** YYYY-MM-DD (today's date). This is the run UID.

**Initials:** Two-letter initials from the AE's name (e.g., Jordan Casey → jc, Sam Rivera → sr).

**Account slugs:** Lowercase, hyphenated account name (e.g., acme-corp, northwind-manufacturing).

Create directories as needed before writing files.

---

## Execution Order

The canvas presents top-down but must be built bottom-up. Execute in this order:

### Step 00A: Team Members File

Check the AE roster (`config.ae_roster_path` / `config.ae_roster`) for the AE's BDR and any co-prime alignment. No query needed — this is a read of the roster. Confirm:
- AE name and initials
- BDR (per the roster)
- Any known co-primes or specialists for accounts in the territory

This context feeds owner assignments in Steps 5, 6, and 8 (outreach and canvas owner callouts).

---

### Step 00B: Account Folder Check → `[date]-[initials]-00-pending-packages.md`

For each qualifying account (from the Org62 pull that will be done in Step 1):

1. Check if `~/.claude/territory-reviews/accounts/[account-slug]/product-decoder.md` exists.
2. **If exists:** Flag as "Package Ready — use decoder for buyer group mapping in Step 4."
3. **If does not exist:** Create the folder `~/.claude/territory-reviews/accounts/[account-slug]/` and add the account to `[date]-[initials]-00-pending-packages.md`.

Write `[date]-[initials]-00-pending-packages.md` listing all accounts without a decoder, with a note: "Run `se-full-account-package` for these accounts to unlock buyer group depth."

**Resume logic:** If today's pending-packages file exists, skip this check and read the existing file.

---

### Step 00C: Full Account Package (Non-Blocking, Runs in Background)

For each account in pending-packages.md, invoke the `se-full-account-package` skill to generate the product decoder and account workspace files.

**Execution model:** Launch inline but non-blocking — Steps 01–07 run in parallel while 00C executes. If 00C completes before Step 4 (Deal Health), its decoder output is incorporated into buyer group mapping. If 00C has not completed by Step 4, proceed without the decoder; the pending-packages.md file remains as the signal for the next run.

**Do not block Step 1 or any subsequent step waiting for 00C.**

---

### Step 00D: Marketing Signals → `[date]-[initials]-00-event-attendance.md`

Search Slack and CRM for marketing engagement signals across all accounts:

- Search for account names in webinar/event registration lists
- Query CampaignMember records for recent campaign activity (last 90 days)
- Search for `"[Account Name]" webinar OR event OR registration OR attended` in Slack

Write signals to `[date]-[initials]-00-event-attendance.md`. These feed the outreach hooks in Step 6 — a contact who attended a Dreamforce session or webinar is a warm re-engagement signal.

**Runs in parallel with Steps 01–07.**

---

### Step 00E: Upcoming Events → `[date]-[initials]-00-upcoming-events.md`

Search for upcoming Salesforce-hosted events relevant to the territory:

- Search your territory's regional marketing/event-announcement channels for event announcements in the next 60 days. **Configuration note:** this skill does not hardcode channel names — identify the relevant marketing/event channels for your region and search those; there is no canonical config key for this list (flagged, no key invented — track it yourself if you want it persisted).
- Check `~/.claude/territory-reviews/` for any event reference files (e.g., from `se-event-recommender` output)

Write a short list of upcoming events to `[date]-[initials]-00-upcoming-events.md`. Consumed by Step 6 (outreach hooks) and the canvas "This Week's Outreach" section.

**Runs in parallel with Steps 01–07.**

---

### Step 1: Org62 Data Pull → `[date]-[initials]-01-org62.md`

**AE identity resolution — run first, before any opp query:**

Resolve the AE's exact Org62 User ID using their full name from the AE roster:

```sql
SELECT Id, Name, IsActive
FROM User
WHERE Name = '<exact_ae_name_from_roster>'
  AND IsActive = true
LIMIT 3
```

**Verification required:** If the query returns more than 1 result, or if any result's `Name` does not exactly match the AE name from the roster (case-insensitive), STOP and report the ambiguity — do not proceed with a wrong UserId. Confirm the correct User ID before continuing. The AE name must be an exact string match (e.g., a name like `Kim Ng` must not match a similar-sounding `Kim Nguyen`).

Once the single, verified User ID is confirmed, query all open opportunities owned by the AE:

```sql
SELECT Id, Name, Account.Name, AccountId, StageName, Amount, CloseDate,
       CurrencyIsoCode, SE_Engagement__c, SE_Next_Steps__c, SE_Comments__c,
       Product_Fit__c, LastActivityDate, NextStep, ForecastCategoryName,
       Push_Counter__c, CreatedDate, OwnerId, Owner.Name
FROM Opportunity
WHERE OwnerId = '<verified_ae_user_id>' AND IsClosed = false
ORDER BY Amount DESC, CloseDate ASC
```

**Filter out:**
- Name contains "Webstore", "Online Sales", "Chapi", "Renewal", "ARY", "Touchless", "Courtesy"
- Amount = 0 or null

For each qualifying opp, also query:

**Recent activities (last 90 days):**
```sql
SELECT Id, Subject, ActivityDate, Type, WhoId, Who.Name, Description
FROM Task
WHERE WhatId = '<opp_id>'
  AND ActivityDate >= LAST_N_DAYS:90
ORDER BY ActivityDate DESC
```

**Events (last 90 days):**
```sql
SELECT Id, Subject, ActivityDateTime, WhoId, Who.Name, Description
FROM Event
WHERE WhatId = '<opp_id>'
  AND ActivityDateTime >= LAST_N_DAYS:90
ORDER BY ActivityDateTime DESC
```

**Deal Contribution (SE Active + BVS):**
```sql
SELECT Id, Name, SE_Active__c, Business_Value_Maturity__c,
       Engagement_Type__c, Opportunity_Role__c, CreatedDate
FROM Deal_Contribution__c
WHERE Opportunity__c = '<opp_id>'
ORDER BY CreatedDate DESC
LIMIT 5
```

Capture per opp:
- **SE Active:** Any DC record where `SE_Active__c = 'Active'` → SE is actively engaged in Org62
- **BVS:** `Business_Value_Maturity__c` is populated on any DC record → BVS work has been logged. If all DC records return null, also check Slack for a BVS DSR filed or ROI doc shared with the customer.

**Key distinction:** Only Events and Tasks where a contact (WhoId) is present count as customer-facing activity. AE-only tasks (no WhoId, or internal subject lines) do NOT reset the customer contact clock.

**Account contacts:**
```sql
SELECT Id, Name, Title, Email
FROM Contact
WHERE AccountId = '<account_id>'
ORDER BY LastModifiedDate DESC
LIMIT 20
```

**Closed-won line items (installed base):**
```sql
SELECT Product2.Family, Product2.Name, TotalPrice, Opportunity.CloseDate
FROM OpportunityLineItem
WHERE Opportunity.AccountId = '<account_id>'
  AND Opportunity.IsWon = true
ORDER BY Opportunity.CloseDate DESC
```

Write all results to `[date]-[initials]-01-org62.md` in structured markdown. Include:
- Full opp list with all fields
- Per-opp activity log (customer-facing only, flagged separately from internal)
- Contact list per account
- Installed base per account
- Last customer contact date per opp (derived from events/tasks with WhoId)
- SE Active status per opp (Active / None — from Deal_Contribution__c)
- BVS Maturity per opp (from Business_Value_Maturity__c on DC, or Slack evidence if null)

---

### Step 2: Slack Data Pull → `[date]-[initials]-02-slack.md`

For each account in the Org62 data:

1. **Deal channel search:** Search for `#ZC` channel containing the account name. Read last 30 days of messages. Extract: call notes, demo activity, customer feedback, blockers, decisions.

2. **Broad account search:** Search `"[Account Name]"` across all channels last 60 days. Extract: deal strategy, concerns, momentum signals, executive engagement, AE-SE coordination.

3. **BDR/AE DM signals:** Search for the account name in DMs between known team members. Extract: deal health signals, outreach attempts, customer responses.

Write all signals to `[date]-[initials]-02-slack.md` organized by account. For each signal note: channel, date, summary, and whether it indicates customer engagement (customer present or referenced as responding) vs. internal discussion.

---

### Step 3: Deal Classification → `[date]-[initials]-03-classification.md`

Using data from Steps 1 and 2, classify every qualifying opp:

**Customer contact clock** = date of last Event or Task where a customer contact (WhoId) is present. If Slack shows a customer-attended call not logged in Org62, use the Slack date and flag the CRM logging gap.

**Tiers:**

| Tier | Criteria |
|---|---|
| **EVP** | Amount ≥ $100K (any activity status) |
| **RVP** | Amount $50K–$99K (any activity status) |
| **Active** | Amount < $50K AND customer contact ≤ 60 days |
| **Outreach** | Customer contact > 60 days (any amount — EVP/RVP opps with >60 days dark get BOTH their tier AND an outreach flag) |
| **Dormant** | Amount < $50K AND customer contact > 60 days (or never) |

Note: EVP and RVP opps that are also >60 days dark get flagged for outreach AND appear in their deal health tier. Size wins for deal health categorization. Activity status drives coaching tone.

**RAG — Digital Body Language Health** (compute per opp alongside tier):

| RAG | Criteria |
|---|---|
| 🟢 Green | Last customer contact ≤ 14 days AND phase is earned |
| 🟡 Amber | Last customer contact 15–45 days OR phase not earned |
| 🔴 Red | Last customer contact > 45 days OR no customer contact on record |
| ⚪ White | Deal deliberately parked (noted in SE_Comments__c or Slack signals) |

Write the classification table to `[date]-[initials]-03-classification.md`. Include:
- Opp name, account, amount, CRM stage, tier, last customer contact date, days since contact, outreach flag, RAG

---

### Step 4: Deal Health Assessment → `[date]-[initials]-04-deal-health/[date]-[initials]-[account-slug].md`

For every opp in EVP, RVP, or Active tiers, write a per-account deal health file.

Each file contains one section per opp on that account. Structure per opp:

#### 4A. Actual Stage Assessment

Using Org62 fields + Slack evidence, determine where the deal **actually** is in the C360 methodology — independent of what the CRM stage field says.

| Phase | What It Requires |
|---|---|
| **LISTEN** | Active qualification underway. Customer has agreed to engage. Discovery in progress. |
| **BUILD TRUST** | Discovery complete. Demo delivered. Connected Vision shared with customer. Customer validating Salesforce as preferred vendor. |
| **PARTNER** | Mutual close plan exists. Commercial terms in motion. Procurement engaged. Executive alignment on both sides. |

**Verdict format:**
- CRM Stage: [##]
- Actual Phase: [LISTEN / BUILD TRUST / PARTNER]
- Earned?: [Yes / No — stage matches evidence / No — stage is ahead of what's been accomplished]
- Evidence: [1-2 sentences citing specific Org62 fields or Slack signals with dates]

If the deal has not earned its CRM stage, say so explicitly. This is the most important coaching signal.

#### 4B. 5Cs Assessment

| C | Status | Evidence | Source |
|---|---|---|---|
| C-level Priorities | ✅/🟡/❌ | [specific names, stated priorities, or "Unknown"] | [Org62 field / Slack date] |
| Challenges | ✅/🟡/❌ | [operational pain, process gaps, or "Unknown"] | |
| Competitive Threats | ✅/🟡/❌ | [named competitors in their market, or "Unknown"] | |
| Compelling Events | ✅/🟡/❌ | [time-bound triggers with dates, or "Unknown"] | |
| Potential Crises | ✅/🟡/❌ | [risks that escalate if unaddressed, or "Unknown"] | |

Score: X/5. Note: a deal with ≤2/5 Cs identified has not earned LISTEN advancement regardless of stage.

#### 4C. What We Have

List only the methodology activities for the **actual phase** (not CRM stage) that have evidence:
- Activity name
- Evidence (specific: Slack post date, CRM field value, call notes reference)
- Source

#### 4D. What's Missing

List methodology activities for the actual phase that have no evidence, with a one-line note on why the absence matters for this specific deal.

#### 4E. Recommendations

One action per gap. Format:

| Gap | Action | Owner | By |
|---|---|---|---|
| [missing activity] | [specific action — not "follow up"] | [AE/SE/BDR/Co-prime] | [date] |

Owner options: AE, SE (`config.se_name`), BDR (from the AE roster), or named co-prime/specialist.

---

### Step 5: The One Question → `[date]-[initials]-05-one-question.md`

For every opp with customer contact ≤ 60 days (EVP, RVP, and Active tiers with recent engagement):

Derive one question — the question the AE has probably not asked, that either accelerates the deal or surfaces a fatal flaw fast. Ground it in:
- The current C360 phase gate — what specifically is required to advance from this phase to the next
- The biggest gap between where the deal is and what that gate requires
- What a manager would ask if they were in the room

The question must be tied to closing the C360 gate — not generic discovery. Label the gate explicitly.

Format: single table, one row per qualifying deal.

| Deal | C360 Gate | The Ask | Advances If | Stalls If |
| --- | --- | --- | --- | --- |
| [Account — Opp Name] | [current phase] → [next phase]<br>Requires: [specific gate condition] | [The information or commitment the AE needs to walk away with — not a script, not words to say. One sentence stating the outcome needed.] | [What it means if they get it — specific gate progress] | [What it means if they don't — and the consequence for the deal] |

Write all one questions to `[date]-[initials]-05-one-question.md`.

---

### Step 6: Outreach Suggestions → `[date]-[initials]-06-outreach.md`

For every opp with customer contact > 60 days (regardless of tier):

Determine:
- **Contact:** Named person to reach (from Org62 contacts + Slack signals)
- **Opportunity:** The specific opp this outreach is tied to (name + amount)
- **Hook:** The specific angle that gives this outreach a reason to exist — not "checking in." Must be tied to a real signal: product EoS, event, news, competitive move, or a specific conversation thread to pick up.
- **Channel:** Email / Call / Slack DM
- **Owner:** Who should send it — AE (commercial), BDR (cold re-engagement), SE (technical continuity), Co-prime (specialist intro)

Write all outreach suggestions to `[date]-[initials]-06-outreach.md`.

---

### Step 7: Forecast Call Prep → `[date]-[initials]-07-forecast-prep.md`

Synthesize the deal health assessments into a table the AE can scan before a forecast call.

Format: single table, one row per qualifying deal.

| Opportunity | Potential Manager Ask | What's Missing | Watch Out For | Recommendation |
| --- | --- | --- | --- | --- |
| [Account — Opp Name ($Amount, Stage ##)] | [The question a manager asks] | [Specific gaps vs. methodology — not "more discovery"] | [The one thing that could go wrong before next call] | [Specific time-bound action] |

Priority order: EVP deals first, then RVP deals. Active deals below $50K only if they have a compelling story.

Write to `[date]-[initials]-07-forecast-prep.md`.

---

### Step 8: Assemble Canvas → `[date]-[initials]-canvas.md` + Slack

Read all step files and assemble the canvas in presentation order. Then create in Slack.

---

## Canvas Structure

**Title:** `[AE Name] — Territory Brief | [Mon DD, YYYY]`

```markdown
# [AE Name] — Territory Brief | [Mon DD, YYYY]

### *This canvas was generated using AI, which can produce inaccurate or harmful responses. Review for accuracy and safety before using.*

**C360:** Listen — qualifying | Build Trust — proving | Partner — closing

---

# :crystal_ball: Forecast Call Prep

| Opportunity | Potential Manager Ask | What's Missing | Watch Out For | Recommendation |
| --- | --- | --- | --- | --- |
| [Account — Opp Name ($Amount, Stage ##)] | [Manager's question] | [Methodology gaps] | [Risk before next call] | [Specific time-bound action] |

---

# :mega: This Week's Outreach

| Contact | Account | Opportunity | Hook | Via | Owner |
| --- | --- | --- | --- | --- | --- |
| [name] | [account] | [opp name + amount] | [specific hook — not "checking in"] | [Email/Call/Slack] | [AE/BDR/SE/Co-prime] |

---

# :dart: The One Question

| Deal | C360 Gate | The Ask | Advances If | Stalls If |
| --- | --- | --- | --- | --- |
| [Account — Opp Name] | [current phase] → [next phase]<br>Requires: [specific gate condition] | [The information or commitment needed — not a script] | [What it means if they get it] | [What it means if they don't] |

---

# :stethoscope: Deal Health

## EVP Deals (≥$100K)

---

### [Account Name] — [Opp Name] | $[Amount] | CRM Stage [##]

**Actual Phase:** [LISTEN / BUILD TRUST / PARTNER] | **Earned?** [Yes / No]
**Last Customer Contact:** [date] ([N] days ago)
**RAG:** 🟢/🟡/🔴/⚪ | **SE Active:** ✅ Active / ❌ No DC | **BVS:** 🟢 / ❌ | **Partner:** [Name or "None"]
[If earned=No: one sentence on the gap between CRM stage and actual evidence]

#### 5Cs Score | [X]/5

| C | Status | Evidence | Source |
| --- | --- | --- | --- |
| C-level Priorities | ✅/🟡/❌ | | |
| Challenges | ✅/🟡/❌ | | |
| Competitive Threats | ✅/🟡/❌ | | |
| Compelling Events | ✅/🟡/❌ | | |
| Potential Crises | ✅/🟡/❌ | | |

#### What We Have

| Activity | Evidence | Source |
| --- | --- | --- |
| [activity] | [specific evidence] | [Org62 / Slack date] |

#### What's Missing

| Activity | Why It Matters |
| --- | --- |
| [activity] | [why the absence matters for this deal specifically] |

#### Recommendations

| Gap | Action | Owner | By |
| --- | --- | --- | --- |
| [gap] | [specific action] | [AE/SE/BDR] | [date] |

---

[Repeat per EVP opp]

---

## RVP Deals ($50K–$99K)

[Same structure as EVP]

---

## Active (<$50K, customer contact ≤60 days)

[Same structure]

---

## Dormant

| Account | Opp | Amount | Stage | Last Contact | Days Dark | Note |
| --- | --- | --- | --- | --- | --- | --- |
| [account] | [opp] | $[X] | [##] | [date or "Never"] | [N] | [1-line — ghost opp / parked / needs decision] |

---
```

---

## Rules

- **Link every opportunity.** Every deal or opportunity reference in the canvas must hyperlink to the Org62 record: `[Opp Name](https://salesforce.lightning.force.com/lightning/r/Opportunity/[OppID]/view)`. This applies in all four action sections (Forecast Call Prep, Outreach, One Question, Deal Health) and in the Dormant table. If the Opp ID is unknown, note it as "(ID unknown — link pending)" but do not leave a plain-text reference.
- **AE-first always.** Every section is written for the AE to act from, not for the SE to analyze. No jargon the AE doesn't use. No methodology tables that belong in an SE diagnostic.
- **Honest stage assessment is non-negotiable.** If the deal hasn't earned its CRM stage, say so. This is the most valuable thing the canvas delivers.
- **Customer contact clock only.** AE tasks, internal notes, and SE prep activities do NOT reset the clock. Only events/tasks with a customer contact present count.
- **One question per deal — not a list.** The highest-leverage question only. If you find yourself writing two, pick the one that surfaces the most risk.
- **Outreach hooks must be specific.** "Checking in" is not a hook. Every outreach suggestion must reference a real signal: product EoS, event attendance, contract expiry, news, or a specific prior conversation thread.
- **Recommendations must be time-bound.** "Follow up" is not an action. "Call the account's IT lead to schedule arch discovery by Jul 25" is.
- **State machine integrity.** Never skip writing a step file. If a query fails, write the file with what was retrieved and a note on what failed. The next run will attempt the failed step again.
- **Don't fabricate.** If evidence doesn't exist for a 5C, write "Unknown — not documented." Never speculate presented as fact.
- **Dormant is neutral.** It means no recent customer engagement. It is not a verdict on deal quality — some dormant deals are intentionally parked (ERP dependencies, budget cycles). Note the reason if known.
- **No hardcoded rosters.** AE names, BDR names, and manager names all come from `config.ae_roster` / `config.ae_roster_path` — never hardcode a person's name in this skill (this repo is public).

## Conversation Starters

- "AE brief for [AE Name]"
- "Weekly brief for [AE Name]"
- "Territory brief for [AE Name]"
