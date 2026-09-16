---
name: se-full-account-package
description: "End-to-end account build orchestrator. Runs all SE skills in sequence to produce a complete strategic package: contract decoder, case insights, buyer group map, competitive intel, account research (deep dive with frameworks), Northstar POV, and Connected Vision. Creates the account folder, all .md files, all Slack canvases, and a progress canvas linking everything together."
metadata:
  type: sales-operations
  version: "1.1"
  depends_on: "se-entitlement-decoder, se-case-insight-analyzer, se-buyer-group-mapper, se-competitive-intel-updater, se-account-research, se-360-builder, se-northstar-pov, se-connected-vision"
---

# SE Full Account Package — Master Orchestrator

You build a complete strategic account package by running all SE skills in sequence. This is the "give me everything" skill — from raw contract data through to an executive-ready Connected Vision slide. Each step produces both a Slack canvas and a local .md file, and a progress canvas tracks completion across the entire build.

## When to Use

- "Build the full package for [Account]"
- "Full account build for [Account]"
- "Give me everything on [Account]"
- "Run the full account package for [Account]"
- "Account build for [Account]"
- Any request implying a comprehensive, end-to-end account strategy package

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `account_root` | Local root folder for account workspaces. This skill creates and reads `<config.account_root>/[slug]/`. **Do not hardcode a local path in this skill.** |
| `slack_team_id` | Used when constructing full Slack canvas URLs for the progress canvas and each step's canvas. |
| `todo_canvas_id` | If SE action items are surfaced during the build, append them to this canvas per the cross-skill to-do tracking behavior. |
| `se_name` | Populates the "SE" field on the progress canvas if not otherwise supplied. |
| `fiscal_calendar` | Used if any step needs to frame dates (e.g., renewal timing) in fiscal-quarter terms. |

This skill itself is a pure orchestrator — it doesn't decode entitlements, run SOQL, or map buyer groups directly. Each sub-skill listed in `depends_on` resolves its own config (e.g., `org_alias` for Org62 connections, `specialist_roster` for buyer group plays). Read this section's `account_root` value once at the start of the build and pass the resolved folder path to every step.

## What It Produces

| # | Deliverable | Skill Used | Outputs |
|---|---|---|---|
| 1 | Website Index + NotebookLM URLs | (manual web research) | .md files |
| 2 | Contract Decoder | se-entitlement-decoder | Canvas + .md |
| 3 | Case Insights | se-case-insight-analyzer | Canvas (if sufficient substance) + .md |
| 4 | Buyer Group Map | se-buyer-group-mapper | Canvas + .md |
| 5 | Competitive Intel Sweep | se-competitive-intel-updater | Canvas + .md |
| 6 | Account Research (Deep Dive) | se-account-research + se-360-builder (Step 9) | Canvas + .md + business-objectives.md |
| 7 | Northstar POV | se-northstar-pov | Canvas + .md |
| 8 | Connected Vision | se-connected-vision | Canvas + .md + Google Slide (collaborative) |
| — | Progress Canvas | (this skill) | Canvas (updated after each step) |

## Inputs

- **Account name** (required) — company name or Org62 account ID
- **Start from step** (optional) — skip already-completed steps (e.g., "start from step 4")
- **Skip steps** (optional) — exclude specific steps (e.g., "skip case insights — only 2 cases")

## Pre-Flight: Account Folder Setup

Before running any skill, ensure the account workspace exists:

1. **Derive the folder slug** — lowercase, hyphenated company name (e.g., "[Account Name]" → `account-name`)
2. **Create the folder** at `<config.account_root>/[slug]/` if it doesn't exist
3. **Verify write access** — create a test file and remove it

```
<config.account_root>/[slug]/
├── website-index.md          (Step 1)
├── notebooklm-urls.md        (Step 1)
├── contract-decoder.md       (Step 2)
├── case-insights.md          (Step 3)
├── buyer-group-map.md        (Step 4)
├── competitive-intel.md      (Step 5)
├── account-research.md       (Step 6)
├── business-objectives.md    (Step 6 — Business Capability Map)
├── northstar-pov.md          (Step 7)
├── connected-vision.md       (Step 8)
└── progress.md               (tracking file)
```

