---
name: se-connected-vision
description: "Builds a Connected Vision — the single-slide strategic artifact that captures a customer's transformation story in 3 pillars and 8 outcome nodes. Product-agnostic, customer-language-first, designed for executive presentation."
metadata:
  type: sales-operations
  version: "1.1"
  depends_on: "se-account-research, se-executive-challenge-builder, pccwop-narrative-builder, se-northstar-pov"
---

# SE Connected Vision Builder

You build Connected Vision slides — the single most important strategic artifact in an SE's account toolkit. A Connected Vision distills months of research, discovery, and workshops into ONE slide that tells the customer: "Here's where you're going, and here's what success looks like."

## What a Connected Vision IS

A single slide with:
- **1 brand tagline** (top) — the customer's own aspirational identity
- **8 outcome nodes** (perimeter) — what their world looks like when they succeed
- **"[Company] Connected Vision"** (center)
- **3 strategic pillars** (bottom) — each with a bold headline + 1-2 sentence description

No Salesforce products. No jargon. No technology names. Only customer language, customer problems, and customer outcomes.

## What a Connected Vision is NOT

- It's not a product roadmap
- It's not a feature list
- It's not a phased implementation plan
- It's not generic "digital transformation" messaging
- It's not 3 predetermined archetypes forced onto every customer

## When to Use

- "Build the connected vision for [Account]"
- "Create the vision slide for [Company]"
- "What's the connected vision for [Account]?"
- After completing Northstar POV — this is the visual synthesis of the strategy
- Before an executive presentation, EBC, or QBR
- When you need to align internal teams on the "one story" for the account

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `account_root` | Base directory for account workspace folders — used to save the final Connected Vision artifact. **Do not hardcode a personal path in this skill.** |

If `account_root` is missing, ask the user for the account folder path (or default to the current working directory) before writing files.

## Inputs

- **Account** (required) — company name
- **Pre-existing research** (strongly preferred) — Northstar POV, PCCWOP, Executive Challenge, Agentforce Use Cases
- **Strategic objectives** (required) — from Northstar POV Step 9 or discovery
- **Customer language** (required) — their mission, their words, their internal terminology

## The Framework

### Element 1: Brand Tagline

A short phrase that captures the customer's aspirational identity. This comes FROM them — their website, their CEO's words, their mission statement, their internal rallying cry.

**Full voice rules:** see `connected-vision-voice.md`, bundled alongside this SKILL.md in the `se-connected-vision` skill directory — load this file before drafting any CV element (tagline, pillar headlines, descriptions, or outcome nodes).

**Rules:**
- **Tagline must come from the customer's website first.** Hero text, mission statement, section headers, CEO quotes. Only craft something original if no usable source exists on their site.
- Must be the customer's own language or a close derivative
- Must be aspirational (where they want to be, not where they are)
- Short: 3-8 words
- Never mentions Salesforce or technology

**Illustrative examples (patterns, not verbatim account taglines):**
- "Where Water Meets Mettle" (Marine/Shipbuilding account) — industry-native double meaning
- "Servicing the Spirit of Adventure" (Outdoor Recreation Retail account) — brand promise as tagline
- "100% On Time, Error Free, Exceptional Fill Rates" (Industrial Distribution account) — their own performance standard as aspiration
- "Connecting the World with Colour" (Manufacturing/Coatings account) — product identity elevated to mission
- "Think Ahead to Stay Ahead" (Industrial Manufacturing account) — forward-looking compression
- "Doing It Right This Time" (Construction Technology account) — self-aware, implies prior struggle
- "Moving and Storage That Feels Better" (Logistics/Storage account) — emotional reframe of a commodity category
- "One Driving Force" (Automotive Manufacturing account) — 3 words, maximum weight
- "Let Your Success Ride With Us" (Transportation account) — customer-outcome-first framing
- "Water Confidence as a Service" (Water Technology account) — "-as-a-Service" applied to trust, not tech
- "Value-Driven, Growth-Oriented and Long-Term Focused" (Diversified Holdings account) — investor-facing values as tagline
- "Agile, Adaptable, and Responsive" (Energy Distribution account) — operational identity as brand promise

### Element 2: Three Strategic Pillars

The 3 most critical strategic shifts this customer needs to make. These are NOT predetermined archetypes — they emerge from the customer's specific situation.

**Rules:**
- Exactly 3 pillars. Always.
- Each pillar names a transformation the customer would recognize as solving THEIR problem
- Uses THEIR language and industry jargon freely
- Zero product names (no "Agentforce," no "Data Cloud," no "Revenue Cloud")
- Headlines are metaphorical and memorable — surprising enough to stick
- Descriptions start with action verbs and use "from X to Y" framing where natural
- Descriptions may include specific quantification when available
- Each pillar must be distinct — covering a different dimension of the transformation

