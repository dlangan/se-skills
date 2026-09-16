---
name: se-event-recommender
description: "Matches an AE's open pipeline to upcoming events, workshops, and programs, then outputs a personalized event recommendation canvas — including a 'New This Week' webinar sweep, regional event matches, and always-available Trailhead Academy workshops and on-demand webinars."
metadata:
  type: sales-operations
  version: "1.0"
---

# SE Event Recommender

You are a Salesforce Solutions Engineering assistant that matches an AE's open pipeline to upcoming events, workshops, and programs — then outputs a personalized event recommendation canvas.

## When to Use

- User says "recommend events for [AE Name]"
- User says "event recommendations for [AE Name]"
- User says "what events should [AE Name]'s customers attend?"

## Inputs

- **AE Name** (required) — the account executive whose territory to analyze
- **Time Horizon** — from today through the end of the current fiscal year. Always cover the full remaining FY unless user specifies a shorter window.

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `fiscal_calendar` | Holds the current fiscal-year end date (e.g., `fy_end: "2027-01-31"`) used as the "through" boundary for the time horizon and canvas templates below. Never hardcode a specific fiscal year's end date in this skill — read it from config so the skill doesn't go stale when the fiscal year rolls over. |
| `se_name` | Your name, used in the canvas footer attribution. |

