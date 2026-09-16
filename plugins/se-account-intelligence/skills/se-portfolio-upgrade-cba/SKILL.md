---
name: se-portfolio-upgrade-cba
description: "Builds a defensible Portfolio 2.0 edition-upgrade cost-benefit analysis for a named account. Decodes the current footprint (contracts-only), recommends the right 2.0 edition (Core / Advanced / Max) from what we know about the customer, then builds an honest baseline, a two-band per-user comparison (Band A per-user, Band B %-of-net add-ons), a discount-sensitivity table, the à-la-carte-vs-bundle 'roughly half' punchline, and a customer-voice TL;DR (why change / why now / why Advanced). Outputs md + a Slack canvas. Use for any EE/UE/A1E → Portfolio 2.0 upgrade or renewal-uplift conversation."
metadata:
  type: sales-operations
  version: "1.2"
  depends_on: "se-entitlement-decoder"
---

# SE Portfolio 2.0 Upgrade CBA

You build the artifact an AE takes into a renewal or expansion conversation: **should this customer move to a Portfolio 2.0 edition, which one, and does it pencil?** The output is honest enough to survive an ROI-driven CFO and framed in the customer's own words.

This skill codifies a methodology proven in the field on a large-footprint manufacturer upgrade (EE → Sales & Service Advanced). It is edition-selection-aware: you **recommend** Core / Advanced / Max from the customer's signals rather than being told which one.

## When to Use

- "Build the Portfolio 2.0 upgrade case for [Account]"
- "Should [Account] move to Advanced or Max?"
- "What edition should [Account] land on at renewal?"
- "Cost-benefit for upgrading [Account] off Enterprise Edition"
- Any renewal where EE/UE/A1E → Core/Advanced/Max is on the table
- After `se-entitlement-decoder` / `se-full-account-package`, when the footprint is known and the conversation turns to the uplift

## Inputs

- **Account** (required) — name, ID, or Org62 URL
- **Current footprint** (strongly preferred) — from `se-entitlement-decoder` or an existing `contract-decoder.md`. If absent, decode it first (Step 1).
- **Account context** (optional but improves the edition call) — objectives, stalled/open pipeline, buyer profile, competitors, from `se-account-research` / the account folder.
- **Target edition** (optional) — if the AE already knows the target, skip the recommendation and go straight to the build; otherwise **you recommend it (Step 2).**

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `org_alias` | Org62 target for the `sfbase__ContractOrderSummary__c` footprint query (Step 1). |
| `account_root` | Where the `.md` deliverable is written — `<account_root>/<slug>/cost-benefit-analysis.md`. |
| `slack_team_id` | Builds the full Slack canvas URL handed to the AE (never just the canvas ID). |
| `se_name` | SE attribution in the document header block. |

Any key that's missing degrades gracefully: fall back to the conversation/account context for the footprint, write the `.md` to the current account folder if one is in play, and note that the canvas URL couldn't be fully formed if `slack_team_id` is absent.

---

## Portfolio 2.0 reference (confirm every number against SKU BOM + FAQ before it reaches a customer)

> :warning: **Pricing has known source discrepancies** (combined-CRM price, Flex pool sizes). The figures below are directional for *building the story*. Any number that goes in front of a customer — especially an ROI-gated one — must be reconciled against the current SKU BOM and Portfolio 2.0 FAQ at quote time. A wrong number kills trust.

| | Core | Advanced | Max |
|---|---|---|---|
| Replaces (End-of-Sale **Nov 1, 2026**; EOS ≠ EOL) | EE | UE | Agentforce 1 (A1E) |
| Single-cloud list /user/mo | — | **$395** | — |
| **Combined CRM (Sales + Service) list /user/mo** | **~$250** | **~$500** *(BOM shows $475 — reconcile)* | **~$675** |
| Flex Credits *(confirm on BOM)* | ~500K (≈50K/user) | ~1M (750K entitlement + 250K provisioning) | ~2.75M |
| Full Sandbox | ❌ | ✅ | ✅ |
| Enhanced security (Backup + Archive + Data Detect + Security Center) | ❌ | ✅ | ✅ |
| Premier Success Plan | ✅ | ✅ | ✅ |
| Slack | Business+ | Business+ | Enterprise+ |
| Built-in SPM | — | ✅ | ✅ |
| User minimum | — | **≥50% of users** when ACV > $250K | ACV > $500K |

