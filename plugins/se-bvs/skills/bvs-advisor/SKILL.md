---
name: bvs-advisor
description: "Master orchestrator for the BVS skill suite. Assesses deal lifecycle stage, recommends and executes the right BVS activity, and carries context forward across sessions via the account folder. Coordinates bvs-discovery-questionnaire, bvs-executive-value-map, bvs-value-tree-benchmarking, and inline guidance activities (CxO POV, deal health check)."
metadata:
  type: business-value
  version: "1.1"
  depends_on: "bvs-discovery-questionnaire, bvs-executive-value-map, bvs-value-tree-benchmarking"
---

# BVS Advisor — Master Orchestrator

Assesses where a deal sits in the BVS lifecycle, recommends the right activity, and orchestrates the BVS skill suite. Context is carried forward across sessions via the account folder — no need to re-establish what's already been done.

## When to Use

- "BVS support for [Account]"
- "What BVS work do we need on [Account]?"
- "Help me prepare the value conversation for [Account]"
- Any deal where business value needs to be established, quantified, or defended
- Returning to a deal after prior BVS work — context recovery is automatic

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `account_root` | Base directory for account workspace folders — used to check for and read prior BVS outputs (`drivers.yaml`, `financials.md`, `account.md`, progress tracking) when recovering context on a returning deal. **Do not hardcode a personal path in this skill.** |

If `account_root` is missing, ask the user for the account folder path (or default to the current working directory) before reading files.

## BVS Activity Map

### Orchestrated Skills

| # | Activity | Skill | When |
|---|---|---|---|
| 1 | Build discovery questionnaire | `bvs-discovery-questionnaire` | Before or after discovery call; need to frame value conversation |
| 2 | Build outside-in value map | `bvs-executive-value-map` | Stage 3–4; need quantified value hypothesis for exec meeting |
| 3 | Benchmark value tree | `bvs-value-tree-benchmarking` | After value map drafted; need CS Metrics data and customer references |

### Inline Guidance Activities

| # | Activity | When |
|---|---|---|
| 4 | CxO POV preparation | Before executive meeting; need persona-targeted value narrative |
| 5 | Deal health check | Anytime; quick pulse on momentum, risks, open actions |
| 6 | Value defence | Customer pushes back on ROI; need objection-handling anchored in the value map |

---

## Execution Flow

### Phase 1 — Deal Identification & Context Recovery

**First time on this deal:**
1. Ask: "Which account and opportunity are we working on?"
2. Query Org62 for account and open opportunities
3. Check for account folder at `<account_root>/<slug>/`
4. Check for `.demo-project/` in current directory
5. Assess deal stage and recommend starting activity

**Returning to this deal (context recovery):**

Check the account folder for prior BVS outputs:
- `drivers.yaml` — driver map and BVS impact fields
- `financials.md` — customer financial context
- Any canvas links recorded in `account.md` or progress tracking

Search Slack for prior BVS canvases matching patterns:
- `Business Value Case Benchmarking – [Customer Name]`
- `[Customer Name] – Executive Value Map`
- `[Customer Name] — Salesforce Business Value Discovery Questionnaire`

Present what was found:
> "I found prior BVS work on this deal: [list with dates]. Carrying that forward. Based on current deal stage [X], I recommend [next activity] because [reason]. Want to proceed?"

**Context carry-forward rules:**

| Element | Carry Forward? | Verify? |
|---|---|---|
| Account name, industry, revenue, Salesforce footprint | Always | No — stable |
| Opportunity stage, ACV, close date | Always | Yes — check current Org62 data |
| Stakeholder map | Always | Yes — contacts change |
| Value drivers and quantification | Always | No — unless user flags changes |
| Financial data | Always | Yes — if older than 90 days, flag as potentially stale |
| Documents found (account folder files) | Always | No — link to them, don't re-search |
| Risks and open actions | Always | Yes — check if resolved |
| CS Metrics benchmarks | Always | No — unless user requests refresh |

### Phase 2 — Deal Lifecycle Assessment

Map current deal state to recommended BVS activity:

| Deal Stage | Primary Recommendation | Secondary |
|---|---|---|
| Stage 1–2 / Pre-discovery | Discovery Questionnaire — frame value conversation | — |
| Stage 2–3 / Post-discovery | Discovery Questionnaire + Value Map prep | Deal health check |
| Stage 3–4 | Executive Value Map | Benchmark if driver tree exists |
| Stage 4 | Value Tree Benchmarking | CxO POV prep |
| Stage 4–5 | CxO POV | Value defence prep |
| Any stage | Deal health check | Any above based on need |

Present recommendation:
> "Based on [context source], the deal is at Stage [X]. I recommend:
> 1. **[Primary]** — [1-sentence why]
> 2. **[Secondary]** — [1-sentence why]
> Which would you like, or tell me what you need?"

If the user's request clearly maps to a specific activity, skip assessment and go directly.

### Phase 3 — Skill Orchestration

When routing to a skill, pre-populate context to skip intake phases already covered:

**Pre-population mapping:**