## Progress Canvas

Create immediately at start. Update after each step completes.

**Canvas title:** `Account Build: [Account] | [Mon DD, YYYY]`

### Progress Canvas Template:

```markdown
# Account Build: [Account] | [Mon DD, YYYY]

### *This canvas was generated using AI, which can produce inaccurate or harmful responses. Review for accuracy and safety before using.*

**Account:** [Name] | **Account ID:** [ID]
**AE:** [Name] | **SE:** [Name — default to config.se_name]
**Started:** [timestamp] | **Status:** In Progress

---

## Build Progress

| # | Step | Status | Canvas | .md | Notes |
|---|---|---|---|---|---|
| 1 | Website Index + NotebookLM URLs | :white_check_mark: / :hourglass: / :fast_forward: | — | [link] | |
| 2 | Contract Decoder | :white_check_mark: / :hourglass: / :fast_forward: | [ID] | [link] | |
| 3 | Case Insights | :white_check_mark: / :hourglass: / :fast_forward: | [ID] | [link] | |
| 4 | Buyer Group Map | :white_check_mark: / :hourglass: / :fast_forward: | [ID] | [link] | |
| 5 | Competitive Intel Sweep | :white_check_mark: / :hourglass: / :fast_forward: | [ID] | [link] | |
| 6 | Account Research (Deep Dive) | :white_check_mark: / :hourglass: / :fast_forward: | [ID] | [link] | |
| 7 | Northstar POV | :white_check_mark: / :hourglass: / :fast_forward: | [ID] | [link] | |
| 8 | Connected Vision | :white_check_mark: / :hourglass: / :fast_forward: | [ID] | [link] | |

**Legend:** :white_check_mark: Complete | :hourglass: In Progress | :fast_forward: Skipped | :red_circle: Blocked

---

## Key Findings (updated as they arrive)

- [Headline from Step 2: contract summary]
- [Headline from Step 3: case pattern]
- [Headline from Step 4: engagement score]
- [Headline from Step 5: competitive landscape]
- [Headline from Step 6: executive challenge 1-liner]

---

## Decisions Needed (Steps 7-8)

- [ ] Northstar POV: Validate strategic objectives with customer
- [ ] Connected Vision: Tagline selection
- [ ] Connected Vision: Pillar selection (3 of 4-5 candidates)
- [ ] Connected Vision: Outcome node refinement
```

## Execution Flow

### Phase A: Autonomous (Steps 1-6)

These steps run without user input. Report key findings as each completes.

#### Step 1: Website Index + NotebookLM URLs

1. Fetch the company website (homepage, about, leadership, products/services, news)
2. Build a structured index of what was found
3. Compile URLs suitable for NotebookLM import
4. **Output:** `website-index.md` + `notebooklm-urls.md`

#### Step 2: Contract Decoder

1. Query Org62 for active contracts on the account
2. Decode line items, entitlements, product license maps
3. Identify whitespace and dormant entitlements
4. **Output:** Canvas + `contract-decoder.md`
5. **Report:** "[X] products on [Y]-year deal, $[Z] ACV. Key finding: [dormant entitlements / whitespace / renewal risk]"

#### Step 3: Case Insights

1. Query Org62 for cases (last 12 months)
2. Categorize, filter admin/billing, identify themes
3. Map themes to Salesforce solutions
4. **Output:** Canvas (if ≥5 meaningful cases) + `case-insights.md`
5. **Report:** "[X] cases, [Y] P1/P2. Key pattern: [theme] → [solution signal]"
6. **If <3 meaningful cases:** Create .md only, note in progress canvas "insufficient substance for canvas"

#### Step 4: Buyer Group Map

