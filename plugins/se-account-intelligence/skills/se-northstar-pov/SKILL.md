---
name: se-northstar-pov
description: "Builds a comprehensive Northstar Point-of-View document for an account — the strategic map that ties research, stakeholders, obstacles, future state, and Salesforce positioning into one cohesive artifact. Follows a 14-step framework designed for collaborative account planning."
metadata:
  type: sales-operations
  version: "1.1"
  depends_on: "se-account-research, se-executive-challenge-builder, se-agentforce-use-case-advisor, pccwop-narrative-builder"
---

# SE Northstar POV Builder

You build comprehensive Northstar Point-of-View documents — the strategic artifact that brings together all account research into a single, presentable planning document. This is the "one place" where AE, SE, and extended team align on account strategy: what we know, who matters, what's broken, what the future looks like, and why Salesforce wins.

## When to Use

- "Build the Northstar POV for [Account]"
- "Create the POV for [Company]"
- "Let's build the strategic map for [Account]"
- After completing SE Account Research — this is the visual/strategic synthesis
- Before an Executive Briefing Center (EBC) visit
- For QBR account planning sessions

## What It Produces

A Slack canvas structured across 14 sections that mirrors a Lucidchart collaborative board — designed to be transferred to Lucidchart, Miro, or presented directly from Slack.

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `account_root` | Base directory for account workspace folders — used to read pre-existing research (Account Research, Contract Decoder, Buyer Group Map, Success Forensics, etc.) and to write POV output artifacts. **Do not hardcode a personal path in this skill.** |

If `account_root` is missing, ask the user for the account folder path (or default to the current working directory) before reading or writing files.

## Inputs

- **Account** (required) — company name or account ID
- **Pre-existing research** (strongly preferred) — files in `<account_root>/[company]/` from prior skills (Account Research, Contract Decoder, Buyer Group Map, Success Forensics, etc.)
- **Additional context** (optional) — meeting notes, discovery call transcripts, executive priorities

## The 14-Step Framework

### Step 1: Account Research Overview

**Purpose:** Establish the foundation — who is this company, what's their market position, and why do we care?

**Content:**
- Company name, industry, HQ, employee count, estimated revenue
- Market position statement (1-2 sentences)
- Current ACV + total pipeline
- Account team (AE, SE, CSM, partners)
- Time context (when this POV was built)

**Sources:** Company Profile, Org62 Intelligence

---

### Step 2: Current Footprint

**Purpose:** What do they own, what are they using, and what's the gap between entitlement and adoption?

#### 2.1: Tech Footprint
- Full technology landscape (Salesforce + third-party + competitors)
- Integration architecture (what connects to what)
- Key platforms by function (ERP, CRM, Marketing, Service, Analytics, Security)