| Skill | Pre-Populate From |
|---|---|
| `bvs-discovery-questionnaire` | Org62 account + opps, account folder, Slack discovery notes |
| `bvs-executive-value-map` | Discovery questionnaire output, drivers.yaml stubs, financials.md, account-research.md |
| `bvs-value-tree-benchmarking` | Value map output (drivers.yaml bvs_impacts), customer name and industry |

State what was pre-populated before starting the skill:
> "I already have [Account] in [Industry] with [Opportunity] at Stage [X] from the account folder. Starting from Phase [N] of the [Skill]."

Follow each skill's execution flow exactly — do not modify skill logic, output format, or key principles. Honor all user confirmation checkpoints defined in each skill.

### Phase 4 — Inline Guidance Activities

#### 4A — CxO POV Preparation

**Inputs needed:** Target executive (name, title, role in buying decision), meeting objective.

**Process:**
1. Research executive: search web for public statements, LinkedIn activity, conference talks. Search Slack for prior interactions or team commentary about this executive.
2. Map Salesforce value drivers (from value map / drivers.yaml) to the executive's stated priorities. Frame in their language and KPIs.
3. Build POV narrative: Executive Summary → Strategic context → Business challenge → Value opportunity → Proof points → Ask
4. Anticipate 3–5 likely questions with evidence-backed responses
5. Identify objections and value-anchored responses

**Output canvas:** `[Account Name] — CxO POV for [Executive Title] — [Date]`

Structure:
```
Executive Profile
- [Name, Title] — [what they care about, decision-making style]
- Known priorities: [from research]

Strategic Alignment
| Their Priority | Our Value Driver | Quantified Opportunity | Evidence |

Value Opportunity Summary
[3–5 sentences, quantified, in their language]
Total opportunity: [$range from value map]

Anticipated Questions & Responses
| Question | Response | Evidence |

Recommended Ask
[Specific, achievable ask for this meeting]

Preparation Notes (internal)
[What to avoid, sensitive topics, relationship status]
```

#### 4B — Deal Health Check

Quick pulse — no canvas unless requested.

**Process:**
1. Pull latest Org62 opportunity data: stage, close date, amount — what changed since last BVS work?
2. Search Slack for recent account mentions (last 14 days): risks, blockers, momentum signals, competitive mentions
3. Check open actions from prior BVS outputs — resolved or still open?
4. Compare to prior assessment: accelerating, on track, stalling, or at risk?

**Output (chat message):**
```
Deal Health Check — [Account] — [Date]

Status: 🟢 On Track / 🟡 Stalling / 🔴 At Risk

Key Signals:
- [Signal — source]
- [Signal — source]

Changes Since Last BVS Work ([Date]):
- [Change]

Open Actions:
- [Action] — [Owner] — ✅ Done / ⏳ Open / 🔴 Overdue

Recommended Next BVS Activity: [Activity] — [1-sentence reason]
```

#### 4C — Value Defence

When the customer pushes back on the ROI or business case.

**Process:**
1. Read `drivers.yaml` bvs_impacts — pull the quantified levers and their sources
2. Read `bvs_plausibility` — confirm all 4 gates passed (if not, flag as risk before the conversation)
3. Identify which lever(s) are being challenged
4. Prepare:
   - 3 value-anchoring statements tied to specific data points from the value map
   - Objection responses for: "too expensive", "we don't believe the ROI", "competitor X is cheaper"
   - Conservative scenario: what ∑A+B looks like at 50% realization — still compelling?

**Output canvas:** `[Account Name] — Value Defence Brief — [Date]` (internal only)

### Phase 5 — Context Update & Next Step

After every activity, update the account folder:

1. **If drivers.yaml exists:** confirm bvs_impacts fields were written by the executed skill
2. **Post activity summary to account.md or progress.md:** date, activity completed, key finding, canvas link
3. **Recommend next activity:**
   > "[Activity] is done. Based on what we found, I recommend **[next]** as the next step because [reason]. Want to proceed?"

---

## Key Principles

- **Context is the product** — always recover and carry forward prior work; never ask for information already in the account folder
- **One activity at a time** — execute one skill fully before moving to the next
- **Skill fidelity** — when executing a sub-skill, follow its flow and principles exactly; the Advisor adds orchestration, not modifications
- **Honor user checkpoints** — if a skill requires confirmation at a specific phase, always pause; never skip user interaction points
- **Graceful degradation** — if a source is unavailable, note the gap and proceed with what's available; never block on missing context
- **Account folder is the source of truth** — not Slack canvases, not memory; the account folder persists across sessions and is the BVS context store
- **Never fabricate** — all data points must trace to real, findable sources
- **Internal/customer boundary** — plausibility checks, analogues, CS Metrics raw data, and negotiation strategy never reach the customer
- **Connected Vision voice** — when the BVS sequence leads into a Connected Vision build, load `plugins/se-account-intelligence/skills/se-connected-vision/connected-vision-voice.md` to ensure value map language translates correctly into CV tagline, pillars, and outcome nodes
