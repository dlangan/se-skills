---
name: df27-account-onboard
description: "Coordinator skill that onboards a new account into the DF27 planning workflow end-to-end. Runs df27-account-profile for the account, then pauses to collect attendee names, then runs df27-attendee-profile for each contact in parallel, then runs df27-recommendation-doc for each profile in parallel. Produces a summary of all files created and any actions needed."
metadata:
  type: event-planning
  version: "1.1"
  origin: "Built 2026-08-06"
  depends_on: "df27-account-profile, df27-attendee-profile, df27-recommendation-doc"
---

# DF27 Account Onboard Coordinator

You onboard a new account into the Dreamforce planning workflow in one guided session. You run the three downstream skills in sequence with a single user pause between Phase 1 and Phase 2.

## Configuration

This skill is a pure orchestrator — it does not read `~/.claude/se-config.json` directly. Each sub-skill it calls (`df27-account-profile`, `df27-attendee-profile`, `df27-recommendation-doc`) resolves its own config keys (`org_alias`, `ae_roster` / `ae_roster_path`, `se_name`) at execution time. If a sub-skill reports a missing config key, surface it to the user rather than hardcoding a substitute.

**Not yet a canonical config key (placeholder only):**
- The Dreamforce planning workspace root (used below as `<dreamforce_root>`, e.g. `~/dreamforce-2027/`) has no dedicated config key today. Use your own path until one is added — do not invent a key name in the skill itself.

## When to Use

- "Onboard [Account] for DF27"
- "Set up DF for [Account]"
- "Build the full DF package for [Account]"
- "New DF registrations from [Account] — set them up"
- Any time a new account has one or more DF registrants and nothing has been built yet

---

## Required Inputs

Confirm before starting:
- **Account name** (required)
- **Account ID** (required — from registrations file or user)
- **AE name** (required — or derive from `config.ae_roster` / `config.ae_roster_path`)
- **Attendee list** — do NOT ask for this upfront. Collect it after Phase 1.

If Account ID is not provided, ask for it. Do not proceed without it.

---

## Phase 1: Account Profile

**Run `df27-account-profile` for the account.**

Follow that skill's full workflow:
1. Check Slack ZC channel for existing canvases
2. Run Org62 queries (contracts, won line items, open opps) in parallel
3. Run entitlement decoder if no product map canvas exists
4. Write `accounts/{slug}/account.md`

Report to the user when complete:
```
✅ Account profile built: accounts/{slug}/account.md

Summary:
- Footprint: [e.g. "Sales Cloud 150 users, FSL 80 techs — Tableau pending"]
- Pipeline: [e.g. "$420K open across 3 opps"]
- Contract risk: [e.g. "⚠️ Service Cloud renewal due Oct 2026" or "None"]
- Why DF this year: [1-sentence from the account.md "Why DF" section]
```

**Then pause and ask:**

> "Account profile is ready. Who's attending Dreamforce from [Account]? Give me their names and titles (I'll look up the rest from Org62). You can list as many as you have."

Wait for the user's response before proceeding.

---

## Phase 2: Attendee Profiles

**Accept the attendee list from the user.** It may come in any format:
- "[Attendee 1] — [Title], [Attendee 2] — [Title]"
- A bulleted list
- Just names with no titles

For each contact on the list:
1. Search Org62 for the contact record: `SELECT Id, Name, Title FROM Contact WHERE AccountId = '[AccountId]' AND Name LIKE '%[Name]%' LIMIT 5`
2. Confirm the match (if multiple results, pick the closest title match or ask the user)
3. Run **`df27-attendee-profile`** for each contact

**Run all attendee profile queries and writes in parallel** — single message with all tool calls where there are no dependencies.

Report when Phase 2 is complete:
```
✅ [N] attendee profiles built:
- accounts/{slug}/{first-last}.md (Archetype: [Archetype])
- accounts/{slug}/{first-last}.md (Archetype: [Archetype])
- ...
```

Then immediately proceed to Phase 3 without pausing.

---

## Phase 3: Recommendation Docs

**Run `df27-recommendation-doc` for each profile built in Phase 2.**

Run all recommendation docs in parallel — single message with all tool calls where there are no dependencies.

Each doc:
- Reads `account.md` + the attendee's `.md` file
- Classifies archetype
- Derives themes
- Selects sessions
- Creates and formats the Google Doc

Report when Phase 3 is complete:
```
✅ [N] recommendation docs created:

- [Attendee Name] — [Google Doc URL]
  Archetype: [Archetype]
  Themes: [Theme 1] / [Theme 2] / [Theme 3]

- [Attendee Name] — [Google Doc URL]
  Archetype: [Archetype]
  Themes: [Theme 1] / [Theme 2] / [Theme 3] / [Theme 4]
```

---

## Final Summary

After Phase 3, output a single completion summary:

```
## DF27 Onboard Complete — [Account Name]

**Account profile:** accounts/{slug}/account.md
**Attendee profiles:** [N] built
**Recommendation docs:** [N] created

### Files created this session:
- accounts/{slug}/account.md
- accounts/{slug}/{first-last}.md  ← [Name], [Archetype]
- accounts/{slug}/{first-last}.md  ← [Name], [Archetype]
- [Google Doc URLs per attendee]

### Actions needed:
- [ ] [Any contract expiry flags from account profile]
- [ ] [Any gaps — e.g. "Economic buyer [Name] is NOT attending DF — flag to AE"]
- [ ] Share docs with AE: [AE Name]
```

If there are no actions needed, say so explicitly — don't leave the section blank.

---

## Error Handling

- **Contact not found in Org62:** Note the gap, write the profile from whatever the user provided (name + title), mark as "Contact ID: not found — manual entry." Proceed.
- **Account profile already exists:** Confirm with user — "An account.md already exists for [Account]. Refresh it or skip to attendee profiles?" Do not silently overwrite.
- **Attendee profile already exists:** Skip and note it. Do not rebuild unless the user explicitly asks.
- **Recommendation doc already exists (Google Doc):** Note it and link to the existing doc. Do not create a duplicate.

---

## Key Conventions

- **Phase 1 always runs first.** Never build attendee profiles before `account.md` exists — the recommendation doc skill depends on it.
- **One pause, between Phase 1 and Phase 2.** No other pauses. Don't ask for confirmation before Phase 3.
- **Parallelism within phases.** All attendee profiles run in parallel. All recommendation docs run in parallel.
- **AE names always from config:** `config.ae_roster` / `config.ae_roster_path` — never hardcode or recall from memory.
- **Org62 queries always require:** `usernameOrAlias: config.org_alias` and the SE's home `directory`
- **Account slug derivation:** lowercase, hyphenated short name. When in doubt, check existing folders under `accounts/`.

---

## Conversation Starters

- "Onboard [Account] for DF27"
- "Build the full DF package for [Account] — they have 3 new registrants"
- "Set up DF27 for [Account] — Account ID [ID]"
- "New DF registrations from [Account]: [Name 1], [Name 2]"
