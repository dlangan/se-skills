---
name: df27-registration-sync
description: "Weekly sync of Dreamforce campaign registrations against the SE's territory. Queries Org62 for all registered contacts owned by territory AEs, diffs against the prior snapshot, writes a new dated snapshot file, and produces a net-new report: new contacts, new accounts, which accounts still need account.md, and which contacts still need a profile and recommendation doc. This is the entry point skill — run it before any other DF27 skill."
metadata:
  type: event-planning
  version: "1.1"
  origin: "Built 2026-07-09"
  depends_on: "mcp__salesforce-org62__run_soql_query"
---

# DF27 Registration Sync

You are the entry point for the Dreamforce planning workflow. Run this skill weekly (or whenever someone says "check DF registrations" or "who's registered for Dreamforce?") to detect new registrations in the SE's territory and trigger the downstream skill chain.

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `org_alias` | Org62 target org / username — supplies the `usernameOrAlias` parameter for `mcp__salesforce-org62__run_soql_query` calls. **Never hardcode a username or email in this skill.** |
| `ae_roster` | List of `{ name, org62_user_id }` for the AEs in the SE's territory — supplies the `Owner.Id IN (...)` filter for the registration query. **Never hardcode AE names or User IDs in this skill.** |

**Not yet a canonical config key (placeholder only):**
- The Dreamforce planning workspace root (used below as `<dreamforce_root>`, e.g. `~/dreamforce-2027/`) has no dedicated config key today. Use your own path until one is added — do not invent a key name in the skill itself.
- The `directory` parameter required by the Org62 SOQL MCP tool is the SE's home directory. Resolve it at runtime rather than hardcoding a path.

If `ae_roster` is empty or missing, ask the user for the AE names/IDs to scope the query, or offer to build the roster now.

## When to Use

- "Sync DF registrations"
- "Check who's registered for Dreamforce"
- "Any new DF registrations this week?"
- "Run the DF registration sync"
- At the start of any DF planning session — always sync before building profiles or docs

---

## Step 1 — Read AE IDs from Config

**ALWAYS** read the AE roster from `config.ae_roster` — never from memory.

Extract the AE User IDs for the territory filter.

---

## Step 2 — Read the Latest Prior Snapshot

List files in `<dreamforce_root>/registrations/` and identify the most recent snapshot by filename date (`YYYY-MM-DD.md`).

Read it to extract the full set of Contact IDs already known. This is the diff baseline.

If no prior snapshot exists, treat all query results as net-new.

---

## Step 3 — Query Org62 for Current Registrations

```soql
SELECT Id, Status, CreatedDate,
  Contact.Id, Contact.Name, Contact.Title, Contact.Email,
  Contact.Phone, Contact.MobilePhone,
  Contact.Account.Id, Contact.Account.Name, Contact.Account.Owner.Name
FROM CampaignMember
WHERE CampaignId = '<df_registration_campaign_id>'
AND Contact.Account.OwnerId IN (
  '<ae_1_org62_user_id>',
  '<ae_2_org62_user_id>',
  '<ae_3_org62_user_id>',
  '<ae_4_org62_user_id>',
  '<ae_5_org62_user_id>'
)
ORDER BY Contact.Account.Owner.Name, Contact.Account.Name, Contact.Name
```

Build the `IN (...)` list dynamically from `config.ae_roster` — do not hardcode User IDs.

Parameters:
- `usernameOrAlias`: `config.org_alias`
- `directory`: the SE's home directory (resolve at runtime)
- Campaign ID: the current Dreamforce Registration campaign (org-instance ID — see "Campaign ID Update" below)

---

## Step 4 — Diff Against Prior Snapshot

Compare Contact IDs from the query against Contact IDs in the prior snapshot.

Produce three lists:
1. **Net-new contacts** — in query results, not in prior snapshot
2. **Net-new accounts** — accounts appearing for the first time (no prior registered contacts)
3. **Unchanged** — already in prior snapshot (no action needed)

---

## Step 5 — Check Downstream Readiness

For each net-new account, check if `account.md` exists:
```
<dreamforce_root>/accounts/{slug}/account.md
```

For each net-new contact, check if their profile exists:
```
<dreamforce_root>/accounts/{slug}/{first-last}.md
```

Build a "work queue" — what needs to be done to bring everything current.

---

## Step 6 — Write New Snapshot File

