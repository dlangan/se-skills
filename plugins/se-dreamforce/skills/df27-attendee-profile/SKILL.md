---
name: df27-attendee-profile
description: "Build a Dreamforce attendee profile for a registered contact. Queries Org62 for contact details, open opportunities, and account context; searches Slack ZC channel and account channel for recent signals; reads relevant account research canvases; and writes a structured markdown profile to the Dreamforce planning folder. Also updates INDEX.md with the account folder reference."
metadata:
  type: event-planning
  version: "1.1"
  origin: "Built 2026-07-09 from DF26 registration planning session"
  depends_on: "mcp__salesforce-org62__run_soql_query, mcp__slack__slack_search_public_and_private, mcp__slack__slack_read_channel, mcp__slack__slack_read_canvas, mcp__slack__slack_search_channels"
---

# DF27 Attendee Profile Builder

You are a Dreamforce planning assistant. When asked to build an attendee profile, you research a registered contact using Org62, Slack, and any available account canvases, then write a structured profile file to the Dreamforce planning folder.

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `org_alias` | Org62 target org / username — supplies the `usernameOrAlias` parameter for `mcp__salesforce-org62__run_soql_query` calls. **Never hardcode a username or email in this skill.** |
| `ae_roster_path` | Path to the SE's AE-alignment file — used to resolve which AE owns the account, never from memory. |

**Not yet a canonical config key (placeholder only):**
- The Dreamforce planning workspace root (used below as `<dreamforce_root>`, e.g. `~/dreamforce-2027/`) has no dedicated config key today. Use your own path until one is added — do not invent a key name in the skill itself.
- The `directory` parameter required by the Org62 SOQL MCP tool is the SE's home directory. Resolve it at runtime rather than hardcoding a path.

## When to Use

- "Build a DF profile for [Name]"
- "Add [Name] to the DF27 folder"
- "Profile [Name] from [Account] — they just registered for Dreamforce"
- Any request to build or refresh a Dreamforce attendee profile

## Required Inputs

Before starting, confirm you have:
- **Contact name** (and ideally Contact ID from the registrations snapshot)
- **Account name** and **AE name**

If not provided, check `<dreamforce_root>/registrations/` for the latest snapshot file.

## Data Sources — Always Pull in This Order

### 1. Org62 — Contact record
```soql
SELECT Id, Name, Title, Email, Phone, MobilePhone, LastActivityDate,
  Account.Name, Account.Owner.Name, Account.Industry
FROM Contact
WHERE Id = '[ContactId]'
```

### 2. Org62 — Open opportunities on the account
```soql
SELECT Id, Name, StageName, Amount, CloseDate, Type, Owner.Name,
  (SELECT Contact.Name, Contact.Title FROM OpportunityContactRoles WHERE IsPrimary = true LIMIT 1)
FROM Opportunity
WHERE Account.Name LIKE '%[AccountName]%'
AND IsClosed = false
ORDER BY CloseDate ASC
```

### 3. Org62 — Other contacts on the account (for org chart context)
```soql
SELECT Id, Name, Title, Email, Phone, MobilePhone, LastActivityDate
FROM Contact
WHERE AccountId = '[AccountId]'
AND Id != '[ContactId]'
ORDER BY LastActivityDate DESC NULLS LAST
LIMIT 10
```

### 4. Slack — ZC channel for the account
Use `mcp__slack__slack_search_channels` with the account name to find the ZC channel (format: `#ZC:...:AccountName`), then read recent messages with `mcp__slack__slack_read_channel`.

### 5. Slack — Keyword search for the contact and account
Run `mcp__slack__slack_search_public_and_private` for:
- Contact full name + account name
- Account name alone (for recent deal signals)

### 6. Slack canvases — Read any account research, product map, or call notes canvases surfaced in the ZC channel (file IDs ending in the canvas format). Use `mcp__slack__slack_read_canvas` with both `channel_id` and `canvas_id`.

### 7. AE roster — ALWAYS read the AE mapping from `config.ae_roster_path`, never from memory.

## Profile File Format

Write to: `<dreamforce_root>/accounts/{account-slug}/{first-last}.md`

Account slug: lowercase, hyphenated, short (e.g., `<account-slug>` — check existing folders under `accounts/` for the convention already in use).

Use this template:

```markdown
# [Full Name] — [Title]
**Account:** [Account Name]
**AE:** [AE Name]
**DF Status:** Registered ([Registration Date]) — in person

---

## Contact

| Field | Value |
|---|---|
| Email | ... |
| Phone | ... |
| Mobile | ... |
| Contact ID | ... |
| Last Activity | YYYY-MM-DD |

---

## Role & Context

[2-4 paragraphs. Cover:]
- What this person does and why they matter to the account
- How they relate to other key contacts (org chart context)
- What Salesforce products/deals are most relevant to their role
- Key signals from Slack, call notes, or account canvases
- Any recent engagement or notable absence of engagement

---

## Open Opportunities

| Opportunity | Stage | Amount | Close Date | Owner | Primary Contact |
|---|---|---|---|---|---|
| ... | ... | ... | ... | ... | ... |

---

## Key Themes for DF

[Numbered list of 3-5 themes, each with 1-2 sentences of rationale. Specific to this person's role — not generic account themes.]

---

## Suggested DF Sessions

| Session | Why |
|---|---|
| [Session Title] | [1-sentence rationale specific to this person] |
| ... | ... |

---

## SE Talking Points

[4-6 bulleted talking points. Role-specific. Actionable. Include:]
- The opener framed in their daily pain
- The "you already own this" angle (if applicable)
- Any specific deals, risks, or next steps this person can influence
- How their role connects to the AE's priority deals
```

## After Writing the Profile

1. Update `<dreamforce_root>/INDEX.md` — set the account's Recommendation File column to `accounts/{account-slug}/` if not already set
2. Confirm the file path to the user

## Key Conventions

- **Registration date → in person:** All DF campaign registrants are assumed to be attending in person
- **AE names:** Always read from `config.ae_roster_path`, never from memory
- **Org62 queries:** Always include `usernameOrAlias: config.org_alias` and the SE's home `directory` on every call
- **Slack canvases:** `mcp__slack__slack_read_canvas` requires both `channel_id` and `canvas_id` as separate parameters (not `file_id`)
- **Account research canvases:** If a canvas is found in the ZC channel, always read it — it contains stakeholder maps, deal health, and strategic context that enriches the profile significantly
- **"Notes by Gemini" documents:** Always use the full transcript tab (2nd tab), not the summary

## Parallelism

When building multiple profiles for the same account, run all data collection in parallel (single message with multiple tool calls), then write all profile files in parallel. For example, a batch of 5 profiles for one account and 2 for another built in a single session should run as two parallel groups.

When building profiles for different accounts, still run Org62 queries and Slack searches in parallel.

## Archetype Classification

After collecting contact data but before writing the profile, classify the contact using `<dreamforce_root>/roles/attendee-archetypes.md`.

**Steps:**
1. Match the contact's title against the title patterns in the archetypes file
2. Assign a **primary archetype** (one of: Executive Buyer, Technical Champion, Process Owner, Sales Leader, Builder, Marketer)
3. Assign a **secondary archetype** if the title or context warrants it (see dual-archetype blending rules)
4. Add the classification to the profile file header, e.g.:
   `**Archetype:** Builder`  or  `**Archetype:** Executive Buyer (secondary: Marketer)`

The archetype determines the framing angle, session type emphasis, and tone of the Key Themes and SE Talking Points sections.

## Profile Differentiation

Each profile must be differentiated by role — don't repeat the same account context in every profile. Use the archetype's framing angle as the primary lens:

| Archetype | Primary Angle |
|---|---|
| Executive Buyer | Business outcomes, peer validation, renewal risk, executive relationships |
| Technical Champion | Architecture decisions, entitlement activation, integration patterns |
| Process Owner | Workflow pain, automation opportunity, "what done looks like" |
| Builder | What they'll configure, HOT sessions, "you already own this, turn it on" |
| Sales Leader | Rep productivity, adoption proof, Slack as sales tool |
| Marketer | Campaign outcomes, MCAE/Data 360, pipeline from marketing |

## Reference Files

- Registrations snapshot: `<dreamforce_root>/registrations/`
- Session catalog: `<dreamforce_root>/sessions/`
- Master index: `<dreamforce_root>/INDEX.md`
- AE roster: `config.ae_roster_path`
- Archetype definitions: `<dreamforce_root>/roles/attendee-archetypes.md`

## Example Profile Log Entry (illustrative)

| Contact | Account | AE | File |
|---|---|---|---|
| [Contact Name] | [Account Name] | [AE Name] | `accounts/<account-slug>/<first-last>.md` |

This is a running log of profiles built to date — keep entries generic in this shared skill file; do not record real attendee, account, or AE names here. Track the real, per-run history in your own Dreamforce planning workspace instead.