**Band B standalone add-on pricing (priced as % of net cloud spend, not per-user):** Premier Success = **30%** (Signature = 40%); Full Sandbox = **30%** (Partial Copy 20% / Dev Pro 5%); Shield-class security ≈ **30%**; Security Center is a large six-figure standalone (Essentials is now free to all orgs). **Ad-hoc usage:** Flex ≈ **$5,000 per ~1M** ($0.005/credit); Service Cloud EE ≈ **$175/user/mo**; Slack Business+ ≈ **$15/user/mo**.

---

## Workflow

### Step 1: Decode the current footprint — contracts ONLY

The baseline must be real. Use `se-entitlement-decoder`, or if you're in Org62, query **`sfbase__ContractOrderSummary__c`** directly (the verified source):

```sql
SELECT sfbase__Contract__r.ContractNumber, sfbase__Contract__r.Status, sfbase__Contract__r.EndDate,
       sfbase__ProductFamily__c, sfbase__Quantity__c, sfbase__AutoRenewalQuantity__c,
       sfbase__AnnualOrderValue__c, CurrencyIsoCode
FROM sfbase__ContractOrderSummary__c
WHERE sfbase__Contract__r.AccountId = '<ACCOUNT_ID>' AND sfbase__Contract__r.Status = 'Activated'
ORDER BY sfbase__Contract__r.EndDate   -- keep only rows with Quantity > 0
```

**Non-negotiables:**
- **Contracts only.** Never infer the footprint from opportunities/OrderItem — they're cumulative bookings and overstate quantities.
- **Scope to the exact Account ID.** Never roll up the parent/hierarchy.
- Capture per line: product family, qty, ARR, contract #, **expiry date**, and the **per-unit rate**.
- For **per-user rates, use the actual per-unit Sales Price** from the line/asset — NOT ARR ÷ qty ÷ 12 if the term is multi-year. Mind the term: if a "Total Price" spans a 2-year term, the true monthly rate is Total ÷ qty ÷ **term-months**. Getting this wrong is the single most common error (it silently halves or doubles a rate).

### Step 2: Recommend the target edition — from what we know (the core call)

Read the signals you already have and land on **one recommended edition + SKU shape**, with the next tier as the named "destination." Default bias: **Advanced is the workhorse** for most Sales-primary EE/UE shops.

**Signals → edition:**

| Signal (from footprint + account research) | Pushes toward |
|---|---|
| Current edition | EE → **Core** (floor) or **Advanced** (step-up); UE → **Advanced**; A1E → **Max** |
| Stalled/open pipeline maps to bundled products (Agentforce, Slack, Data Cloud) | **Advanced+** — the consolidation thesis (N dead point-deals collapse into one edition move) |
| Owns no Service Cloud but needs it (service/portal/IT-desk objective) | **Combined CRM "super SKU"** (Sales + Service), not single-cloud |
| AI maturity — has NOT productionized any Agentforce | **Advanced** (credible first step); frame Max as destination |
| AI maturity — already consuming Flex at scale / agentic use cases live or imminent | **Max** (if ACV clears $500K and buyer has AI conviction) |
| Lacks Premier / Full Sandbox / Shield-class security today | **Advanced+** (these are Band B avoided-cost — big part of the value) |
| ROI-gated / cost-sensitive buyer | Start at the **credible** step (usually Advanced), require a BVS case, frame Max as the destination |
| Pure cost play, no AI appetite, no whitespace, ROI case not ready | **Core** — the no-regret, rename-class fallback |