Write to: `<dreamforce_root>/registrations/YYYY-MM-DD.md`
(Use today's date as the filename.)

Use exactly this format, matching the existing snapshot structure:

```markdown
# DF Registrations — Territory Snapshot
**Date:** YYYY-MM-DD
**Campaign ID:** [current DF registration campaign ID]
**Query method:** CampaignMember WHERE Contact.Account.OwnerId IN (AE User IDs from config.ae_roster)
**Total registered:** [N] contacts across [N] accounts

---

## How to refresh

Run the SOQL below. Any contact not in this file is net-new — add them here and build a recommendation file.

```soql
SELECT Id, Status, CreatedDate,
  Contact.Id, Contact.Name, Contact.Title, Contact.Email,
  Contact.Phone, Contact.MobilePhone,
  Contact.Account.Id, Contact.Account.Name, Contact.Account.Owner.Name
FROM CampaignMember
WHERE CampaignId = '<df_registration_campaign_id>'
AND Contact.Account.OwnerId IN (
  -- one row per config.ae_roster entry, e.g.:
  '<ae_org62_user_id>'  -- <AE name>
)
ORDER BY Contact.Account.Owner.Name, Contact.Account.Name, Contact.Name
```

---

## Registered Contacts

### [AE Name]

**[Account Name]**

| Contact | Title | Email | Phone | Mobile | Contact ID | Campaign Member ID | Registered |
|---|---|---|---|---|---|---|---|
| [Name] | [Title] | [Email] | [Phone] | [Mobile or —] | [ContactId] | [CampaignMemberId] | [YYYY-MM-DD] |

[Repeat per account, grouped under AE]

---

## Accounts Summary

| Account | AE | # Registered |
|---|---|---|
| [Account] | [AE] | [N] |
```

**Notes on format:**
- Group contacts by AE, then by account within each AE section
- Use `—` for missing phone/mobile (not blank)
- Registration date = `CreatedDate` from CampaignMember (the date they registered, not today)
- If a contact has a Campaign Member ID already in the prior snapshot, carry it forward exactly

---

## Step 7 — Update INDEX.md

Update `<dreamforce_root>/INDEX.md`:
- Add a new row to the Registrations table with today's snapshot
- Update the total count
- Update any account rows where the Recommendation File column is still `—`

---

## Step 8 — Produce the Net-New Report

Output a concise report to the user in this format:

```
## DF Registration Sync — [Date]

**Total registered:** [N] contacts, [N] accounts
**Net-new since [prior snapshot date]:** [N] contacts, [N] accounts

### New Contacts
[AE Name] — [Account]:
  • [Contact Name], [Title] (registered [date])

[Repeat per AE/account group]
[If none: "No new contacts since last sync."]

### New Accounts
  • [Account Name] (AE: [AE Name]) — [N] contacts registered

[If none: "No new accounts since last sync."]

### Work Queue
Skills to run to bring everything current:

  [ ] df27-account-profile  → [Account Name] (no account.md yet)
  [ ] df27-attendee-profile → [Contact Name], [Account] (no profile yet)
  [ ] df27-recommendation-doc → [Contact Name], [Account] (profile exists, no doc yet)

[If everything is current: "All registered contacts have profiles and docs. Nothing to build."]
```

---

## Key Conventions

- **AE IDs always from config** — never hardcode or recall from memory; read `config.ae_roster`
- **Snapshot filename is today's date** — `YYYY-MM-DD.md` — even if no changes since last sync (the file is a point-in-time record)
- **Campaign ID is fixed** for a given Dreamforce year — this is an org-instance identifier, not a per-run input. Update it here when a new year's campaign ID is confirmed (see "Campaign ID Update" below).
- **Org62 queries** always require `usernameOrAlias: config.org_alias` and the SE's home `directory`
- **Account slugs** for folder/file checking: derive as lowercase-hyphenated short name (e.g., `acme-manufacturing` → `acme`, `north-star-logistics` → `nsl`). When in doubt, check what folders already exist under `accounts/`.
- **Do not rebuild existing profiles** — the work queue only flags missing files. If a profile already exists, it's not in the queue even if the contact is "net-new" by some other metric.

---

## Campaign ID Update

When a new Dreamforce year's registration campaign opens and a new campaign ID is confirmed:
1. Update the Campaign ID in this skill file
2. Update `INDEX.md` with the new campaign ID
3. Run a fresh sync — the prior snapshot becomes last year's baseline, all new-year registrants are net-new

To find the current year's campaign ID when it's available:
```soql
SELECT Id, Name, StartDate, Status, Type
FROM Campaign
WHERE Name LIKE '%Dreamforce%'
   OR Name LIKE '%DF27%'
ORDER BY CreatedDate DESC
LIMIT 10
```

---

## Full Skill Chain (Order of Operations)

This skill is Layer 1. Always run in this sequence:

```
Layer 0: df-update              (one-time — when the DF27 session catalog drops)
Layer 1: df27-registration-sync (weekly — this skill)
Layer 2: df27-account-profile   (per net-new account — after Layer 1)
Layer 3: df27-attendee-profile  (per net-new contact — after Layer 2)
Layer 4: df27-recommendation-doc (per contact — after Layer 3)
```

Layers 2, 3, and 4 can be parallelized within each layer. Layers must be run in sequence.

---

## Reference Files

- Prior snapshots: `<dreamforce_root>/registrations/`
- Master index: `<dreamforce_root>/INDEX.md`
- AE roster: `config.ae_roster` (`~/.claude/se-config.json`)
- Account folders: `<dreamforce_root>/accounts/`
