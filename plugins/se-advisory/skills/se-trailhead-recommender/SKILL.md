---
name: se-trailhead-recommender
description: "Recommends Trailhead learning paths matched to a customer account's needs. Analyzes account folder content, Slack conversations, and Org62 activity history to identify relevant products and personas, then searches Trailhead to build curated recommendations. Outputs a Slack Canvas organized by persona and product with direct Trailhead links."
metadata:
  type: sales-operations
  version: "1.1"
  depends_on: "mcp__trailhead__content_search, mcp__trailhead__fetch_content, mcp__slack__slack_create_canvas, mcp__slack__slack_search_public_and_private, mcp__salesforce-org62__run_soql_query"
---

# SE Trailhead Recommender

You recommend Trailhead learning content matched to what a specific customer account needs — based on their products, challenges, personas, and engagement history.

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `account_root` | Local root folder for account workspaces. This skill reads `<config.account_root>/<slug>/`. **Do not hardcode a local path in this skill.** |
| `org_alias` | If Org62 queries need a specific target org, resolve the alias from config rather than hardcoding a `--target-org` value. |

## When to Use

- "Recommend Trailheads for [Account]"
- "What should [Account] be learning on Trailhead?"
- "Build a learning plan for [Account]"
- "Trailhead recommendations for [Account]'s team"
- Any request to match Trailhead content to a customer's situation

## Invocation — Ask These Questions

When invoked, ask the user:

1. **Which account?** — List available accounts from `<config.account_root>/` if the user doesn't specify.
2. **Persona approach** — Ask: "Should I use Trailhead's built-in roles (Administrator, Developer, Architect, etc.) or custom personas based on this account's stakeholders?" Options:
   - Trailhead standard roles
   - Custom personas (user will describe them)
   - Hybrid — identify actual stakeholders and map to closest Trailhead roles

## Data Gathering Flow

```
START: Account identified + persona approach chosen
    │
    ▼
┌─────────────────────────────────────────────┐
│  PHASE 1: Account Intelligence (parallel)    │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Account  │  │ Slack    │  │ Org62    │  │
│  │ Folder   │  │ Search   │  │ Activity │  │
│  │ Read     │  │          │  │ History  │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
│       └──────────────┴──────────────┘        │
└───────────────────────┬─────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────┐
│  PHASE 2: Analysis                           │
│                                              │
│  • Identify products in play / owned         │
│  • Identify personas (per chosen approach)   │
│  • Surface pain points and use cases         │
│  • Note skill gaps and learning needs        │
└───────────────────────┬─────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────┐
│  PHASE 3: Trailhead Search (parallel)        │
│                                              │
│  For each [persona × product] combination:   │
│  • content_search with targeted keywords     │
│  • Filter by level appropriate to persona    │
│  • fetch_content for top matches to verify   │
│    relevance and get full details            │
└───────────────────────┬─────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────┐
│  PHASE 4: Publish                            │
│                                              │
│  • Create Slack Canvas with recommendations  │
│  • Organized by persona, then by product     │
│  • Each recommendation includes Trailhead    │
│    URL for easy trailmix assembly            │
└─────────────────────────────────────────────┘
```

## Phase 1: Account Intelligence

### Account Folder

Read content from the account folder at `<config.account_root>/<account-slug>/` — contracts, decoder output, research, opportunities, notes, deliverables.

Read all available files to understand:
- What Salesforce products the customer owns or is evaluating
- Key challenges and use cases discussed
- Stakeholder names and roles mentioned
- Technical maturity signals

### Slack Search

