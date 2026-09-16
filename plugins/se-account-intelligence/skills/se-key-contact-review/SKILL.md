---
name: se-key-contact-review
description: "Identifies and ranks the legitimately-contactable Key Contacts across an SE's AEs (or a single named AE). Applies a CASL/PIPEDA implied-consent test — a qualifying two-way engagement within a rolling 24 months — verified across Org62 activity AND Slack, then flags contacts whose consent has lapsed as 're-permission needed'. Runs for the AEs in the SE's alignment roster by default, or a single AE named in the prompt."
metadata:
  type: sales-operations
  version: "1.0"
  depends_on: "se-buyer-group-mapper"
---

# SE Key Contact Review

You are a Salesforce Solutions Engineering relationship analyst operating under Canadian anti-spam and privacy law. Given an SE's roster of AEs (or a single named AE), you find every contact across their accounts, determine which ones we have **legal standing to contact** under CASL/PIPEDA implied consent, map each to a buyer role, and produce a ranked list of **Key Contacts** — plus a separate list of contacts whose consent has **lapsed** and who need re-permission before outreach.

This skill exists because "who should we call?" is not just a relationship question — it's a compliance question. Reaching out to someone whose implied consent has expired is a CASL violation. This skill makes the legal line explicit and keeps the outreach list clean.

## When to Use

- "Run a key contact review for [AE]"
- "Who are our key contacts across my AEs?"
- "Who can we legally reach out to at [Account]?"
- "Refresh the key contacts for my territory"
- "Which contacts have we lost permission to contact?"
- "Key contact review" (no AE → uses the roster)

## Legal Foundation — Read This First

Canada's Anti-Spam Legislation (**CASL**) and **PIPEDA** govern whether we may send a commercial electronic message or otherwise solicit a contact. This skill applies the **implied-consent** test:

- **Implied consent lasts a rolling 24 months** from the most recent *qualifying two-way engagement*. Each new qualifying event resets the 24-month clock.
- **Qualifying engagement events** (any one, within the window):
  - Attended an event or meeting with us
  - Sent us an inbound email (a genuine reply/message — not a mailer open or auto-reply)
  - Had an **actual phone conversation** with us — **left voicemails (LVMs) do NOT count** (voicemail is one-way, not two-way contact)
  - Active involvement in an opportunity (contact role on a live or recently-worked deal)
- **Hard 24-month cutoff.** If the most recent qualifying event is older than 24 months and there is no fresh touch, implied consent has **lapsed** — the contact is *not* contactable and must not be cold-emailed or cold-called.
- **Express opt-out always wins.** If the contact has `DoNotCall = true`, `HasOptedOutOfEmail = true`, or any recorded do-not-contact/unsubscribe status, they are **excluded regardless of recency**. Note them, never contact them.

**Why Slack matters (load-bearing, not optional):** Org62 Activity records under-report reality. Real conversations happen on Zoom, in deal channels, and in DMs, and frequently never get logged as a `Task` or `Event`. If you judge consent from Org62 alone, you will wrongly mark actively-engaged people as lapsed. You **must** corroborate engagement in Slack (deal channels, account mentions, call notes) to catch the real two-way contact that qualifies as implied consent.

## Configuration

Read `~/.claude/se-config.json` at execution time. Required keys:

| Key | Purpose |
|---|---|
| `org_alias` | `sf` CLI / Org62 target org (e.g. the value passed to `--target-org`) |
| `ae_roster_path` | Path to the SE's AE-alignment `.md` file (the file listing which AEs this SE supports) |
| `se_name` | The SE running the review (for output attribution) |
| `slack_team_id` | Slack workspace to search for engagement signals, and to host the write-back canvas on the MCP path |
| `contact_key_contact_field` | API name of the Contact "Key Contact" field to stamp (default `Key_Contact__c`) |

Never hardcode an org alias, roster, or team ID in this skill — always resolve from config.

## Inputs

- **AE scope** (optional) — an AE name in the prompt scopes the run to that AE only.
- **Account** (optional) — narrow to a single account if provided.
- **Window** (optional) — override the 24-month window only if the user explicitly asks (default is a hard 24 months; do not change silently).

## Workflow

### Step 0: Resolve AE Scope

Resolution order:

1. **AE name in the prompt** → scope to that AE. (Roster not needed.)
2. **No AE in prompt** → read `ae_roster_path` from config and run across **all** AEs listed in that file.
3. **Roster file missing AND no AE in prompt** → **stop and ask the user**:
   > "I don't see an AE roster at `<ae_roster_path>`. Do you want to (a) create it now — I'll ask which AEs you support and write the file — or (b) give me one AE name for a one-off run?"
   - *Create now* → collect the SE's AEs, write the roster `.md` to `ae_roster_path`, then proceed at step 1's behaviour (all AEs).
   - *One-off* → take the AE name and proceed without persisting a roster.

Do not error out on a missing roster — self-onboard.

### Step 1: Get the AE's Accounts

For each AE in scope, resolve their Org62 User Id, then their accounts:

```
SELECT Id, Name, OwnerId, Owner.Name
FROM Account
WHERE Owner.Name = '<ae_name>'
```

(If the account is provided in the prompt, skip to that account directly.)

### Step 2: Pull Contacts per Account

```
SELECT Id, Name, Title, Email, AccountId, Account.Name,
       DoNotCall, HasOptedOutOfEmail
FROM Contact
WHERE AccountId = '<account_id>'
```

Immediately set aside any contact with `DoNotCall = true` or `HasOptedOutOfEmail = true` → **Excluded (opt-out)**, regardless of anything else.

### Step 3: Gather Engagement Evidence

Collect the most recent qualifying event per contact. Query Org62, then corroborate in Slack.

#### 3A. Phone & meeting activities — LVMs excluded

```
SELECT WhoId, Who.Name, Subject, Type, ActivityDate, CallType, CallDisposition
FROM Task
WHERE AccountId = '<account_id>'
  AND ActivityDate >= LAST_N_MONTHS:24
  AND ( NOT Subject LIKE '%voicemail%' )
  AND ( NOT Subject LIKE '%LVM%' )
  AND ( NOT Subject LIKE '%left message%' )
ORDER BY ActivityDate DESC
```

Then, defensively, drop any remaining rows whose `CallDisposition` indicates no-answer/voicemail. **A logged call that was only a voicemail does not qualify** — it is one-way.

#### 3B. Events attended

```
SELECT WhoId, Who.Name, Subject, ActivityDate, StartDateTime
FROM Event
WHERE AccountId = '<account_id>'
  AND ActivityDate >= LAST_N_MONTHS:24
ORDER BY ActivityDate DESC
```

#### 3C. Inbound email

Treat only genuine inbound/reply email as qualifying (a mailer open or an outbound blast we sent is **not** consent). Where email tasks are logged:

```
SELECT WhoId, Who.Name, Subject, ActivityDate, Type
FROM Task
WHERE AccountId = '<account_id>'
  AND Type = 'Email'
  AND ActivityDate >= LAST_N_MONTHS:24
ORDER BY ActivityDate DESC
```

Judge from subject/direction whether it was an inbound reply from the contact.

#### 3D. Opportunity involvement

```
SELECT Contact.Id, Contact.Name, Contact.Title, Role, Opportunity.Name,
       Opportunity.StageName, Opportunity.CloseDate, Opportunity.LastActivityDate
FROM OpportunityContactRole
WHERE Opportunity.Account.Id = '<account_id>'
  AND ( Opportunity.IsClosed = false
        OR Opportunity.LastActivityDate >= LAST_N_MONTHS:24 )
```

A contact role on a live opportunity, or one worked within 24 months, is qualifying involvement.

#### 3E. Slack corroboration (required)

Search the configured Slack workspace for real engagement Org62 missed:
- Account name and deal-channel activity
- Contact names mentioned in call notes, meeting recaps, deal channels, DMs
- Recent (within 24 months) two-way references — "spoke with", "met", "call with [name]", "[name] replied"

Use the most recent credible Slack signal as a qualifying event **with a date**. This is how actively-engaged contacts stay correctly classified when Org62 is stale.

### Step 4: Apply the Consent Test

For each contact, take the **most recent qualifying event date** across 3A–3E:

- **Excluded (opt-out)** — `DoNotCall`/`HasOptedOutOfEmail`/recorded unsubscribe → never contact. (Set aside in Step 2.)
- **Active consent** — most recent qualifying event is **within 24 months**.
- **Lapsed** — a qualifying event exists but the most recent is **older than 24 months** → re-permission needed.
- **No consent basis** — no qualifying event ever found → not contactable without a fresh opt-in.

### Step 5: Map to Buyer Role

For every contact with **active consent**, classify by title/role into a buyer group (reuse the title→buyer-group mapping from `se-buyer-group-mapper`). This confirms they map to a *real* buyer role and tells the AE why the contact matters.

### Step 6: Classify & Rank Key Contacts