**Decision defaults:**
- **Advanced (combined CRM)** — the default recommendation when there's stalled AI/collab/data pipeline to consolidate AND/OR Service Cloud whitespace, but the customer hasn't productionized Agentforce yet.
- **Core** — only when it's a pure cost/rename move: no AI appetite, no whitespace, ROI case not ready. Position as the no-regret floor (EOS ≠ EOL means there's no gun to their head).
- **Max** — only when they're already consuming Flex at scale or agentic use cases are live/imminent, ACV > $500K, and the buyer has AI conviction. Otherwise **quote Advanced, frame Max as the destination** once agentic use cases are in production.

State the recommendation in one line with the SKU #, then justify it against the signals. (Large-footprint example: "Advanced, combined CRM SKU 200004519 — Sales-primary today but Objective 4 needs Service they don't own; hasn't productionized any Agentforce, so Advanced is the credible first step, Max is the destination.")

### Step 3: Build the honest baseline

The baseline is **only what they pay today that the edition upgrades/absorbs** — the honest comparison base. This is not "$0 → edition" and it is not a rip-and-replace: the same seats upgrade **in place**.

- **Include:** the edition-replaceable core (e.g. Sales Cloud EE seats), plus any owned products the edition folds in (Inbox, classic Tableau → Tableau Next, Backup/Own).
- **EXCLUDE stalled/open opportunities.** They don't pay for those today — they are **whitespace the edition closes**, counted on the *value* side (Step 6/benefits), never the cost side. (Putting phantom pipeline in the baseline is the fastest way to lose CFO trust.)
- **Call out what stays separate** and is NOT absorbed (e.g. incentive comp / Spiff, Revenue Cloud, separate-org IT/HR) so no one over-promises the bundle.

Output a small table: baseline component → annual $ → what happens under the edition. Sum to the run-rate total.

### Step 4: The two-band comparison — apples-to-apples at the seat level

**Band A — per-user / month.** Two sub-groups:
- **Owned** — their real per-user rate today (Step 1) vs. $0 under the edition (the whole edition sits on the core line).
- **Net-new (❌ today)** — ad-hoc standalone list price for each thing the edition adds (Service, Slack, Einstein, Enablement, Agentforce/Data Cloud via Flex).

Mark directional prices "(dir.)". **Honesty caveat (always include):** the annual dollars are the exact basis; the per-user *sum* is illustrative because seat counts differ by product (not every seat carries every line).

**Band B — %-of-net add-ons the edition folds in at $0.** Premier Success, Full Sandbox, Shield-class security + Security Center + Data Detect — each ~30% of net cloud spend, computed on the core ARR — so the subtotal **scales with the core, it is not a fixed number** (a ~$226K core ≈ ~$200K/yr Band B; a ~$27K core ≈ only ~$16–24K/yr). Never carry the large-footprint example's ~$200K figure onto a smaller account. This is the avoided-cost that makes the deal pencil.

**The punchline — annual dollars.** Three rows: edition at list, edition at a realistic discount (~50%), and à-la-carte to replicate everything (owned stack + separate products + Band B + AI usage). The headline: **the edition at a realistic discount is roughly half the cost of assembling the same stack piecemeal.**

> :warning: **Scale check — run this before you write the punchline (it does NOT hold at every size).** The "roughly half" headline is a *large-footprint* result: it works when the core ARR is big enough that Band B avoided-cost (~30%-of-net × 3) and the owned/net-new stack dwarf the edition uplift. At a **small seat count** (roughly ≲ 30–40 seats, or core ARR well under ~$100K) the arithmetic inverts: the à-la-carte stack is small, Band B is small, and the edition at *list* costs **more** than assembling the pieces — even at a real discount it's roughly **neutral**, not "half." **Compute the three rows for the actual footprint before asserting anything.** If "half" doesn't hold, say so plainly and pivot the case from *savings* to *qualitative* (bundled security/compliance, Flex to activate dormant AI, contract consolidation, sequencing into the Connected Vision) — do NOT force the large-footprint headline onto a footprint it doesn't fit. (Small-utility example: 15 seats, ~$27K core → Advanced at list was **+$63K/yr**; effective net at 50% ≈ ~$2K/yr — the case was real but entirely qualitative.)

