---
name: bvs-executive-value-map
description: "Builds an outside-in executive value map for a deal — revenue uplift and cost reduction levers quantified from customer financials, tagged with source provenance, and validated through a mandatory 4-gate plausibility loop. Writes quantified bvs_impacts back to drivers.yaml. Outputs a Slack canvas in customer language."
metadata:
  type: business-value
  version: "1.1"
  depends_on: "bvs-discovery-questionnaire"
---

# BVS Executive Value Map (Outside-In)

Builds a quantified, exec-ready outside-in value map. Every lever is grounded in the customer's own financials, tagged with source provenance, and validated through a mandatory 4-gate plausibility loop before output. When a demo project is active, writes quantified `bvs_impacts` back to `drivers.yaml` — making the value map the live financial backbone of the requirements chain.

## When to Use

- "Build the value map for [Account]"
- "What's the business case for [Account]?"
- "I need a quantified value hypothesis before the exec meeting"
- Stage 3–4 deals — after discovery, before formal business case
- After BVS Discovery Questionnaire inputs are collected

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `account_root` | Base directory for account workspace folders — used to read account context (`account.md`, `account-research.md`, `pccwop.md`, `northstar-pov.md`, `drivers.yaml`) in Phase 1, and to write quantified `bvs_impacts` back to `drivers.yaml` in Phase 5. **Do not hardcode a personal path in this skill.** |

If `account_root` is missing, ask the user for the account folder path (or default to the current working directory) before reading or writing files.

---

## Execution Flow

### Phase 1 — Understand Context

Accept inputs in any combination:
- Account folder (`<account_root>/<slug>/`)
- `.demo-project/` files (objectives.yaml, drivers.yaml)
- Org62 opportunity
- Call notes or canvas link
- Free-text description

Identify and confirm:
- Customer name, industry/vertical
- Salesforce solution(s) in scope — infer from opportunity line items; ask if ambiguous
- Key executive persona(s) targeted (CFO, CRO, COO, CIO)
- Revenue scale and cost base as quantification anchors
- Any DRT or prior quantification (ask explicitly if not provided — contains baseline inputs)

Read all available account folder files before proceeding. Do not ask for information that can be derived from files already present.

### Phase 2 — Gather Intelligence & Build Business Model Profile

Search all sources. Tag every data point with its provenance:
- 📄 *Investor/Earnings* — from investor decks, earnings calls, annual reports
- 🌐 *Customer Website* — from public company website
- 📊 *Public Data* — press releases, financial filings, analyst reports
- 💼 *Salesforce CRM* — from Org62 account, opportunity, activity records
- 💬 *Slack/Internal* — from Slack messages, canvases, or internal documents
- 📁 *Account Folder* — from account.md, account-research.md, pccwop.md, northstar-pov.md
- 🤔 *Assumption* — reasoned estimate where no data is available (always flag)

Extract specifically:
- Total revenue, revenue by division if available
- Key cost lines (headcount, IT spend, operational costs)
- Strategic priorities and stated transformation goals
- Known pain points and current-state metrics (from discovery notes, pccwop.md)
- Named KPIs or commitments from the customer
- Salesforce fit signals from opportunity or prior discussions

**At end of Phase 2 — produce the Business Model Profile (internal working document):**

```
## Business Model Profile

Business model framing: [2–3 sentences: what makes this customer genuinely distinct —
go-to-market model, revenue structure, operational complexity. Not their size.]

Business model traits for value quantification:
- [Trait 1: e.g. "Time-and-materials service labour — direct capacity-to-revenue link"]
- [Trait 2: e.g. "Tribal knowledge dependency — scheduling is person-dependent, not system-dependent"]
- [Trait 3: e.g. "SAP as ERP backbone — integration, not replacement, is the value unlock"]
- [Trait 4...]
- [Trait 5...]

Closest Salesforce analogues (internal only):
- [Reference 1]: [1-line rationale — business model trait match, not just industry]
- [Reference 2]: [rationale]
- [Reference 3]: [rationale]

CS Metrics industry classification: [which benchmark pool to use + caveats]
```

This profile drives all Phase 3 lever selection and calibration. Only include levers where a business model trait match justifies them.

### Phase 3 — Build the Value Map

Before selecting levers, re-read the Business Model Profile. Ask: which lever types are most credible for THIS customer's model? Which analogues have the strongest benchmark evidence? Exclude levers that don't fit.