#### 2.2: Current Salesforce Usage
- Contracts → Line Items → Entitlements (from Contract Decoder)
- Adoption metrics (MAU, activation, shelfware)
- Feature depth (what's entitled but unused)
- Position Map (Incumbent / Entitled / Absent per category)

#### 2.3: Account Plan
- Open pipeline (all opps with stage, amount, owner, close date)
- Won history (last 3-5 deals)
- Renewal timeline and risk

**Sources:** Contract Decoder, Success Forensics, App Consolidation, Org62 Intelligence

---

### Step 3: Stakeholders & Customers

**Purpose:** Map the people who matter — buyers, champions, blockers, and the relationships between them.

**Content:**
- Buyer Group Map (all 7 groups with engagement status)
- Key contacts with roles, engagement level, last activity
- Relationship gaps (who's missing, who's dark)
- Activity velocity (who's active, who's gone silent)
- Multi-threading assessment

**Sources:** Buyer Group Map, Org62 contacts, Slack intel

---

### Step 4: Value Propositions

**Purpose:** Understand what this company actually DOES — how they create value for their customers.

**Content:**
- Core value delivered to their customers
- Products/services and how they're delivered
- Differentiation from their competitors (not ours)
- Problems they solve for their market
- Business model (subscription, product, service, hybrid)

**Sources:** Company Profile, Workshop Deck (Business Model Canvas), Web research

---

### Step 5: Org Chart

**Purpose:** Visualize reporting relationships and decision-making hierarchy.

**Content:**
- Executive leadership (C-suite + VPs)
- Reporting lines (who reports to whom)
- Decision-making authority (who signs checks, who evaluates, who blocks)
- Salesforce relationship map (who do WE know vs. who do we need to know)
- Power dynamics (formal authority vs. informal influence)

**Sources:** Org62, Buyer Group Map, competitive-intelligence POV tooling, LinkedIn research

---

### Step 6: Branding & Mission Statement

**Purpose:** Understand their aspirational identity — this becomes the opening of any executive conversation.

**Content:**
- Mission statement (from website/annual report)
- Vision statement (where they're going)
- Core values (stated publicly)
- Recent executive quotes that reveal strategic priorities
- Brand positioning (how they want to be seen)

**Sources:** Company website, competitive-intelligence POV tooling (executive priorities), earnings calls, press releases

---

### Step 7: Revenue Streams

**Purpose:** Understand HOW they make money — this shapes every ROI conversation.

**Content:**
- Revenue model breakdown (product sales, subscriptions, services, rental, licensing, etc.)
- Revenue mix (% by stream if known)
- Growth drivers (which streams are growing vs. flat vs. declining)
- Pricing model complexity (simple vs. complex — informs CPQ/Revenue Cloud conversations)
- Recurring vs. one-time revenue split

**Sources:** Company Profile, Workshop Deck, Web research, financial filings (if public)

---

### Step 8: Investor Relations / Financial Context

**Purpose:** Understand financial health, growth trajectory, and what stakeholders care about.

**Content for public companies:**
- Revenue, growth rate, margins
- Recent earnings themes
- Analyst consensus / concerns
- Stock performance context

**Content for private companies:**
- Estimated revenue range
- Growth signals (hiring, acquisitions, expansion)
- Ownership structure (PE-backed, founder-led, family)
- Financial decision-making culture (conservative vs. growth-oriented)

**Sources:** competitive-intelligence POV tooling, Web research, Org62 (ACV as % of revenue)

---

### Step 9: Customer Strategic Objectives

**Purpose:** Identify 3-5 EXECUTIVE-LEVEL strategic objectives — not IT projects, not Salesforce initiatives, but BUSINESS outcomes.

**Rules:**
- Must be in executive language (board-level, not admin-level)
- Must be achievable with or without Salesforce (not "implement Data Cloud")
- Must connect to revenue, margin, growth, or risk reduction

**BVS Value Equation framing:** Each objective is the "Business Objective" in the BVS formula — *Business Objective + Enablers = Outcomes*. Frame objectives so that Salesforce capabilities can be positioned as Enablers in Step 11, and Connected Vision outcome nodes become the Outcomes. Load `~/bvs-brain/digests/tools/bvs-self-service-hub.md` for the value equation vocabulary if needed.

**Good examples:**
- "Transition from transactional distribution to recurring service revenue"
- "Scale field operations 40% without proportional headcount growth"
- "Achieve 360-degree customer visibility across all business units"

**Bad examples (too tactical):**
- "Consolidate Salesforce orgs"
- "Implement Marketing Cloud"
- "Reduce case handle time"

**Sources:** competitive-intelligence POV tooling (executive priorities), Workshop Deck, Slack intel, Executive Challenge, `~/bvs-brain/digests/tools/bvs-self-service-hub.md`

---

### Step 10: Obstacles & Opportunities

**Purpose:** Identify what's blocking the strategic objectives (obstacles) and what's creating opening for change (opportunities).

**Obstacles format:**
- Specific, evidence-based (not generic)
- Tied to a strategic objective
- Quantified using the two-step protocol below

**Obstacle quantification protocol (two steps):**
1. **Size the population:** How many people, interactions, or transactions does this obstacle affect? (e.g., "100 reps × 30 min/day on manual CPQ work")
2. **Apply a loaded cost or velocity multiplier:** Use $75–150/hr for knowledge worker time, or a pipeline velocity impact (e.g., "each extra day in quote cycle = N% deal velocity loss"). Flag as internal estimate.

**If the data doesn't support quantification:** Write the consequence narrative instead — *"Each automation failure generates a support case, a manual workaround, and a rep productivity interruption. At 5 incidents/year, this compounds."* Named consequences beat empty cells.

**Opportunities format:**
- Market shifts, technology changes, competitive moves
- Internal momentum (new leadership, budget cycles, failed projects)
- Regulatory or compliance drivers

**Sources:** Workshop Deck (pain points), PCCWOP (Pain + Consequence), Case Signals, Slack intel, Competitive landscape, `~/bvs-brain/digests/benchmarks/agentforce-bvs-benchmarks.md`

---

### Step 11: Future State Vision (Tactics)

**Purpose:** Describe HOW the obstacles get removed and opportunities get seized — in USE CASE language, not product language.

**Rules:**
- Frame as capabilities, not products
- Use customer language ("drive customer self-service" not "deploy Experience Cloud")
- Connect each tactic to a specific obstacle or opportunity from Step 10
- Be specific enough to demo against

**BVS Value Chain mapping:** Each future state tactic traces the BVS value chain — *Workflow → Capacity → Effectiveness → Financial Outcome*. When filling the "Future State (Tactic)" column, describe the workflow change first, then the capacity it unlocks, then the effectiveness improvement, then the financial outcome. This creates the narrative thread that connects tactics to Connected Vision outcome nodes. Load `~/bvs-brain/digests/framework/value-play-catalogue.md` to identify the right BVS value play (Ch 1–4) for each tactic.

**Format:**
| Objective | Obstacle/Opportunity | Future State (Tactic) | Capability Required | Financial Outcome (Est.) |
|---|---|---|---|---|
| [from Step 9] | [from Step 10] | [workflow → capacity → effectiveness, in customer language] | [product family, not SKU] | [quantified estimate — internal only; not customer-ready without BVD validation] |

**Guardrail:** Financial Outcome estimates must reference a benchmark from `~/bvs-brain/digests/benchmarks/agentforce-bvs-benchmarks.md` or CS Metrics. If no benchmark applies, write "Sizing required — log BVD" rather than leaving the cell blank.

**Sources:** Agentforce Use Cases, Connected Vision, Account Research canvas, `~/bvs-brain/digests/framework/value-play-catalogue.md`, `~/bvs-brain/digests/benchmarks/agentforce-bvs-benchmarks.md`

---

### Step 12: Why Salesforce

**Purpose:** Articulate why Salesforce — specifically — is the right platform for THIS customer's future state.

**Rules:**
- Must be customer-specific (not generic "one platform" messaging)
- Must address their competitive alternatives directly
- Must reference their existing investment as an accelerator
- Must connect to at least 2 strategic objectives from Step 9

**Framework:**
1. **Existing investment** — what they already have and the switching cost argument
2. **Platform advantage** — what Salesforce does that point solutions can't (cross-cloud, single data model, AI native)
3. **Ecosystem fit** — AppExchange, partners, industry-specific solutions
4. **Competitive displacement** — why the alternative (Microsoft, HubSpot, etc.) falls short for THEIR specific needs
5. **Time to value** — why now vs. build/buy later

**Proof point requirement:** Each of the 5 framework elements must be paired with at least one externally shareable benchmark from `~/bvs-brain/digests/benchmarks/agentforce-bvs-benchmarks.md` or CS Metrics. Mark all proof points as "(Externally Shareable)" or "(Internal Only)" per the benchmark source.

**Quick reference — highest-value proof points for typical Why Salesforce sections:**
- Time to value vs. DIY: Futurum (4–6 weeks vs. >12 months custom AI) — shareable
- Quote speed: Salesforce internal (75% faster quote creation, 87% fewer clicks) — shareable
- Support deflection: Credit union / banking benchmark (60% case deflection, 30K min/day saved) — shareable
- Sales productivity: Salesforce Sales Coach (10% ↑ win rate) — shareable
- Setup accuracy vs. DIY: Valoir (Agentforce 4.8 months setup vs. 75.5 months DIY) — shareable

**Sources:** Competitive landscape, Contract Decoder (existing investment), PCCWOP (Proof section), `~/bvs-brain/digests/benchmarks/agentforce-bvs-benchmarks.md`

---

### Step 13: Why Now

**Purpose:** Create urgency through compelling events — specific, time-bound triggers that make inaction costly.

**Rules:**
- Must have a DATE or TIMEFRAME
- Must have a CONSEQUENCE of missing the window
- Must be real (not manufactured urgency)

**Format:**
| Compelling Event | Date/Trigger | What Happens If We Miss It |
|---|---|---|
| [event] | [specific date] | [specific consequence] |

**Cost-of-inaction calculation (internal use — not customer-ready without BVD):**
For each compelling event, estimate: *"If we miss this window, what does the next 12–24 months cost in unrealized value?"*

Formula: [Financial Outcome from Step 11 best use case] × [months of delay / 12]

Add an **Internal Urgency Anchor** row at the bottom of the Why Now table:
| Internal Urgency Anchor | Est. monthly value deferred | Audience |
|---|---|---|
| [derived from Step 11 financial outcomes] | [$X/month] | AE + SE deal team — not customer-facing |

**Sources:** Account Research (Why Now table), Pipeline close dates, Renewal dates, Slack intel, Step 11 Financial Outcome estimates

---

### Step 14: Theme Ideas

**Purpose:** Create 2-3 memorable, repeatable themes that capture the entire story in a phrase.

**Rules:**
- Must be customer-facing ready (could be said to an exec)
- Must be memorable (could be repeated without notes)
- Must capture the transformation, not the technology
- Should be testable: "Does this resonate with the champion?"

**Good examples:**
- "From 342 Reps to 492: Scaling Intelligence, Not Just Headcount"
- "Delivering the Connected Enterprise: One Customer, One View, Every Vertical"
- "Turning Decades of Expertise into 24/7 Digital Labour"

**Bad examples:**
- "Implementing Salesforce Across the Enterprise" (boring, product-centric)
- "Digital Transformation Journey" (generic, meaningless)

**BVS chapter routing:** Once a theme is selected, note which BVS chapter it routes to so the AE knows what to request from BVS:
- Transformation / vision themes → **Ch 2: Value Imperative** (CEO, board-level audience)
- Productivity / headcount efficiency themes → **Ch 4: Agentic Sales Transformation** (CRO, CFO audience)
- Business case / "why act now" themes → **Ch 1: Case for Change** (champion, VP audience)
- Any account where champion says "we tried AI" or "wait and see" → **Ch 3: De-Risk** (pair with any above)

Include the chapter routing alongside each theme option in the output.

**Sources:** Executive Challenge, PCCWOP (What If section), Company mission/vision, `~/bvs-brain/digests/framework/value-play-catalogue.md`

**Voice guidance:** Follow the voice rules in the `se-connected-vision` skill's `connected-vision-voice.md` file (bundled alongside its SKILL.md) for theme wording. Key rules: shortest possible, no rhyme/pun, tension implied not explained, generic = not done. The tagline should come from the customer's website if possible.

---

## Output Format

### Slack Canvas Structure

Create a Slack canvas titled: **Northstar POV: [Company] | [Mon DD, YYYY]**

Each of the 14 steps becomes a section with:
- **Section header** (Step number + title)
- **Content** (tables, narrative, or both as appropriate)
- **Evidence tags** (source attribution for each claim)
- **Open questions** (what we still need to validate — marked with ❓)

### Design Principles

1. **Executive-ready language.** Every section should be presentable to a VP+ without translation.
2. **Evidence-based.** Every claim has a source. Assumptions are labeled as assumptions.
3. **Action-oriented.** Each section ends with "What this means for us" (1 sentence).
4. **Living document.** Mark sections as 🟢 (complete), 🟡 (draft/needs validation), 🔴 (missing/blocked).
5. **Transferable.** Structure maps cleanly to Lucidchart sections, Miro boards, or slide decks.

---

## Coordination with Other Skills

| Skill | What It Provides to the POV |
|---|---|
| **se-account-research** | Steps 1-3 (foundation, footprint, pipeline) |
| **se-entitlement-decoder** | Step 2.2 (entitled capabilities, position map) |
| **se-buyer-group-mapper** | Step 3 (stakeholder map, engagement scores) |
| **se-executive-challenge-builder** | Steps 9-10 (objectives, obstacles) + Step 14 (themes) |
| **pccwop-narrative-builder** | Steps 10-11 (obstacles → future state narrative arc) |
| **se-agentforce-use-case-advisor** | Step 11 (future state tactics — specific agent use cases) |
| **se-competitive-intel-updater** | Step 12 (why Salesforce vs. alternatives) |
| **se-deal-coach** | Step 2.3 (pipeline health, methodology gaps) |
| **se-case-insight-analyzer** | Step 10 (obstacles revealed through support patterns) |
| **bvs-brain** (`~/bvs-brain/`) | Step 9 (value equation framing for objectives); Step 10 (obstacle quantification methodology); Step 11 (value chain mapping for tactics + financial outcome estimates); Step 12 (proof points — benchmark attachment per claim); Step 13 (cost-of-inaction anchor); Step 14 (BVS chapter routing for themes). Key digests: `tools/bvs-self-service-hub.md`, `framework/value-play-catalogue.md`, `benchmarks/agentforce-bvs-benchmarks.md` |

---

## Process

### If Pre-Existing Research Exists:

1. Read all files in `<account_root>/[company]/`
2. Map available data to the 14 steps
3. Identify gaps (steps with insufficient data)
4. Build canvas with available data; mark gaps as 🔴
5. Suggest specific actions to fill gaps

### If Starting Fresh:

1. Run `se-account-research` first (or gather context from user)
2. Then build the POV from the research output
3. Mark all sections as 🟡 until validated with customer intelligence

### BVS Engagement Triage (run at end of every POV)

After completing Steps 1–14, assess combined ACV + active pipeline and add a **BVS Recommendation** line to the POV's Open Questions section:

| Combined ACV + Active Pipeline | Recommendation |
|---|---|
| < $500k | Self-serve — use BVD 3.0 and Value Map templates |
| $500k–$750k | **Player/Coach** — AE/SE owns business case; log DSR for BVS review |
| > $750k | **Full BVS Engagement** — log DSR immediately; BVS co-delivers |
| Any size with C-suite business case need | BVS Office Hours or Deal Clinic — no minimum |

**Output line to add to Open Questions:**
> "BVS Triage: Combined ACV + pipeline = $[X]. Recommended engagement: [Self-Serve / Player-Coach / Full BVS]. [If Player/Coach or Full: Log DSR in OrgCS — Request Type: Business Value Services (BVS). Channel: `#bvs-roi-tco`.]"

---

## Rules

- **Never fabricate data.** If a section can't be populated, mark it 🔴 and say what's needed.
- **Use customer language in Steps 9-14.** No Salesforce jargon until Step 12.
- **Strategic objectives are BUSINESS outcomes.** If it sounds like an IT project, reframe it.
- **Every obstacle must have evidence.** Workshop quotes, case data, Slack intel — cite it.
- **Themes must be repeatable.** If you can't imagine the AE saying it in an elevator, rewrite it.
- **This is a LIVING document.** Always include a "Last Updated" date and a "Next Steps to Validate" section.

## Conversation Starters

- "Build the Northstar POV for [Account]"
- "Create the POV for [Company]"
- "I have all the research for [Account] — build the strategic map"
- "Update the Northstar POV for [Account] with new discovery notes"
- "Build Steps 9-14 for [Account] — I'll provide Steps 1-8 from the research canvas"
