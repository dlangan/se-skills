---
name: df-update
description: "SE Dreamforce Session Monitor. Checks the Dreamforce FY27 breakout campaign containers in Org62 for newly populated sessions and writes the results to a JSON data file for consumption by the SE Event Recommender skill."
metadata:
  type: event-planning
  version: "1.0"
  depends_on: "mcp__salesforce-org62__run_soql_query, se-event-recommender"
---

# SE Dreamforce Session Monitor

Checks the Dreamforce FY27 breakout campaign containers in Org62 for newly populated sessions and writes the results to a JSON data file for consumption by the SE Event Recommender skill.

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `org_alias` | Org62 target org / username — supplies the `usernameOrAlias` parameter for `mcp__salesforce-org62__run_soql_query` calls. **Never hardcode a username or email in this skill.** |

**Not yet a canonical config key (placeholder only):**
- The output JSON file path (used below as `<df_sessions_data_file>`, e.g. `~/.claude/data/dreamforce-fy27-sessions.json`) has no dedicated config key today. Use your own path until one is added — do not invent a key name in the skill itself.
- The `directory` parameter required by the Org62 SOQL MCP tool is the SE's home directory. Resolve it at runtime rather than hardcoding a path.

The Campaign IDs below are org-instance identifiers (Dreamforce FY27 campaign hierarchy in this org) — they are data, not PII, and are kept as-is because the skill can't function without them.

## When to Use

- User says "check dreamforce sessions"
- User says "update dreamforce breakouts"
- User says "dreamforce session monitor"
- User says "any new DF sessions?"

## Campaign IDs

- **Grandparent:** `701ed00000joSDkAAM` (WW--PGM--Dreamforce Grandparent for FY27--FY27Q3)
- **San Francisco Breakout:** `701ed00000zhh3NAAQ`
- **Virtual Breakout:** `701ed00000zhaOUAAY`
- **On Demand Breakout:** `701ed00000zglqcAAA`

## Workflow

### Phase 1: Query Breakout Containers

Run SOQL against Org62 (`usernameOrAlias: config.org_alias`, `directory`: the SE's home directory) to find child campaigns under each breakout container:

```soql
SELECT Id, Name, Status, StartDate, EndDate, Type, NumberOfContacts, NumberOfLeads, Description, ParentId
FROM Campaign
WHERE ParentId IN ('701ed00000zhh3NAAQ', '701ed00000zhaOUAAY', '701ed00000zglqcAAA')
ORDER BY StartDate, Name
```

### Phase 2: Also Check for Booth/Expo/Keynote/Theater Siblings

Search for any new campaign types that may have appeared under the grandparent:

```soql
SELECT Id, Name, Status, StartDate, ParentId, NumberOfContacts, NumberOfLeads
FROM Campaign
WHERE ParentId = '701ed00000joSDkAAM'
  AND (Name LIKE '%Booth%' OR Name LIKE '%Expo%' OR Name LIKE '%Keynote%'
       OR Name LIKE '%Theater%' OR Name LIKE '%Demo%' OR Name LIKE '%Lab%'
       OR Name LIKE '%Summit%' OR Name LIKE '%Workshop%')
ORDER BY Name
```

### Phase 3: Write JSON Output

Write results to: `<df_sessions_data_file>` (e.g. `~/.claude/data/dreamforce-fy27-sessions.json`)

Create the parent directory if it doesn't exist.

**JSON Schema:**

```json
{
  "lastChecked": "2026-06-17T10:30:00Z",
  "grandparentCampaignId": "701ed00000joSDkAAM",
  "event": {
    "name": "Dreamforce FY27",
    "startDate": "2026-09-15",
    "location": "San Francisco"
  },
  "containers": {
    "sanFrancisco": {
      "campaignId": "701ed00000zhh3NAAQ",
      "sessionCount": 0,
      "sessions": []
    },
    "virtual": {
      "campaignId": "701ed00000zhaOUAAY",
      "sessionCount": 0,
      "sessions": []
    },
    "onDemand": {
      "campaignId": "701ed00000zglqcAAA",
      "sessionCount": 0,
      "sessions": []
    }
  },
  "otherFormats": [],
  "totalSessions": 0,
  "status": "empty|populating|populated"
}
```

Each session object in the `sessions` arrays:

```json
{
  "id": "701...",
  "name": "Session Title",
  "status": "Confirmed",
  "startDate": "2026-09-15",
  "endDate": null,
  "type": null,
  "contacts": 0,
  "leads": 0,
  "description": null
}
```

**Status logic:**
- `"empty"` — 0 total sessions across all containers
- `"populating"` — 1–20 sessions (partial load in progress)
- `"populated"` — 21+ sessions (catalog likely complete)

### Phase 4: Compare to Previous Run

If the JSON file already exists, read it first and compare:
- Report new sessions added since last check
- Report any sessions removed
- Report changes in member counts (contacts/leads enrolled)

### Phase 5: Report Summary

Output a brief summary to the user:

```
## Dreamforce FY27 Session Monitor

**Last checked:** [timestamp]
**Status:** [empty/populating/populated]

| Container | Sessions | Members |
|-----------|----------|---------|
| San Francisco | X | Y |
| Virtual | X | Y |
| On Demand | X | Y |

[If changes detected:]
### Changes Since Last Check
- Added: [list new session names]
- Removed: [list removed session names]
- Enrollment changes: [list]

[If empty:]
No sessions populated yet. The breakout containers are still empty shells.
```

## Integration with Event Recommender

The SE Event Recommender skill (`se-event-recommender`) should reference `<df_sessions_data_file>` when matching accounts to Dreamforce sessions. When sessions are populated, the recommender can match by:
- Session name keywords → product focus
- Session description → use case alignment
- Session date → customer availability

## Notes

- Run this periodically as Dreamforce approaches (weekly from July, daily from August)
- The containers going from empty to populated is the key signal that the agenda has been finalized
- Once populated, individual session enrollments (contacts/leads) indicate which sessions are filling up
- **Sessions must always come from this org's live catalog verbatim** — never invent a session name when populating downstream skills (df27-recommendation-doc) from this data.