**Revenue Uplift levers** (adapt to customer's business model):
- Increase in win/conversion rate
- Increase in average deal/order/job size
- Reduction in customer churn / increase in retention
- Upsell / cross-sell / NRR improvement
- Faster time-to-revenue / shorter sales or service cycle
- New market or segment penetration

**Cost Reduction / Efficiency levers** (adapt to customer's business model):
- Field force / commercial team productivity improvement
- Reduction in manual processes / administrative burden
- Faster cycle time (dispatch, quote, order, invoice)
- Reduction in errors, rework, exception handling, or failed first visits
- IT system consolidation or license rationalization
- Customer service deflection / handling time reduction

**For each lever, produce:**

1. **Value Driver** — short, punchy label in customer language. No Salesforce product names unless the customer uses them.
2. **Rationale** — 1–2 sentences: the qualitative mechanism and which Salesforce capabilities enable it. Reference capabilities by function, not product name.
3. **Baseline** — the current-state metric or cost pool being impacted. Tag with provenance emoji.
4. **Improvement Formula** — quantification logic in a single formula-style cell:
   - Format: `[Baseline] × [improvement%] × [margin or cost rate]`
   - Example (illustrative only): `35 technicians × 10% utilization lift × $85K CAD fully-loaded rate`
   - Tag improvement % as: ✅ SF Benchmark | 🔢 Industry Benchmark | 🤔 Assumption
5. **Profit Impact (p.a.)** — annual value as a range in customer's currency.
   - **Critical: revenue uplifts MUST be multiplied by gross margin before reporting as profit impact**
   - Cost reductions are reported directly
   - Format: `$[X]–[Y] [currency] p.a.`
6. **Baseline Source** — explicit source for the baseline figure
7. **Benchmark/Assumption** — the specific benchmark or assumption for the improvement %, with source citation

Include subtotals:
- **∑ A** — total profit impact from revenue uplift levers
- **∑ B** — total profit impact from cost/efficiency levers
- **∑ A+B** — combined total

Aim for 4–8 levers total. Balance: at least 2 revenue uplift, at least 2 cost/efficiency.

### Phase 4 — Mandatory Plausibility Loop

Run all 4 gates. If any gate fails, adjust lever assumptions and re-run. Iterate until all pass. This is mandatory — never skip.

**Gate 1: ∑A/∑B Ratio (revenue vs. cost balance)**

| Ratio | Signal | Action |
|---|---|---|
| < 1.0x | Revenue levers too conservative or missing | Review scope |
| 1.0–1.5x | Acceptable but revenue story is weak | Consider adding a revenue lever |
| 1.5–2.5x | ✅ Target range | Proceed |
| > 2.5x | Revenue levers too aggressive | Haircut largest revenue lever first, then re-check |

∑B is the firm anchor — only adjust ∑B if ∑A+B still fails Gate 3 after full revenue haircutting.

**Gate 2: ∑A+B / EBIT Ratio (credibility check)**

Estimate EBIT if not publicly available (use industry average margin, flagged as 🤔 Assumption).

| Ratio | Signal | Action |
|---|---|---|
| < 10% | Conservative — may be understating | Consider whether levers are complete |
| 10–30% | ✅ Credible and compelling | Proceed |
| 30–50% | Ambitious — add multi-year realization caveat | Add caveat |
| > 50% | High risk to credibility | Haircut aggressively; always add caveat |

Note: in low-margin industries (distribution, logistics ~3–5% EBIT), 20–30% of EBIT is already high — be extra conservative.

**Gate 3: ∑A+B / AOV Ratio (ROI check)**

| Ratio | Signal | Action |
|---|---|---|
| < 5x | Weak business case | Revisit levers or investment sizing |
| 5–10x | ✅ Minimum credible ROI | Proceed |
| 10–20x | ✅ Strong — compelling for exec audience | Proceed |
| > 20x | Question mark | Check for overly aggressive assumptions |

**Gate 4: Individual Lever Sanity Check**

For each lever contributing > 20% of ∑A+B:
- Is the baseline data verified or 🤔 Assumption?
- Is the improvement % sourced (✅/🔢) or 🤔 Assumption?
- If largest lever AND assumption-based: haircut to conservative scenario, or split into base/upside

**Iteration log** — track every adjustment:
```
v1: Initial → ∑A $X, ∑B $Y, ∑A+B $Z, EBIT ratio X%, AOV ratio Xx
v2: [What changed] → ∑A $X, ∑B $Y, ∑A+B $Z, EBIT ratio X%, AOV ratio Xx
```

Only finalize once all 4 gates pass (✅).

### Phase 5 — Write Back to drivers.yaml

If `.demo-project/drivers.yaml` or `<account_root>/<slug>/drivers.yaml` exists:

For each lever quantified, find the matching driver (match on title or dimension) and update `bvs_impacts`:

```yaml
bvs_impacts:
  - lever_type: "[revenue|cost]"
    dimension: "[driver category — matches value map lever label]"
    baseline: "[quantified baseline from value map]"
    baseline_source: "[provenance tag + source]"
    improvement_formula: "[formula from value map — e.g. '35 technicians × 10% × $85K CAD']"
    improvement_pct: "[X%]"
    improvement_source: "[✅ SF Benchmark | 🔢 Industry | 🤔 Assumption + citation]"
    profit_impact_pa: "[$X–Y currency p.a.]"
    cs_metrics:         # populated by bvs-value-tree-benchmarking — leave empty for now
      label: ""
      p25: null
      average: null
      p75: null
      source: ""
    customer_references: []   # populated by bvs-value-tree-benchmarking
    q_identity_ref: null      # populated when Solutions Workspace connected
```

Also add `bvs_plausibility` block at the top of the driver file:

```yaml
bvs_plausibility:
  ab_ratio: "[X.Xx]"         # ✅/🟡/🔴
  ebit_ratio_pct: "[X%]"     # ✅/🟡/🔴
  aov_ratio: "[Xx]"          # ✅/🟡/🔴
  lever_sanity: "[pass|review]"
  status: "[pass|review|fail]"
  last_run: "[YYYY-MM-DD]"
  notes: "[any caveats — e.g. 'revenue levers have multi-year realization caveat']"
```

### Phase 6 — Create Output Canvas

Create a Slack canvas titled: **`[Customer Name] — Executive Value Map (Outside-In View) — [Date]`**

Canvas structure:
1. Business Model Profile & Company Context (customer-facing)
2. Table A: Revenue Uplift levers
3. Table B: Cost Reduction & Efficiency levers
4. Total Value Picture (∑A, ∑B, ∑A+B, 5-year ROI indicator)
5. Strategic Value Dimensions (non-monetized factors worth naming)
6. Next Steps (validation checklist)
7. `--- INTERNAL USE ONLY ---` divider
8. Closest Salesforce Analogues (internal)
9. CS Metrics Classification (internal)
10. Plausibility Check results + iteration log (internal)

**Company Context table:**
| KPI | Value | Source |
|---|---|---|
| Revenue | $[X] p.a. | [source tag] |
| Gross Margin % | ~[X]% | [source or 🤔 Assumption] |
| Gross Profit | $[X] p.a. | Derived |
| EBIT | $[X] | [source or 🤔 Assumption] |
| Employees | [X] | [source] |

**Flag the canvas:** "Outside-In View — For Joint Validation with [Customer Name]"

This sets the right expectation — it's a hypothesis to validate together, not a final business case.

---

## Key Principles

- **Business model first** — only include levers where a trait match justifies them; don't force generic levers
- **Revenue uplifts × gross margin** — never report revenue as profit impact directly
- **Source every number** — every baseline and improvement % gets a provenance tag
- **Plausibility loop is mandatory** — never skip; a weak case that passes the gates is better than a strong case that fails them
- **Customer language** — no Salesforce product names unless the customer uses them
- **Outside-in framing** — this is a value hypothesis for joint validation, not a guaranteed ROI
- **Write back to YAML** — the value map is only useful if it feeds the requirements chain; always update drivers.yaml
- **Internal/customer boundary** — plausibility check, analogues, and CS Metrics classification never reach the customer
- **Currency consistency** — use the customer's currency throughout; infer from Org62 opportunity or ask
- **Connected Vision voice** — value map output language feeds directly into CV pillars and outcome nodes; follow `plugins/se-account-intelligence/skills/se-connected-vision/connected-vision-voice.md` for vocabulary and framing when the map will be used to build a CV