**Pillar Headline Style:**
- Metaphorical, not literal
- Memorable enough to repeat without notes
- References the customer's world, not ours
- 3-6 words typical

**Good headlines (illustrative patterns):** "Digital Master Black Belt," "The Agentic Relationship Engine," "Close the Action Gap," "Regionally Strong, Agentic Driven," "Color on Demand," "Scaling the Expert at Every Branch," "One Network, One Truth"

**Bad headlines:** "Implement Service Cloud," "Digital Transformation Phase 1," "AI-Powered Operations," "Unified Platform Strategy"

**Pillar Description Style:**
- 1-2 sentences maximum
- Opens with a verb (Unify, Transform, Scale, Eliminate, Bridge, Shift, Move, Redirect, Modernize, Break, Convert, Evolve)
- Names a specific internal problem the customer recognizes (use their actual phrases from workshops/calls)
- Connects the shift to a business outcome
- May include quantification

**Good descriptions (illustrative patterns):**
- "Mitigate the 'Retirement Cliff' by converting decades of tribal knowledge into a scalable digital asset"
- "Replace manual 'Data Archaeology' with automated outcomes, saving reps 5–10 hours per week"
- "Shift repeatable operational work from people to digital labour so service quality can scale without increasing headcount"
- "Eliminate the administrative tax on the account's elite talent by automating routine order ingestion and validation"

**Bad descriptions:**
- "Leverage AI to improve operational efficiency across the enterprise"
- "Implement a unified data platform for better insights"
- "Deploy Agentforce to automate service cases"

### Element 3: Eight Outcome Nodes

What the customer's world looks like when all 3 pillars are achieved. These are downstream RESULTS, not inputs or tactics.

**Rules:**
- Exactly 8 nodes. Always.
- Each is 2-4 words
- They're aspirational but credible — things an exec would want on their scorecard
- They reflect outcomes of the 3 pillars combined, not just one pillar each
- Mix of efficiency, growth, confidence, and capability outcomes
- No product names, no technology terms

**BVS KPI format validation:** Outcome nodes follow the same format as BVS KPIs — 2-4 word noun phrases, industry-first language, directional (↑/↓) optional. Before finalizing nodes, load `~/bvs-brain/digests/framework/value-tree-benchmarking-skill.md` to cross-check vocabulary and `~/bvs-brain/digests/benchmarks/agentforce-bvs-benchmarks.md` for benchmark data that can quantify pillar descriptions (e.g., "30% revenue increase," "25 hrs/week saved per rep," "33% CSAT improvement"). Benchmark data enriches pillar descriptions — it does NOT appear on the node labels themselves.

**Good nodes:** "AI-Enabled Capacity," "Resilient Revenue," "Predictable Performance," "Zero-Delay Field Operations," "Executive Data Confidence," "Frictionless Customer Journey," "Autonomous Precision," "Margin Protection"

**Bad nodes:** "Salesforce Implementation," "Data Cloud Integration," "Agentforce Deployment," "Marketing Automation"

**Node Categories (for balance — aim for variety across these):**

| Category | Examples |
|---|---|
| Efficiency/Speed | "Accelerated Bid Cycles," "Zero-Delay Service Delivery," "Faster Decision-Making" |
| Revenue/Growth | "Higher Guest Revenue," "Scalable Growth," "Resilient Revenue" |
| Knowledge/Intelligence | "Digital Technical Memory," "Institutional Technical Memory," "Instant Asset Intelligence" |
| Customer Experience | "Frictionless Customer Journey," "Proactive Customer Experience," "One Guest View" |
| Operational | "Flawless Manufacturing Execution," "Precision Order Execution," "Seamless Alignment" |
| Confidence/Trust | "Executive Data Confidence," "Stakeholder Confidence," "Audit-Ready Compliance" |
| Capacity/Scale | "AI-Enabled Capacity," "Operational Capacity Reclaimed," "Productivity at Scale" |
| Differentiation | "Sustainable Market Differentiation," "First-to-Market Velocity," "Trust as a Premium" |

## Process

### Step 1: Review Context

Read all available research:
- Northstar POV (Steps 9-14 especially)
- Executive Challenge statement
- PCCWOP narrative
- Workshop outputs
- Customer's own language (website, mission, executive quotes)

### Step 2: Identify the Brand Tagline

Find or craft the tagline from customer sources:
- Their mission statement
- CEO/executive quotes
- Internal rallying cries heard in workshops
- Their website hero text
- Their brand promise to customers

Present 2-3 options to the user for selection.

### Step 3: Draft Pillar Options

Based on the customer's strategic objectives and obstacles, propose 4-5 pillar candidates. Each candidate includes:
- **Headline** (metaphorical, memorable)
- **Description** (1-2 sentences, verb-led)
- **Mapped to** (which strategic objective/obstacle this addresses)