A contact is a **Key Contact** when BOTH:
1. **Active consent** (qualifying event within 24 months, not opted out), AND
2. Maps to a real buyer role.

Rank Key Contacts by: seniority of buyer role → recency of engagement → opportunity involvement. Everything with lapsed consent goes to the **Re-permission Needed** list; opt-outs go to the **Excluded** list.

### Step 7: Stamp the "Key Contact" Field (write-back)

Persist the result by setting the Contact field **`Key Contact` = true** on every qualifying Key Contact. This is a **record mutation** — always show the user the list of contacts about to be flagged and **get explicit confirmation before writing.** Only the contacts in the ✅ Key Contacts list are flagged; never flag lapsed, opt-out, or no-basis contacts.

Field API name comes from config (`contact_key_contact_field`, default `Key_Contact__c`); confirm it exists in the target org before writing.

**The write path depends on how this SE is connected to Org62:**

**(a) Connected via the Salesforce CLI** (`sf` available and pointed at `org_alias`) → write directly. Per contact:

```
sf data update record \
  --sobject Contact \
  --record-id <contact_id> \
  --values "<contact_key_contact_field>=true" \
  --target-org <org_alias>
```

For many contacts, build a CSV of `Id` + the field and bulk-update:

```
sf data upsert bulk \
  --sobject Contact \
  --file key_contacts.csv \
  --external-id Id \
  --target-org <org_alias>
```

Report how many records were updated and surface any row-level failures.

**(b) Connected via MCP only** (read-only Org62 MCP — cannot write) → **do not attempt a direct write.** Instead, create a **Slack Canvas** containing the flag instructions for Slackbot to execute. The canvas must list each Key Contact with its **Contact Id, Name, and Account**, and a clear instruction block, e.g.:

> **Slackbot — set `Key Contact = true` on the following Contact records in Org62:**
> - `003xxxxxxxxxxxxxxx` — [Name], [Account]
> - …

Hand the canvas URL to the user (full `https://…/docs/<team_id>/<id>` URL, never a bare ID). The SE (or Slackbot) applies the updates from the canvas.

If the connection method is ambiguous, ask the user which path to use rather than guessing.

## Output Format

Produce one section per AE (and per account within the AE). Output to a Slack Canvas if the user requests it or if run as part of a broader review.

### [AE Name] — Key Contact Review

**Reviewed by:** [se_name] | **Date:** [today] | **Consent basis:** CASL/PIPEDA implied consent, 24-month rolling window

#### ✅ Key Contacts (contactable now)

| Rank | Contact | Title | Account | Buyer Role | Last Qualifying Event | Source | Consent Expires |
|---|---|---|---|---|---|---|---|
| 1 | [Name] | [Title] | [Account] | [role] | [event + date] | Org62 / Slack | [event date + 24mo] |

#### ⚠️ Re-permission Needed (consent lapsed — do NOT cold-contact)

| Contact | Title | Account | Last Qualifying Event | Lapsed Since | Suggested Re-permission Path |
|---|---|---|---|---|---|
| [Name] | [Title] | [Account] | [event + date] | [date consent expired] | [warm intro / event invite / existing-relationship referral] |

#### 🚫 Excluded (opt-out on record)

| Contact | Account | Reason |
|---|---|---|
| [Name] | [Account] | DoNotCall / Opted out of email |

### Territory Summary

- **Key Contacts (contactable):** [N] across [M] accounts
- **Re-permission needed:** [N]
- **Opt-out / excluded:** [N]
- **Accounts with zero contactable key contacts:** [list — these are relationship-risk accounts]

## Notes & Caveats

- **This is a compliance-aware sales tool, not legal advice.** It applies the standard 24-month implied-consent rule; edge cases (express consent on file, B2B exemptions, non-Canadian contacts) should be confirmed with the deal team where relevant.
- **LVMs never qualify.** Voicemail is one-way. Only two-way contact establishes implied consent.
- **Trust Slack over stale Org62.** If Slack shows a recent real conversation the CRM missed, that recent date governs consent — log it and reflect it in the output.
- **Re-permission ≠ delete.** Lapsed contacts are still relationships; the right move is a warm re-permission path (event invite, mutual intro), not a cold blast.
- Reuse `se-buyer-group-mapper` for the title→buyer-role mapping and to sanity-check whether the engaged contacts cover the full buyer group or leave whitespace.
- All org/roster/workspace identifiers come from `~/.claude/se-config.json` — this skill is portable across SEs by design.
