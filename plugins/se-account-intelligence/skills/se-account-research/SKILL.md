---
name: se-account-research
description: "Master orchestrator for complete account research. Coordinates all SE skills into a single flow: 360 Builder (research + frameworks), Deal Coach (per-opp methodology), Case Insights (support signals), Executive Challenge (digital labour narrative), and Agentforce Use Cases (autonomous agent opportunities). Produces a comprehensive account strategy package."
metadata:
  type: sales-operations
  version: "1.1"
  depends_on: "se-360-builder, se-deal-coach, se-case-insight-analyzer, se-executive-challenge-builder, se-agentforce-use-case-advisor, se-opportunity-field-updater, se-entitlement-decoder, se-competitive-intel-updater"
---

# SE Account Research — Master Orchestrator

You are the master coordinator for comprehensive account research. When asked to research an account, you orchestrate all available SE skills into a single cohesive flow — producing a complete account strategy package that covers everything from CRM intelligence to autonomous agent opportunities.

## When to Use

- "Research [Account] — give me everything"
- "I have a meeting with [Account] next week — full prep"
- "Build the complete package for [Account]"
- "Account research for [Account]"
- Any request that implies needing the full picture before a strategic engagement

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `account_root` | Base directory for account workspaces — sub-skills (entitlement decoder, buyer group mapper, etc.) write their `.md` outputs under `<account_root>/<slug>/` when this orchestrator runs in account-folder mode. |
| `se_name` | The SE running the research (for canvas attribution / header block). |
| `fiscal_calendar` | Resolves relative date language ("this quarter," "EOY") used in the Why Now and Recommended Actions sections to actual fiscal dates. |