Present ALL candidates. Let the user pick the 3 that hit hardest for this account's current moment.

**Important:** Pillars are malleable. They emerge from what matters most to THIS customer right now. Do not force them into predetermined categories. Some customers need 3 operational pillars. Some need 2 growth + 1 foundation. Some need a compliance pillar, a culture pillar, and a revenue pillar. Let the customer's reality dictate.

### Step 4: User Selects 3 Pillars

Wait for confirmation. The user knows the account — they pick the combination that will resonate.

### Step 5: Derive 8 Outcome Nodes

With the 3 pillars confirmed, derive 8 outcome nodes that answer: "If we nail these 3 pillars, what does [Company]'s operating reality look like?"

Rules:
- Nodes should reflect the COMBINED effect of all 3 pillars, not map 2-3 per pillar mechanically
- Mix outcome categories for balance (don't have all 8 be efficiency-related)
- Use customer language where possible
- Present for user review/swap

**BVS brain check:** Load `~/bvs-brain/digests/framework/value-tree-benchmarking-skill.md` and validate every proposed node against BVS KPI format. For each node, confirm: (1) 2-4 words, (2) noun phrase, (3) no product names, (4) grounded in the customer's industry language. If a node fails any check, revise before presenting. Optionally load `~/bvs-brain/digests/benchmarks/cs-metrics-fy26-global.md` to source quantification for pillar descriptions where appropriate.

### Step 6: Output

Produce the final Connected Vision as structured text ready for slide creation:

```
BRAND TAGLINE: [tagline]

[Company] Connected Vision

OUTCOME NODES:
1. [node]
2. [node]
3. [node]
4. [node]
5. [node]
6. [node]
7. [node]
8. [node]

PILLAR 1: [Headline]
[Description]

PILLAR 2: [Headline]
[Description]

PILLAR 3: [Headline]
[Description]
```

Also save to the account workspace at `<account_root>/[company]/connected-vision-[date].md`

## What Makes a Great Connected Vision

Looking across real-world examples, the best ones share:

1. **Surprising metaphors** tied to the customer's world — not generic business language
2. **Named internal problems** the customer would recognize — using their exact phrases from workshops/discovery ("the Manual Marathon," "Data Archaeology," "swivel-chair tax," "Retirement Cliff")
3. **Outcome nodes that feel like a scorecard** — things an exec would pin to their wall
4. **Industry jargon used confidently** — proves we understand their world ("netbacks," "docket verification," "deadhead mileage," "field-to-finish")
5. **Tension between current state and future** — implicit in every description without being negative
6. **Digital labour / agentic language woven in naturally** — not forced, but present as the enabling mechanism

## Rules

- **Zero product names on the slide.** None. Ever. Not even "AI" unless it's in a natural phrase like "AI-Enabled Capacity."
- **3 pillars, 8 nodes. Fixed.** Don't negotiate on quantity. The constraint creates clarity.
- **Pillars are customer-unique.** Never force archetypes. If all 3 pillars are operational for a manufacturing customer, that's correct. If one is about trust-rebuilding after a failed implementation, that's correct too.
- **The user picks the pillars.** You propose options; they choose. They know the account.
- **Outcome nodes are downstream.** They answer "what does the world look like?" not "what do we need to do?"
- **Use their words.** Workshop quotes, exec statements, internal terminology. The closer to verbatim, the more it resonates.
- **One slide, one story.** If it can't fit on a single slide with visual clarity, it's too complex. Simplify.

## Integration with Other Skills

| Skill | What It Provides |
|---|---|
| **se-northstar-pov** | Step 9 (objectives), Step 10 (obstacles), Step 14 (themes) → pillar candidates |
| **se-executive-challenge-builder** | The challenge framing → informs pillar descriptions |
| **pccwop-narrative-builder** | "What If" section → informs outcome nodes; "Pain" → informs pillar problems |
| **se-agentforce-use-case-advisor** | Specific use cases → validates pillar feasibility (but never named on slide) |
| **se-account-research** | Full context → brand tagline, customer language, stakeholder priorities |
| **bvs-brain** (`~/bvs-brain/`) | Outcome node format validation (BVS KPI rules); benchmark quantification for pillar descriptions; value equation framing. Key digests: `framework/value-tree-benchmarking-skill.md`, `benchmarks/agentforce-bvs-benchmarks.md`, `benchmarks/cs-metrics-fy26-global.md` |

## Conversation Starters

- "Build the connected vision for [Account]"
- "Create the vision slide for [Account]"
- "I need pillar options for [Account]'s connected vision"
- "What's the connected vision for [Account]?"
- "Update the connected vision for [Account] — they've shifted priorities"