### Step 5: Discount sensitivity (never quote one number)

Net cost turns on the negotiated discount, which is Deal Desk's to set. Show a sensitivity, not a point:

| Scenario | Eff. $/user/mo | Edition annual (N seats) | Gross vs baseline |
|---|---|---|---|
| List / 40% / 50% / 60% | … | … | … |

Note that a large chunk of the gross increment buys Band B add-ons, so the **effective** net is far lower. State it plainly.

### Step 6: Net — does it pencil?

Walk it by benefit tier, honestly:
1. **Effective net** after crediting Band B avoided-cost (large-footprint example: ~$97K/yr at 50%).
2. **The floor** (efficiency + deflection) — does it cover the effective net?
3. **The upside** (margin defense + growth, BVS-validated) — does it clear every scenario incl. list?
4. **The à-la-carte reinforcement** — the "roughly half" anchor.

Be transparent: on efficiency/deflection alone it's often marginal at list; it pencils cleanly only with a real discount **and** a BVS-validated revenue thesis.

**When it does NOT pencil on cost (small footprints):** if the scale check (Step 4) showed the edition costs more than piecemeal, say so and re-anchor the recommendation on *sequencing + qualitative value* — the edition as a capability unlock gated behind the real strategic lever (e.g. MuleSoft/integration), positioned at the core renewal, not a near-term standalone upsell. Recommend leading with that lever, not the edition. (Small-utility example: Advanced was the right *eventual* edition but a Connected-Vision Phase-3 move — lead with MuleSoft, sequence the edition after.)

### Step 7: The TL;DR — in the customer's voice (why change / why now / why Advanced)

Lead the document with a TL;DR the AE carries into the room, **written from the customer's perspective** — the business case *they'd* nod along to, not the internal sell. Structure:

- **The one-liner** (blockquote) — their pain + the one-move fix + the timing + the "roughly half" economics, in their language.
- **Why [Customer] should change** — their business pain in their terms (fragmented data, margin pressure, manual service, stalled modernization). Use concrete tells from the footprint/research (e.g. duplicate exec records = "the data isn't connected").
- **Why now** — the renewal fork + EE End-of-Sale as the reason to start shaping now + decide-early-to-fold-initiatives-into-one-move.
- **Why [edition]** — one contract, ~half the à-la-carte cost, predictable AI spend, ~$200K/yr of bundled tooling, defends the two things they care most about, clear path to the next tier.
- Tuck internal-only mechanics (SPM overlap, BVS, Deal Desk) into a small italic *"For the rep"* note so they don't leak into the customer framing.

### Step 8: Costs, watch-outs & deal mechanics