If `fiscal_calendar` is missing, ask the user for the current fiscal year end date before computing the time horizon (the quarter structure below is evergreen and doesn't need to be configured). If `se_name` is missing, omit the footer attribution or ask the user for a name to use.

## Salesforce Fiscal Calendar

- FQ1: Feb 1 – Apr 30
- FQ2: May 1 – Jul 31
- FQ3: Aug 1 – Oct 31
- FQ4: Nov 1 – Jan 31

The fiscal year is +1 from calendar year for Nov-Jan (e.g., Nov 2026 = FQ4 FY27).

## Workflow

### Phase 1: Identify the AE & Pull Pipeline

1. Look up the AE's User ID in Org62
2. Query all open opportunities:
   ```
   SELECT Id, Name, Account.Name, Account.Id, StageName, Amount, CloseDate, 
          CurrencyIsoCode, Product_Fit__c, SE_Engagement__c, SE_Comments__c,
          ForecastCategoryName
   FROM Opportunity
   WHERE OwnerId = '<ae_user_id>' AND IsClosed = false
   ORDER BY Amount DESC
   ```
3. Filter out: Webstore, Online Sales, Chapi, Renewal, ARY, Touchless, Courtesy, $0/null Amount
4. Group remaining opps by Account — note total pipeline per account, products involved, and most advanced stage

### Phase 2: Discover Available Events

Search Slack for upcoming events from today through the fiscal-year end date (`config.fiscal_calendar.fy_end`). Search these channels and patterns:

**Channels to search:**
- #marketing-north-amer-webinars (weekly webinar announcements — PRIMARY source for "New This Week" section)
- #ca-marketing-updates (Canada marketing spotlight — weekly event roundups)
- #grb-canada-fy27 (field marketing programs)
- #mm-maefy27-extended-team (team events)
- Regional channels (social-mtl, social-tor, etc.)
- #events-world-tours-amer

*(These channel names are organization-specific and fiscal-year-tagged — no canonical config key covers a channel list today, so they're left as literal examples. Update them for your own workspace/region and fiscal year; flag for a future `event_channels` config key if this list needs to be centralized.)*

**Search queries (run multiple):**
- "workshop registration customer" with date filters for each remaining month
- "event customer registration" with your relevant regional cities
- "Dreamforce"
- "AI Lab" OR "Guardrails" OR "Zero Copy" OR "Agentic Contact Center"
- "hands-on workshop" OR "customer workshop"

**Also check for:**
- Trailhead Academy workshops (virtual) — reference the Workshop Catalog in the Architect Brain (~60 workshops available year-round). Always include the direct Trailhead Academy registration link for each recommended workshop using the format: `https://trailheadacademy.salesforce.com/classes/[code]` (e.g., `https://trailheadacademy.salesforce.com/classes/agt501-agentforce-vision-and-use-cases---agt501`)
- On-demand webinars — search #ca-marketing-updates and regional channels for "on-demand" or "on demand" links. These are always available and can be shared immediately with no scheduling.
- Dreamforce (San Francisco, dates vary by year — check the current year's announcement)
- World Tours (check #events-world-tours-amer)
- Regional executive dinners
- Industry-specific events
- Connections / TrailblazerDX / Tableau Conference
- Salesforce+ content and demo series (e.g., "Agentforce Demo Series")
- **Weekly webinar sweep** — search #marketing-north-amer-webinars (and #ca-marketing-updates) for webinar announcements posted in the **last 7 days**. These are time-sensitive — new webinars drop weekly and have registration deadlines. Match them to open pipeline using the same product-to-event mapping logic. Present matched webinars in a dedicated "New This Week" section at the TOP of the canvas (above the full FY calendar) so the AE sees fresh content first.

**Capture for each event:**
- Event name
- Date
- Location (city)
- Format (full-day, half-day, virtual, dinner, conference)
- Registration link (see extraction rules below)
- Target audience / product focus
- Capacity status (if mentioned)

**Link extraction — mandatory step:**
Search results return message titles, not the links inside them. For every webinar or event found via search, you MUST read the actual message content to extract the registration URL:
1. Use `slack_read_channel` or `slack_read_thread` on the source message to get full content
2. Look for: `https://` URLs, `register`, `registration`, `join`, `bit.ly`, `sfdc.co`, `go.salesforce.com`, `salesforce.com/events`, `trailhead.salesforce.com`, Zoom links, or any non-internal URL in the message body
3. If a customer-accessible link is found, include it verbatim
4. If only an internal link exists (salesforce.enterprise.slack.com, Basecamp, internal Google Docs), mark as "(AE to share — internal registration)" — do NOT leave the cell blank
5. If no link at all is found after reading the message, mark as "(Link not found — check source channel)"
6. Never fabricate or guess a URL

### Phase 3: Match Accounts to Events

For each account with ≥$50K pipeline, apply this matching logic:

**Product-to-Event Mapping:**

| Opp Product Focus | Event Type Match |
|---|---|
| Data Cloud / Tableau / Analytics | Zero Copy Future workshops, Data 360 workshops (SDC series), Data Cloud dinners |
| Agentforce / AI | Agentforce Guardrails (security concerns), AI Lab (broad vision), AGT501-504 Activation workshops |
| Service Cloud / Field Service | Agentic Contact Center, Service Agent Activation (AGT503), ASC series workshops |
| Sales Cloud | Sales Agent Activation (AGT504), SSC series workshops |
| Slack | AI Lab (Slack as AI center), Slack-specific events |
| Marketing Cloud | MOC series workshops, Connections conference |
| MuleSoft | MSA series workshops, Agent Fabric events |
| Revenue Cloud | REV series workshops |
| Commerce | B2C/B2B Commerce workshops, Connections |
| Manufacturing Cloud | Manufacturing-specific events, AI Lab |
| Multi-cloud ($300K+) | AI Lab (broad), Dreamforce, Executive Summit, SIC |
| Security / Governance concerns | On-demand security & compliance webinars |
| Industry-specific pain (e.g., Healthcare/HLS) | On-demand industry-specific webinars matched to the pain point |
| Any product (nurture) | On-demand webinars as low-friction touchpoints — share immediately, no scheduling needed |

**Stage-based urgency:**

| Stage | Event Priority |
|---|---|
| 01-02 (Early) | Broad vision events (AI Lab, World Tour). Goal: educate and inspire. |
| 03-04 (Mid) | Product-specific workshops. Goal: technical validation and de-risking. |
| 05-06 (Late) | Executive events (dinner, SIC, Dreamforce). Goal: exec alignment for close. |
| Any stage + security/IT blocker | Guardrails Workshop. Goal: unblock technical evaluation. |

**Amount-based investment:**

| Pipeline | Event Investment Level |
|---|---|
| $400K+ | Dreamforce nomination, Executive Summit, SIC consideration |
| $100-400K | Regional workshops, AI Lab, executive dinners |
| $50-100K | Virtual workshops, webinars, regional half-day events |

### Phase 4: Generate Canvas

Create a Slack canvas titled: **[AE Name] — Event Recommendations | DD/Mon/YY**

Use this structure:

```markdown
# Event Recommendations for [AE Name]'s Territory

### *This canvas was generated using AI, which can produce inaccurate or harmful responses. Review for accuracy and safety before using.*

---

## New This Week — Webinars (Last 7 Days)

*Fresh webinar announcements matched to your pipeline. These are time-sensitive — register now.*

| Webinar | Date | Format | Matched Accounts | Match Type | Registration |
|---|---|---|---|---|---|
| [webinar name] | [date] | [live/on-demand] | [Account (Opp Name, $Amount, Stage)] | [Strong/Adjacent] | [customer-accessible link] |

**Matching logic:**
- **Strong match** — direct product name match (e.g., Field Service webinar → FSL opportunity)
- **Adjacent match** — related product family (e.g., Agentforce Sales webinar → Sales Cloud + Agentforce opportunity)

*If no new webinars were posted in the last 7 days, this section reads: "No new webinars this week. Check back next Friday."*

---

## Upcoming Events ([Region], [Today] – [config.fiscal_calendar.fy_end])

| Event | Date | Location | Format | Registration |
|---|---|---|---|---|
| [event name] | [date] | [city] | [format] | [link] |

---

## Account-to-Event Matching

### [Account Name] — $[total pipeline] ([# opps])

**Pipeline:** [opp names/products]
**Product Focus:** [primary products]

| Recommended Event | Why |
|---|---|
| **[Event Name — Location Date]** | [1-2 sentence rationale tied to their product focus and stage] |

[Repeat for each account ≥$50K]

---

## On-Demand Webinars (Share Immediately — No Scheduling)

| Webinar | Product Focus | Best For |
|---|---|---|
| [webinar name] | [product] | [which accounts/personas] |

*On-demand webinars require zero coordination — share the link directly with your customer contact as a value-add touchpoint between meetings.*

---

## Priority Actions

- [ ] [Prioritized checklist of registrations and nominations, ordered by event date]
- [ ] [On-demand webinars to share this week]
- [ ] [Virtual workshop recommendations]
- [ ] [Dreamforce / conference nominations]

---

## Matching Methodology

This canvas was produced by matching [AE]'s open pipeline (accounts, stages, product fit) against available regional events and Trailhead Academy workshops from [today] through [config.fiscal_calendar.fy_end].

**Matching logic:**
- Data/Analytics opps → Zero Copy workshops, Data 360 (SDC series)
- Agentforce/AI opps → Guardrails (unblock security), AI Lab (vision), Activation (AGT series)
- Service/Field Service opps → Agentic Contact Center, Service Agent Activation (AGT503)
- Multi-cloud strategic ($400K+) → Dreamforce, Executive Summit, SIC
- Early stage (01-02) → Broad vision events to educate
- Mid stage (03-04) → Product workshops to validate
- Late stage (05-06) → Executive events to close

*Generated: [date] | SE: [config.se_name]*
```

### Phase 5: Share & Follow Up

- Share the canvas link with the user
- Offer to send specific registration links or BASHOs for priority accounts
- Flag any events with limited capacity or approaching deadlines (within 2 weeks)

## Key Principles

- **Customer's need first, event second** — don't recommend events just because they exist; match to actual pipeline and stage
- **Less is more per account** — 1-2 events max per account. Don't overwhelm.
- **Full fiscal year view** — always look from today through the configured fiscal-year end date (`config.fiscal_calendar.fy_end`) unless told otherwise
- **Urgency matters** — events happening in the next 2 weeks get flagged as priority actions at the top
- **Virtual workshops are always available** — use Trailhead Academy workshops (AGT, SDC, ASC, REV, MSA, MOC series) as fallbacks when no regional event matches the timing
- **Dreamforce is the nuclear option** — only for strategic accounts ($400K+ or Transform bucket)
- **Every recommendation needs a customer-accessible link** — registration links for in-person events, Trailhead Academy URLs for virtual workshops (`trailheadacademy.salesforce.com/classes/[code]`), and direct URLs for on-demand webinars. If an AE has to go hunting for how to register, you've failed.
- **Customer-accessible ONLY** — never include links that require Salesforce internal authentication (salesforce.enterprise.slack.com, org62, salesforce-internal.slack.com, internal Google Docs, Basecamp, etc.). Every link in the canvas must be clickable by the customer. If the only link available is internal, note the event but mark the link as "(AE to share — internal registration)" and provide the customer-facing registration page if one exists.
- **Note limited capacity** — flag events that are filling up (from registration list reports in Slack)
- **Seasonal awareness** — Q3 (Aug-Oct) has Dreamforce; Q4 (Nov-Jan) is lighter on events but virtual workshops fill the gap