1. Query Org62 contacts, OpportunityContactRoles, activities
2. Map contacts to 7 buyer groups
3. Assess engagement per group
4. Identify expansion plays ranked by priority
5. **Output:** Canvas + `buyer-group-map.md`
6. **Report:** "[X]/7 buyer groups engaged. Key gap: [unengaged group with highest whitespace opportunity]"

#### Step 5: Competitive Intel Sweep

1. Check CI record freshness
2. Sweep Org62 (opps, activities, account fields)
3. Sweep Slack (deal channels, broad search)
4. Build position map and signal table
5. **Output:** Canvas + `competitive-intel.md`
6. **Report:** "SF position: [incumbent in X, absent in Y]. Competitors: [named or 'none found']. CI record: [current/stale/empty]"

#### Step 6: Account Research (Deep Dive)

1. Combine all findings from Steps 1-5 (don't re-query — USE what's already gathered)
2. Conduct external research (company profile, PE/investors, leadership, industry, market)
3. Build Business Frameworks:
   - Business Model Canvas
   - Porter's Five Forces
   - Blue Ocean Strategy Canvas
   - SWOT Analysis
4. Write Executive Challenge Statement (digital labour framed)
5. Identify Agentforce Use Cases (mapped to their operations)
6. Build Connected Vision roadmap (phased)
7. Produce recommended actions (this week / month / quarter)
8. **Run Business Objective & Driver Mapping (se-360-builder Step 9):**
   - Derive 3-5 executive-level Business Objectives from the frameworks + challenge statement
   - Break each objective into 2-4 Business Drivers
   - For every driver, populate all 7 fields:
     - Business Objective (executive/board language)
     - Business Driver (specific sub-initiative or capability need)
     - Detailed User Story ("As a [role], when [situation], I need [capability], so that [outcome]")
     - Business Impact (what breaks or is lost without this capability)
     - Financial Impact (quantified with benchmark source; flag as internal estimate if needed)
     - Key Stakeholders (role titles affected)
     - Salesforce Solutions (specific products, agents, or features — not cloud families)
   - Produce Summary Scorecard (4-column rollup: Objective / Driver / Solutions / Est. Annual Impact)
   - **Output:** `business-objectives.md` (separate file from account-research.md)
9. **Output:** Canvas + `account-research.md` + `business-objectives.md`
10. **Report:** "[X] Agentforce use cases identified. [Y] business objectives mapped. Executive challenge: [1-sentence summary]. Roadmap: [# phases] from [quick win] to [transformation]"

---

### Phase B: Collaborative (Steps 7-8)

These steps require user decisions. Present options, wait for selection, then build.

#### Step 7: Northstar POV

1. Map all gathered data to the 14-step framework
2. Populate Steps 1-8 (foundation — mostly from Phase A outputs)
3. Draft Steps 9-14 (strategic — requires synthesis):
   - Step 9: Strategic Objectives (3-5, executive-level, business outcomes)
   - Step 10: Obstacles & Opportunities (evidence-based, tied to objectives)
   - Step 11: Future State Vision (tactics in customer language)
   - Step 12: Why Salesforce (customer-specific, not generic)
   - Step 13: Why Now (compelling events with dates and consequences)
   - Step 14: Theme Ideas (2-3 memorable, repeatable, customer-facing)
4. Mark sections as :green_circle: (complete), :large_yellow_circle: (draft/needs validation), or :red_circle: (blocked)
5. **Output:** Canvas + `northstar-pov.md`
6. **Pause:** Surface any decisions needed (e.g., "Steps 9-14 are drafted from available signals — should I proceed with these or do you want to adjust?")

#### Step 8: Connected Vision

1. **Propose tagline options** (2-3) — derived from customer's own language (website, mission, executive quotes)
2. **Propose pillar candidates** (4-5) — executive-oriented, mapped to strategic objectives
3. **Wait for user to select:** tagline + 3 pillars
4. **Derive 8 outcome nodes** from the selected pillars
5. **Present nodes for refinement** — user may swap, reword, or approve
6. **Produce final Connected Vision** in structured format
7. **Output:** Canvas + `connected-vision.md`
8. **Note:** The Google Slide is created manually by the user from the structured output. The skill produces the content, not the slide itself.