- **One-time / implementation** (scope with SI): flag **Tableau classic → Tableau Next is a different product** (rebuild + retrain — commonly under-weighted); Service Cloud stand-up; Agentforce + Data Cloud enablement; Slack rollout; data dedup.
- **Recurring watch-outs:** Flex overage vs the included pool → overage or the next-tier step-up; Backup bundle size vs current units (size the gap); **org mixing (1.0/2.0) needs Deal Desk approval**.
- **Deal mechanics:** the real edition window is the **core renewal date** (not trivial off-cycle expirations); **EOS ≠ EOL** (they can still renew the old edition — so it's a genuine *renew-as-is vs. upgrade* decision, not a forced move); user minimums (Advanced ≥50% when ACV>$250K; Max ACV>$500K); contract-sprawl consolidation is itself a benefit.

---

## Output Format

**Delivery (dual-mode):** write the local `.md` **and** publish a Slack canvas (hand the AE the full canvas URL, not just the ID).

| Target | When | → |
|---|---|---|
| **Account folder** | inside `se-full-account-package`, or when `<account_root>/<slug>/` exists | `<account_root>/<slug>/cost-benefit-analysis.md` |
| **Slack Canvas** | standalone, or when a shareable link is wanted | `[Account] Cost-Benefit Analysis (→ [Edition]) [dd/Mon/yy]` |

If Slack is unavailable, write the `.md` and note the canvas wasn't created; publish once reconnected.

**Document order:**
1. Header block (Account, ID, AE, SE, Generated date, decision window) + companions
2. Honesty callout (net turns on discount; marginal at list without BVS)
3. **TL;DR — customer's case** (Step 7) — first thing after the header
4. §1 Baseline (Step 3)
5. §1b Two-band comparison + punchline (Step 4)
6. §2 Costs — subscription sensitivity (Step 5) + one-time + watch-outs (Step 8)
7. §3 Benefits (hard/quantifiable — BVS-gated; soft/strategic; à-la-carte value stack)
8. §4 Net — does it pencil (Step 6)
9. §5 Unknowns to close (pricing vs BOM, discount, migration effort, BVS)
10. §6 Recommendation

**Canvas rules:** ATX headings only; no tables inside callouts; section IDs change after every update — re-read before updating; share the full URL (built from `config.slack_team_id`).

## Rules

- **Contracts only, exact account scope.** Baseline footprint from `sfbase__ContractOrderSummary__c` (or `se-entitlement-decoder`), Quantity > 0, Status Activated, scoped to the given Account ID — never opportunities, never the parent.
- **Per-user rates from the actual per-unit Sales Price**, term-adjusted. Never divide a multi-year total as if it were annual. Reconcile every per-user figure to the annual dollars.
- **Honest baseline.** Only what they pay today that the edition absorbs. Stalled/open opps are value-side whitespace, never cost-side baseline.
- **You recommend the edition** from the signals (Step 2) unless the AE specifies one. Default to Advanced; reserve Max for scaled/committed AI + ACV > $500K; Core for pure cost plays.
- **Never quote one net number** — always a discount sensitivity, and always credit Band B avoided-cost to show the effective net.
- **Scale check before the punchline.** Band B and "roughly half" scale with the core ARR — they are not fixed figures. Compute the three punchline rows on the *actual* footprint first. Below ~30–40 seats (core ARR ≪ ~$100K) the edition typically costs *more* than piecemeal at list and is roughly neutral at a real discount — say that plainly and pivot the case to qualitative value (bundled security/compliance, Flex to activate dormant AI, consolidation, Connected-Vision sequencing). Never carry a large-footprint example's ~$200K Band B or "half" headline onto a smaller account.
- **Confirm every price against BOM + FAQ before it reaches a customer.** Flag the known discrepancies (combined-CRM $475 vs $500; Flex pool 500K vs 1M) as open items.
- **EOS ≠ EOL.** Never manufacture false urgency off End-of-Sale. It's a *reason to start shaping now*, not a gun to the head.
- **Don't over-promise the bundle.** Explicitly list what stays separate (incentive comp, Revenue Cloud, separate-org IT/HR).
- **Flag SPM overlap** when the customer owns a standalone commissions tool (Spiff) — built-in SPM may not fully replace it; confirm before that tool's renewal.
- **TL;DR is customer-voice.** Business rationale in their terms; internal mechanics only in the italic "for the rep" note.

## Integration with Other Skills

| Skill | Relationship |
|---|---|
| **se-entitlement-decoder** | Upstream — supplies the verified current footprint (the honest baseline) |
| **se-full-account-package** | Parent — this CBA slots in as the Portfolio 2.0 / upgrade artifact |
| **bvs-executive-value-map** | Downstream — hardens the margin-defense + growth numbers the CBA is gated on |
| **se-deal-coach** | Consumes the recommendation + watch-outs for the sales motion |
| **pccwop-narrative-builder** | The TL;DR pain framing can feed a fuller PCCWOP narrative |

## Conversation Starters

- "Build the Portfolio 2.0 upgrade CBA for [Account]"
- "Should [Account] go Advanced or Max?"
- "What edition should [Account] land on at their renewal, and does it pencil?"
- "Cost-benefit for moving [Account] off Enterprise Edition"
- "Make the case for [Account] to upgrade — in their words"