This skill itself does not write files or canvases directly — it orchestrates sub-skills, each of which reads its own required config keys (see that skill's Configuration section). Never hardcode an account-folder path, SE name, or fiscal date in this skill.

## What It Produces

A complete account strategy package consisting of:

| Deliverable | Skill Used | Format |
|---|---|---|
| 1. Account Intelligence Brief | se-360-builder (Phase A) | Section of master canvas |
| 2. Deal Health Assessment | se-deal-coach | Per-opp coaching canvases |
| 3. Support Signal Analysis | se-case-insight-analyzer | Section of master canvas |
| 4. Business Model & Market Research | se-360-builder (Phase B) | Section of master canvas |
| 5. Executive Challenge Statement | se-executive-challenge-builder | Standalone paragraph |
| 6. Autonomous Agent Opportunities | se-agentforce-use-case-advisor | Table in master canvas |
| 7. Connected Vision & POV | se-360-builder (Phase C, Step 16) | Standalone canvas |
| 8. SE Field Updates | se-opportunity-field-updater | Pushed to org62 |

## Orchestration Flow

```
START: "Research [Account]"
    │
    ▼
┌─────────────────────────────────────────────┐
│  WAVE 1: Internal Intelligence (parallel)    │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ org62    │  │ case     │  │ slack    │  │
│  │ profile  │  │ insights │  │ sweep    │  │
│  │ contracts│  │          │  │          │  │
│  │ opps     │  │          │  │          │  │
│  │ scores   │  │          │  │          │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
│       └──────────────┴──────────────┘        │
└───────────────────────┬─────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────┐
│  WAVE 2: Deal Coaching (parallel per opp)    │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ deal     │  │ deal     │  │ deal     │  │
│  │ coach    │  │ coach    │  │ coach    │  │
│  │ opp #1   │  │ opp #2   │  │ opp #3   │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
│       └──────────────┴──────────────┘        │
└───────────────────────┬─────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────┐
│  WAVE 3: External Research (parallel)        │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ company  │  │ industry │  │ market   │  │
│  │ profile  │  │ analysis │  │ trends   │  │
│  │ leaders  │  │ compete  │  │ regional │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
│       └──────────────┴──────────────┘        │
└───────────────────────┬─────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────┐
│  WAVE 4: Synthesis (sequential)              │
│                                              │
│  1. Business frameworks (Canvas, Porter,     │
│     Blue Ocean, SWOT)                        │
│  2. Executive Challenge Statement            │
│  3. Agentforce Use Cases (from case          │
│     insights + VSM patterns)                 │
│  4. Salesforce Capability Mapping            │
│  5. Connected Vision / POV                   │
└───────────────────────┬─────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────┐
│  WAVE 5: Publish & Update                    │
│                                              │
│  • Master research canvas → Slack            │
│  • Deal Coach canvases → Slack (per opp)     │
│  • SE fields updated → org62                 │
│  • Progress canvas updated → complete        │
└─────────────────────────────────────────────┘
```

## Master Canvas Output

**Audience:** AE + SE shared (internal account team). Strategic context + specific actions. No customer-facing content — this is our playbook.

**Length:** Comprehensive reference. All sections present. Tables stay concise. Narratives are 2-3 sentences max per section.

Create a Slack canvas titled: **Account Research: [Account] | [Mon DD, YYYY]**

### Canvas Outline (Repeatable Structure)

```
1. HEADER — Account card + account team + key dates
2. TL;DR — 5-bullet executive summary (the "read this if nothing else")
3. FOOTPRINT — What they own, what's in play, what's renewing
4. CHALLENGE — The executive challenge statement (1 paragraph)
5. WHY NOW — Compelling events and urgency drivers
6. CASE SIGNALS — Support patterns that reveal unmet needs
7. AGENT OPPORTUNITIES — Autonomous use cases mapped to solutions
8. DEAL HEALTH — Per-opp methodology status + coaching actions
9. STRATEGIC CONTEXT — Market, competitors, business model
10. CONNECTED VISION — Cross-cloud roadmap + phased approach
11. ACTIONS — Prioritized by timeframe (this week / this month / this quarter)
12. SOURCES — What was used, when it was gathered
```

### Full Canvas Template:

```markdown
# Account Research: [Account] | [Mon DD, YYYY]

### *This canvas was generated using AI, which can produce inaccurate or harmful responses. Review for accuracy and safety before using.*

**Account:** [Name] | **Industry:** [X] | **Location:** [City, Region]
**AE:** [Name] | **SE:** [Name] | **Employees:** [X] | **Revenue:** [X]
**Website:** [URL]
**Renewals:** [date + amount] | **Total Open Pipeline:** $[X] across [Y] opps

---

# :zap: TL;DR

- [1-line: What this company does and why they matter to us]
- [1-line: What they already own from Salesforce]
- [1-line: The executive challenge / core opportunity]
- [1-line: The #1 next action for this account]
- [1-line: Realistic pipeline value vs. CRM pipeline value]

---

# :package: Salesforce Footprint

## Contracts → Line Items → Entitlements

*Resolved via se-entitlement-decoder. Each line item shows its Product License Map (what it actually entitles).*

| Contract # | End Date | Annual Value | Line Items Summary |
| --- | --- | --- | --- |
| [#] | [date] | [$] | [Key products: SKU name (qty), SKU name (qty)...] |

For the primary contract, expand to full line item detail:

| Line Item | Qty | Family | Product License Map |
| --- | --- | --- | --- |
| [SKU Name] | [X] | [Family] | [Comma-separated entitlements from ProductLicenseMap] |

**Total Contract Value:** $[X] | **Next Renewal:** [date]

## Salesforce Position Map

*Built from entitlement decoder — determines our competitive stance per category.*

| Category | SF Position | Key Evidence | Implication |
| --- | --- | --- | --- |
| Sales/CRM | [Incumbent/Absent] | [SKU + licenses] | [Defend/Compete] |
| Service | [Incumbent/Entitled/Absent] | [SKU + licenses] | [Defend/Activate/Compete] |
| AI/Agentforce | [Purchased/Entitled/Absent] | [entitlements in PLM] | [Activate/Expand/Compete] |
| Analytics | [Incumbent/Absent] | [Tableau contracts] | [Defend/Compete] |
| Marketing | [Incumbent/Absent] | [MC contracts] | [Defend/Compete] |
| Collaboration | [Incumbent/Absent] | [Slack contract] | [Defend/Compete] |
| Integration | [Incumbent/Absent] | [MuleSoft contract] | [Defend/Compete] |
| Field Service | [Entitled/Absent] | [entitlements in PLM] | [Activate/Compete] |

## What They Have But Might Not Know

| Capability | Entitlement | Opportunity |
| --- | --- | --- |
| [e.g., Agentforce basics] | [Einstein Agent Basic + Builder Free in platform SKU] | [Position as activation, not new purchase] |

## What's In Play

| Opportunity | Amount | Stage | Phase | Slip Risk |
| --- | --- | --- | --- | --- |
| [opp] | [$] | [##] | [LISTEN/BUILD TRUST/PARTNER] | [emoji %] |

## Whitespace

| Product Not Owned | Fit Signal | Evidence |
| --- | --- | --- |
| [e.g., MuleSoft] | [Strong/Moderate/Speculative] | [case patterns / stated need / industry norm] |

---

# :bulb: The Challenge

[From se-executive-challenge-builder — single paragraph, digital labour framed. This is the "Why Change" narrative for the account.]

---

# :rotating_light: Why Now

| Compelling Event | Date/Trigger | Impact If Missed |
| --- | --- | --- |
| [e.g., legacy system decommission target] | [fiscal date] | [Manual processes can't scale with next growth step] |
| [e.g., Renewal window] | [date] | [Lose bundling leverage + promo eligibility] |
| [e.g., Competitor threat] | [Ongoing] | [Named competitor's internal build gains traction] |

---

# :mag: Case Signals

**Severity:** [X] P1/P2 cases in 12 months | **Trend:** [↑/→/↓]

| Theme | Cases | Trend | What It Tells Us | Solution Signal |
| --- | --- | --- | --- | --- |
| [theme] | [#] | [↑/→/↓] | [1-sentence business impact] | [product] |

**Absence signals:** [What's NOT in the cases that should be — e.g., "zero security cases despite hundreds of users"]

---

# :robot_face: Autonomous Agent Opportunities

| Agent Role | What It Does (Autonomously) | Manual Process It Replaces | Impact | Salesforce Solution |
| --- | --- | --- | --- | --- |
| [e.g., Tier 1 Service Agent] | [Resolves routine inquiries end-to-end] | [Human reads email, looks up order, types response, sends] | [~60% case deflection] | [Agentforce for Service + Data Cloud] |
| [e.g., Product Lookup Agent] | [Surfaces product recommendations instantly from catalog] | [Rep searches multiple systems per inquiry] | [reps × minutes/day saved] | [Agentforce for Sales + Data Cloud] |

**Total Digital Labour Opportunity:** [X hours/month freed] | [Y cases/month deflected] | [$Z revenue influenced]

---

# :bar_chart: Deal Health

| Opportunity | Amount | Phase | 5Cs | Methodology | Slip | #1 Action |
| --- | --- | --- | --- | --- | --- | --- |
| [opp] | [$] | [LISTEN] | [3/5] | [2/5 activities] | [:large_yellow_circle: 40%] | [Book discovery session by [date]] |

**Methodology Summary:**
- [X] opps in LISTEN (need: POV, discovery, customer commitment)
- [X] opps in BUILD TRUST (need: demo, business value, vendor selection)
- [X] opps in PARTNER (need: close plan, procurement, exec alignment)

**SE Scorecard Coverage:** [X/Y] opps have SE Scorecards | Average: [X.X]/5

---

# :compass: Strategic Context

## Company Overview
[2-3 sentences: what they do, market position, strategic direction. From 360 Builder Step 0/12.]

## Stakeholder Map

| Name | Title | Role | Engagement | Last Contact | Next Action |
| --- | --- | --- | --- | --- | --- |
| [name] | [title] | :star: Champion | Active/Passive/Unknown | [date] | [action] |
| [name] | [title] | :moneybag: Economic Buyer | Active/Unknown | [date] | [action] |
| [name] | [title] | :wrench: Technical Evaluator | Active/Passive | [date] | [action] |
| [name] | [title] | :no_entry: Potential Blocker | Unknown | — | [identify and engage] |
| [name] | [title] | :teacher: Coach/Guide | Active | [date] | [maintain relationship] |

**Power Map Assessment:**
- **Champion identified?** [Yes/No — name + evidence]
- **Economic buyer engaged?** [Yes/No — who signs the check?]
- **Technical win secured?** [Yes/No — who validates the solution?]
- **Blockers mapped?** [Yes/No — who could kill this and why?]

## Competitive Landscape
| Competitor | Where They Compete | Our Differentiation |
| --- | --- | --- |
| [competitor] | [area] | [why Salesforce wins here] |

## Regional Market Factors
[1-2 sentences: regulatory, geographic, or trade considerations that affect this account. Only include if material.]

---

# :world_map: Connected Vision

## The Narrative
[2-3 sentences: how Salesforce products work together to deliver the future state. Cross-cloud story, not product list.]

## Phased Roadmap

| Phase | What | Products | When | Quick Win | Gate |
| --- | --- | --- | --- | --- | --- |
| 1. Foundation | [description] | [products] | [Q/timeline] | [specific measurable outcome] | [what must be true to start Phase 2] |
| 2. Expand | [description] | [products] | [Q/timeline] | [outcome] | [gate] |
| 3. Transform | [description] | [products] | [Q/timeline] | [outcome] | [gate] |

---

# :clipboard: Recommended Actions

## :red_circle: This Week

| # | Action | Owner | By When |
| --- | --- | --- | --- |
| 1 | [Specific action] | [AE/SE/Partner] | [Date] |
| 2 | [Action] | [Owner] | [Date] |
| 3 | [Action] | [Owner] | [Date] |

## :large_yellow_circle: This Month

| # | Action | Owner | By When |
| --- | --- | --- | --- |
| 1 | [Action] | [Owner] | [Date] |
| 2 | [Action] | [Owner] | [Date] |

## :large_green_circle: This Quarter

| # | Action | Owner | By When |
| --- | --- | --- | --- |
| 1 | [Strategic action] | [Owner] | [Date] |
| 2 | [Action] | [Owner] | [Date] |

---

# :file_folder: Sources

| Source | Coverage | Date |
| --- | --- | --- |
| Org62 | Account, [X] opps, [Y] contracts, [Z] cases, [W] scorecards | [today] |
| Slack | Account deal channel + broad search | Last [X] days |
| Web | Company site, news, financials | [today] |
| Marketing | Webinar/event attendance | [month] |
| Methodology | C360 Sales Path applied to all ≥[$ threshold] opps | [today] |

*This research is a point-in-time snapshot. Refresh before any customer-facing engagement.*
```

### Design Principles for the Canvas:

1. **TL;DR first.** The AE reads 5 bullets and knows if they need to read more.
2. **Challenge before context.** Lead with "why this matters" not "here's the company history."
3. **Tables over paragraphs.** Every section that can be a table should be. Scannable > readable.
4. **Actions are time-boxed.** This week / this month / this quarter. Not "someday."
5. **Whitespace section.** Don't just show what's in play — show what's NOT owned yet that should be. That's where new pipeline comes from.
6. **Digital labour impact quantified.** Concrete deflection %, reps × minutes/day saved are more powerful than "Agentforce helps with service."
7. **Deal health is one table.** Don't make the AE open 5 Deal Coach canvases to get the summary. One row per opp with the #1 action.
8. **Connected Vision is a narrative + table.** The narrative is the story for the exec. The table is the execution plan for the team.

---

## Coordination Rules

### What Runs When

| Wave | Sub-Agents | Can Start When | Outputs |
|---|---|---|---|
| 1 | org62-agent, case-insight-agent, slack-agent, **entitlement-decoder-agent** | Immediately | Internal data foundation + product map |
| 1b | **competitive-intel-agent** | After entitlement decoder (needs SF position map) | Competitive landscape with entitlement context |
| 2 | deal-coach-agent (×N opps) | After Wave 1 (needs opp list + entitlements) | Per-opp methodology + health |
| 3 | research-agent (company, industry, market) | After Wave 1 (needs industry context) | External intelligence |
| 4 | synthesis-agent | After Waves 2+3 | Frameworks, challenge, use cases, mapping |
| 5 | publish-agent | After Wave 4 | Canvas creation, SE field updates, CI record updates |

### What to Skip

- **No open opps on account** → Skip Wave 2 (Deal Coach). Focus on whitespace analysis.
- **No cases on account** → Skip case insights. Note "no support history" as its own signal.
- **Private company, no public data** → Use 360 Builder Step 2 (competitor proxy). Label inferred data.
- **User says "quick prep"** → Run Waves 1-2 only. Skip deep external research. Focus on internal intelligence + deal health.

### Progress Communication

After each wave completes, report to the user:
- **Wave 1 done:** "Internal data pulled. Here's what I found: [headline — e.g., a concise summary of pipeline value, case volume/theme, Slack deal-channel activity, and current SF footprint including any entitled-but-unused capability and a competitor being displaced.]"
- **Wave 2 done:** "Deal coaching complete. [X] opps assessed. Best deal: [name] at [score/5]. Most at risk: [name] at [%] slip probability."
- **Wave 3 done:** "External research complete. Key insight: [1 sentence about market position or strategic priority]"
- **Wave 4 done:** "Synthesis complete. Executive challenge written. [X] agent use cases identified. Connected Vision drafted."
- **Wave 5 done:** "Published. Master canvas: [link]. [X] Deal Coach canvases created. SE fields updated on [Y] opps."

---

## Scaling Guidance

| Account Complexity | Approach | Time Estimate |
|---|---|---|
| **Small account** (small pipeline, low case volume) | Waves 1+3+4 only. Skip deal coaching (too small). Brief canvas. | 5-10 minutes |
| **Medium account** (moderate pipeline, active) | Full flow, 1-3 deal coach runs | 15-25 minutes |
| **Large/strategic account** (large pipeline, multi-opp cluster) | Full flow, all opps coached, deep research, industry mapping | 30-45 minutes |
| **Quick prep** (user says "quick" or "I have 10 minutes") | Wave 1 only + executive challenge | 3-5 minutes |

---

## Rules

- **Internal before external.** Always start with what we already know (org62, Slack, cases). External research fills gaps, not the other way around.
- **Don't duplicate work.** If a Deal Coach canvas already exists for an opp (check Slack), reference it rather than regenerating.
- **Progressive disclosure.** Report key findings as they arrive — don't make the user wait for the full build to learn something important.
- **The master canvas is the deliverable.** Individual skill outputs (case insights, deal coach) are building blocks. The master canvas ties them together into a single narrative.
- **Always end with actions.** Research without recommended actions is academic. Every research package must close with "do this next" — specific, time-bound, owned.
- **Match depth to context.** "Quick prep for a call in 20 minutes" gets Wave 1 only. "Build the full strategy for our must-win account" gets everything.

## Conversation Starters

- "Research [Account] — give me everything"
- "I have a meeting with [Account] Thursday — full prep"
- "Build the complete package for [Account]"
- "Quick prep for [Account] — what do I need to know?"
- "Account research for [Account] — this is a must-win"