---

## Resumption Protocol

When the user says "continue the account build for [Account]" or "pick up where we left off on [Account]":

1. Read `<config.account_root>/[slug]/progress.md`
2. Identify the last completed step
3. Resume from the next incomplete step
4. Don't re-run completed steps unless explicitly asked

When the user says "start from step [N] for [Account]":

1. Create the account folder if needed
2. Mark steps 1 through N-1 as :fast_forward: Skipped
3. Begin at step N
4. If step N depends on outputs from earlier steps (e.g., Step 6 needs contract data), gather that data inline without producing the full earlier-step canvas

## Data Flow Between Steps

Each step builds on prior steps. The key dependencies:

```
Step 1 (Website) ──────────────────────────────────────┐
Step 2 (Contract) ─────────────────────────────────────┤
Step 3 (Cases) ────────────────────────────────────────┤──→ Step 6 (Account Research)
Step 4 (Buyer Groups) ─────────────────────────────────┤         │
Step 5 (Competitive Intel) ────────────────────────────┘         │
                                                                  ▼
                                                          Step 7 (Northstar POV)
                                                                  │
                                                                  ▼
                                                          Step 8 (Connected Vision)
```

**Critical rule:** Step 6 must NOT re-query data that Steps 1-5 already gathered. It synthesizes — it doesn't duplicate. Read the .md files from the account folder rather than re-running queries.

## .md File Requirements

Every step that produces output MUST create a .md file in the account folder. The .md file:

- Contains the FULL content (not just a link to the canvas)
- Includes the canvas ID and URL at the top (if a canvas was created)
- Uses the same structure as the canvas content
- Serves as the durable local record (canvases can be deleted; .md files persist)

## Progress Tracking (.md)

Maintain `<config.account_root>/[slug]/progress.md` with:

```markdown
# Account Build Progress: [Account]

**Started:** [date]
**Last Updated:** [date]
**Status:** [In Progress / Complete]

| # | Step | Status | Canvas ID | Completed |
|---|---|---|---|---|
| 1 | Website Index | :white_check_mark: | — | [date] |
| 2 | Contract Decoder | :white_check_mark: | [canvas ID] | [date] |
| 3 | Case Insights | :white_check_mark: | — | [date] |
| 4 | Buyer Group Map | :hourglass: | | |
| 5 | Competitive Intel | | | |
| 6 | Account Research | | | |
| 7 | Northstar POV | | | |
| 8 | Connected Vision | | | |
```

Update this file after EVERY step completion. This is how resumption works.

## Rules

- **Create the account folder first.** Before any queries, before any canvases. The folder is the workspace.
- **Every step produces a .md file.** No exceptions. Even if the canvas is skipped (e.g., case insights with <3 cases), the .md is created.
- **Don't re-query what's already gathered.** If Step 2 decoded the contract, Step 6 reads `contract-decoder.md` — it doesn't re-query Org62.
- **Report key findings progressively.** After each step, give the user a 1-line headline. Don't make them wait for Step 8 to learn something important from Step 2.
- **Steps 7-8 are collaborative.** Present options, wait for decisions. Don't auto-select pillars or themes — the user knows the account.
- **Update progress canvas after EVERY step.** Real-time visibility builds confidence.
- **Match the individual skill workflows exactly.** This skill orchestrates — it doesn't modify how each sub-skill operates. The contract decoder follows its own SKILL.md. The case insight analyzer follows its own SKILL.md. This skill just sequences them.
- **Handle thin data gracefully.** If an account has zero cases, note it and move on. If there's no CI record, note the gap. Don't block the build on missing data — flag it and continue.
- **The Connected Vision is always last.** It requires all prior steps as input. Never run it before the Northstar POV.

## Conversation Starters

- "Build the full package for [Account]"
- "Full account build for [Account]"
- "Give me everything on [Account]"
- "Account build for [Account] — start from step 4"
- "Continue the account build for [Account]"
- "Run the full package for [Account] — skip case insights"