Search Slack for the account name (and common variations/abbreviations):
- Look for deal-related channels (e.g., #zc-accountname)
- Search public and private messages for account mentions
- Focus on: product discussions, technical challenges, training requests, stakeholder conversations

### Org62 Activity History

Pull comprehensive activity data:

```soql
-- Tasks and Events (activity history)
SELECT Subject, Description, ActivityDate, WhoId, Who.Name, Who.Title, Type, Status
FROM Task
WHERE Account.Name LIKE '%[account]%'
ORDER BY ActivityDate DESC
LIMIT 50

SELECT Subject, Description, StartDateTime, WhoId, Who.Name, Who.Title, Type
FROM Event
WHERE Account.Name LIKE '%[account]%'
ORDER BY StartDateTime DESC
LIMIT 50

-- Opportunities and products
SELECT Name, StageName, Amount, CloseDate,
  (SELECT PricebookEntry.Product2.Name, PricebookEntry.Product2.Family FROM OpportunityLineItems)
FROM Opportunity
WHERE Account.Name LIKE '%[account]%' AND IsClosed = false

-- Cases
SELECT Subject, Description, Status, Priority, CreatedDate, Type
FROM Case
WHERE Account.Name LIKE '%[account]%'
ORDER BY CreatedDate DESC
LIMIT 30

-- Contracts
SELECT ContractNumber, StartDate, EndDate, Status,
  (SELECT PricebookEntry.Product2.Name, Quantity FROM ContractLineItems)
FROM Contract
WHERE Account.Name LIKE '%[account]%' AND Status = 'Activated'
```

## Phase 2: Analysis

From all gathered data, build a matrix of:

| Persona | Products Relevant | Skill Level | Key Needs / Pain Points |
|---------|-------------------|-------------|-------------------------|
| [role or name] | [products] | [Foundational/Intermediate/Advanced] | [what they need to learn and why] |

**Persona identification rules:**
- If using Trailhead roles: map account stakeholders to Administrator, Developer, Architect, Business Analyst, Sales Professional, etc.
- If using custom personas: use names/titles from Org62 contacts and Slack mentions
- If hybrid: identify real people, then tag each with their closest Trailhead role

**Product identification rules:**
- Products owned → recommend advanced/optimization content
- Products in active opportunity → recommend foundational/getting-started content
- Products adjacent to their needs (whitespace) → recommend awareness/benefits content

## Phase 3: Trailhead Search

For each persona × product cell in the matrix:

1. **Search** using `content_search`:
   - Use focused keywords (product name + use case, e.g., "agentforce service", "data cloud segmentation")
   - Apply filters: `roles` (if using standard roles), `levels` (match persona skill level), `types` (prefer TRAIL and MODULE for depth, PROJECT for hands-on)
   - Pull 6-12 results per search

2. **Verify relevance** using `fetch_content`:
   - For the top 2-3 candidates per cell, fetch full content
   - Confirm the module actually covers what the customer needs (not just keyword match)
   - Check `pointTotal` and `minuteTotal` for time commitment

3. **Rank** recommendations:
   - Priority 1: Directly addresses a stated pain point or active use case
   - Priority 2: Covers a product they're evaluating (helps them self-educate pre-demo)
   - Priority 3: Builds foundational knowledge for products they own but underuse
   - Priority 4: Awareness content for whitespace opportunities

## Phase 4: Output — Slack Canvas

Create a Slack canvas titled: **Trailhead Recommendations: [Account] | [Mon DD, YYYY]**

### Canvas Template:

```markdown
# Trailhead Recommendations: [Account] | [Mon DD, YYYY]

### *This canvas was generated using AI, which can produce inaccurate or harmful responses. Review for accuracy and safety before using.*

**Account:** [Name] | **Industry:** [X]
**Products Owned:** [list]
**Products In Play:** [list]
**Persona Approach:** [Standard Roles / Custom / Hybrid]

---

## How to Use This

These recommendations are organized by persona and product. Each entry includes a direct Trailhead link — use these to assemble trailmixes for customer stakeholders or to guide enablement conversations.

---

## :mortar_board: [Persona 1: e.g., "Administrators" or "[Stakeholder Name] (IT Manager)"]

**Role context:** [1 sentence: why this persona matters and what they need]
**Trailhead role filter:** [Administrator / Developer / etc., if applicable]

### [Product A: e.g., "Agentforce"]

| # | Content | Type | Level | Time | Why This One |
|---|---------|------|-------|------|--------------|
| 1 | [Title](https://trailhead.salesforce.com/...) | Module | Intermediate | 45 min | [1-line reason tied to their specific need] |
| 2 | [Title](https://trailhead.salesforce.com/...) | Trail | Foundational | 2 hrs | [reason] |
| 3 | [Title](https://trailhead.salesforce.com/...) | Project | Intermediate | 1 hr | [reason] |

### [Product B: e.g., "Data Cloud"]

| # | Content | Type | Level | Time | Why This One |
|---|---------|------|-------|------|--------------|
| 1 | [Title](https://trailhead.salesforce.com/...) | Module | Foundational | 30 min | [reason] |
| 2 | [Title](https://trailhead.salesforce.com/...) | Trail | Foundational | 3 hrs | [reason] |

---

## :mortar_board: [Persona 2: e.g., "Developers" or "[Stakeholder Name] (Integration Lead)"]

**Role context:** [1 sentence]
**Trailhead role filter:** [Developer]

### [Product A]

| # | Content | Type | Level | Time | Why This One |
|---|---------|------|-------|------|--------------|
| 1 | [Title](https://trailhead.salesforce.com/...) | Module | Advanced | 1 hr | [reason] |

---

## :rocket: Quick Wins — Start Here

The top 3 recommendations across all personas that deliver immediate value:

| # | For Whom | Content | Link | Why Start Here |
|---|----------|---------|------|----------------|
| 1 | [persona] | [title] | [URL] | [reason — e.g., "directly addresses their #1 pain point"] |
| 2 | [persona] | [title] | [URL] | [reason] |
| 3 | [persona] | [title] | [URL] | [reason] |

---

## :jigsaw: Suggested Trailmix Groupings

Based on the recommendations above, here are natural groupings for trailmixes you could create:

| Trailmix Name (Suggested) | Target Audience | Contents | Total Time |
|----------------------------|-----------------|----------|------------|
| "[Account] Admin Foundations" | Admins new to [product] | [list of titles from above] | ~X hrs |
| "[Account] Developer Deep Dive" | Dev team | [list] | ~X hrs |
| "[Account] Executive Overview" | Business leaders | [list] | ~X hrs |

---

## :file_folder: Sources & Signals

| Signal | Source | What It Told Us |
|--------|--------|-----------------|
| [e.g., "3 cases about integration failures"] | Org62 Cases | Need for MuleSoft/integration training |
| [e.g., "Agentforce POC in flight"] | Opportunity | Team needs Agentforce hands-on content |
| [e.g., "Admin asked about flows in Slack"] | Slack | Flow Builder skills gap |

*Recommendations generated [today's date]. Re-run after major account changes.*
```

## Rules

1. **Every recommendation must include a clickable Trailhead URL.** This is non-negotiable — the user will manually build trailmixes from these links.
2. **Customer-accessible content ONLY.** Exclude any content that requires Salesforce employee access, internal org credentials, or partner-only permissions. The audience is the CUSTOMER — they must be able to access every recommended module on trailhead.salesforce.com with a free Trailhead account. Signals to exclude:
   - Modules with "Internal" or "Employee" in the title/description
   - Content flagged for specific Salesforce internal audiences
   - Partner-gated content that requires partner community access
   - Content that references internal tools, internal orgs, or Salesforce employee workflows
   When in doubt, check the module description via `fetch_content` — if it mentions "Salesforce employees" or "internal use," exclude it.
3. **Relevance over volume.** 3-5 highly relevant modules per persona×product cell is better than 10 tangential ones. Quality > quantity.
4. **Explain WHY for each recommendation.** "Because they're evaluating Agentforce and their admin asked about setup in Slack" — not just "covers Agentforce."
5. **Level-match to the persona.** Don't recommend Foundational content to an experienced architect. Don't recommend Advanced content to someone just exploring.
6. **Products owned = optimize/advance content.** Products in play = foundations/getting started. Whitespace = awareness/benefits.
7. **Always include the Quick Wins section.** Give the user an immediate answer to "if they only do 3 things, what should they be?"
8. **Include the Trailmix Groupings section.** Since the user manually assembles trailmixes, suggest logical bundles.
9. **Fetch before recommending.** Don't just rely on search result titles — use `fetch_content` on your top picks to confirm the module actually teaches what you think it does.
10. **Don't echo Trailhead content verbatim.** Use the full markdown to verify relevance and write the "Why This One" column, but the canvas shows titles + links + your reasoning, not Trailhead's instructional text.

## Conversation Starters

- "Recommend Trailheads for [Account]"
- "What should [Account]'s team be learning?"
- "Build a learning plan for [Account]'s admins"
- "Trailhead recs for [Account] — they're evaluating Agentforce and Data Cloud"
- "What Trailheads should I point [Account]'s dev team to?"
