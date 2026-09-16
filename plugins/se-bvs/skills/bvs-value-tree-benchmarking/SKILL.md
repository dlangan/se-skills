---
name: bvs-value-tree-benchmarking
description: "Enriches a value driver tree with CS Metrics benchmark data (25th/avg/75th percentile), public customer references with quantified outcomes, and internal BVS references. Accepts a drivers.yaml file, a pasted tree, or a canvas link. Writes benchmark data back to drivers.yaml bvs_impacts.cs_metrics fields."
metadata:
  type: business-value
  version: "1.1"
  depends_on: "bvs-executive-value-map"
---

# BVS Value Tree Benchmarking

Enriches a value driver tree with Salesforce Customer Success Metrics benchmark data, public customer references with quantified outcomes, and internal BVS references. The output is both a Slack canvas and — when a demo project is active — updated `cs_metrics` and `customer_references` fields in `drivers.yaml`.

## When to Use

- "Benchmark the value drivers for [Account]"
- "Find CS Metrics data for our FSL value tree"
- "Add references and benchmarks to the value map"
- After the BVS Executive Value Map is drafted — to harden the improvement % assumptions with real data
- Before an executive value conversation where credibility of numbers matters

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `account_root` | Base directory for account workspace folders — used to read the `drivers.yaml` value driver tree in Phase 1, and to write enriched `cs_metrics` / `customer_references` fields back to it in Phase 4. **Do not hardcode a personal path in this skill.** |

If `account_root` is missing, ask the user for the account folder path (or default to the current working directory) before reading or writing files.

---

## Execution Flow

### Phase 1 — Input Collection & Parse

Accept the value driver tree in any of these forms:
- `drivers.yaml` from the account folder (preferred — `<account_root>/<slug>/drivers.yaml`; reads all `bvs_impacts` entries)
- Pasted text with Objective → Value Lever → KPI hierarchy
- Canvas link (read via Slack MCP)
- Free-text description of the drivers

Also confirm:
- **Customer name** (for canvas title)
- **Customer industry** (for CS Metrics edition selection)

If reading from `drivers.yaml`: extract all drivers with `lever_type` and `dimension` populated. Each driver's `dimension` + `improvement_formula` fields become the KPI set for benchmarking.

### Phase 1b — Read-Back & Confirm

Present the full extracted tree in hierarchical bullet format:

> **[Objective or Product Area]**
> - Value Lever: [dimension]
>   - KPI: [what's being measured]

End with: *"Does this look right? Let me know if anything is missing before I start research."*

**Do not proceed until user confirms.** Once confirmed, the tree is locked — no restructuring after research begins.

### Phase 2 — Benchmark Research per Driver (run in parallel)

For each driver/KPI, run all three research streams simultaneously.

**A. Salesforce Customer Success Metrics (priority source)**

Search Slack and files for the latest CS Metrics edition:
- Search terms: "Customer Success Metrics FY27", "Customer Success Metrics FY26", "Customer Success Metrics FY25" — use highest FY found
- Search for industry-specific edition first (e.g., "Customer Success Metrics FY26 Manufacturing"); fall back to general if not found
- Always prioritize the customer's industry; if no match, use most applicable adjacent industry and note it

For each KPI, extract:
- Exact CS Metrics label
- 25th percentile value
- Average value  
- 75th percentile value
- Direct Slack file URL for the source

**Synonym search — exhaust all variants before using a proxy:**

| KPI | Also search for |
|---|---|
| Technician utilization | field resource utilization, FSL utilization, technician productivity |
| First-time fix rate | FTFR, first visit resolution, one-and-done rate |
| Time to dispatch | dispatch cycle time, time-to-assign, scheduling time |
| Case handle time | AHT, average handle time, case resolution time, time-to-close |
| Case opening time | case creation time, intake time, time to log |
| Win rate | opportunity win rate, close rate, conversion rate |
| Sales cycle length | time-to-close, days to close, sales velocity |
| Rep ramp time | time to productivity, onboarding time, new hire ramp |
| Customer status inquiries | inbound call volume, where-is-my-order, WISMO |
| Quote cycle time | CPQ cycle time, time-to-quote, proposal cycle |

Partial label matches count as direct matches — note the label difference in `notes` but do NOT mark as proxy.

**B. Public Customer References (mandatory — every KPI)**

Search salesforce.com customer stories, Slack, and files:
- Source priority: public website (salesforce.com/customers) > documents/PDFs > Slack messages
- Industry match first; adjacent industry with note if none found
- Only references with a **specific quantified outcome** (e.g., "reduced dispatch time by 35%")
- Select 1–2 references per KPI — mandatory, not optional
- Format: `[Customer Name](url): "quantified outcome"`
- If no URL after retry: `Customer Name (url unavailable): "outcome"` — last resort only
- If no quantified reference found after thorough search: write "No quantified reference found" — never leave blank

**C. Internal SF/BVS References**

Search Slack for past BVS business cases or ROI analyses with data for this KPI.
- Only include if **anonymous** (no identifiable customer name)
- Industry match preferred
- Place in `other_benchmark` field with Slack message link as source

**D. Proxy benchmarks — absolute last resort**

Only after exhausting all synonyms, partial matches, and adjacent industries. Must pass the common-sense test: plausible to say "while this metric is from [domain], it is directionally applicable because..."
- Always justify in `notes`: what the proxy is, where it's from, why it applies
- Never fabricate numbers — all data must trace to a real, findable source

### Phase 3 — Pre-Output Self-Check

Before creating any output, verify:
- [ ] Canvas title will be: `Business Value Case Benchmarking – [Customer Name] – [Date]`
- [ ] Single table — not split by objective or driver
- [ ] All rows present — one row per KPI in input order
- [ ] CS Metrics columns: Average, 25th %ile, 75th %ile, Source (clickable link) — each in own column
- [ ] Every Benchmark Public Reference cell has at least one `[Name](url): "outcome"` or explicit "No quantified reference found"
- [ ] All public references are hyperlinked — no plain-text customer names
- [ ] No fabricated numbers — every figure traces to a real source

If any item fails, resolve before proceeding.

### Phase 4 — Write Back to drivers.yaml

If `.demo-project/drivers.yaml` or `<account_root>/<slug>/drivers.yaml` exists:

For each driver that was benchmarked, update the `bvs_impacts` block:

```yaml
cs_metrics:
  label: "[exact CS Metrics label used]"
  p25: "[value with unit]"
  average: "[value with unit]"
  p75: "[value with unit]"
  source: "[Slack file URL]"
  edition: "CS Metrics FY[XX] — [Industry]"
  industry_match: exact    # exact | adjacent (note adjacent industry used)

customer_references:
  - customer: "[Name]"
    outcome: "[quantified outcome]"
    url: "[direct URL]"
  - customer: "[Name]"
    outcome: "[quantified outcome]"
    url: "[direct URL]"

other_benchmark: "[anonymized internal data point if found]"
other_benchmark_source: "[Slack message link]"
```

Also update `improvement_source` if a CS Metrics benchmark was found:
```yaml
improvement_source: "✅ SF Benchmark — CS Metrics FY[XX] [Industry], avg [value]"
```

### Phase 5 — Create Output Canvas

Create a Slack canvas titled: **`Business Value Case Benchmarking – [Customer Name] – [Date]`**

Canvas structure (in this order):
1. Summary callout block
2. Quick Summary section (3 sub-sections — also post as chat message)
3. Single enriched table

**Summary callout:**
```
Customer: [Name] | Industry: [Industry] | CS Metrics Edition: FY[XX] – [Industry] (n=[sample size]) | 
KPIs Enriched: [X of Y] | Proxy Benchmarks Used: [N] | No Quantified Reference Found: [N]
Benchmark data is directional. Do not share raw CS Metrics numbers externally.
```

**Enriched table columns:**
| Value Driver | Lever Type | CS Metrics Label | 25th %ile | Average | 75th %ile | CS Metrics Source | Benchmark Public Reference | Reference Source | Other Benchmark | Other Benchmark Source | Industry Match | Notes |

**Quick Summary (also post as chat message):**
- **Source & Coverage:** which CS Metrics edition, how many KPIs matched vs proxied
- **Key Benchmark Highlights:** top 3 most compelling data points with direct quotes
- **Things Worth Noting:** proxy benchmarks used, industry mismatches, gaps

---

## Key Principles

- **Industry-first** — always match customer industry for both CS Metrics and references; document mismatches
- **Most recent CS Metrics only** — never use an older edition if a newer one exists
- **Every reference must be hyperlinked** — no plain text customer names
- **Never fabricate** — every number must trace to a real, findable source
- **Proxy is last resort** — try all synonyms and adjacent industries before proxying
- **Single table always** — never split by objective, lever, or product
- **Dual output for Quick Summary** — always in canvas AND as chat message
- **Write back to YAML** — always update drivers.yaml when a project file exists; benchmarks without a home are wasted
- **Raw CS Metrics numbers are internal** — never include in customer-facing outputs
- **Connected Vision voice** — benchmark language used in pillar descriptions must follow `plugins/se-account-intelligence/skills/se-connected-vision/connected-vision-voice.md`; numbers enrich descriptions but never appear on outcome node labels
