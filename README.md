# SE Skills

A Claude Code **plugin marketplace** that packages a Solutions Engineer's account, deal, and territory workflows as installable skill bundles. Point Claude Code at this repo, install the families you want, set one config file, and Claude gains ~55 repeatable SE motions — account research, Connected Visions, BVS value maps, territory briefs, Dreamforce planning, and more.

> **Status:** Phase 1 build. The marketplace scaffold is in place, and the **entire `se-account-intelligence` plugin — 13 skills — is migrated, de-personalized, and PII-scrubbed** (verified by a repo-wide scan: no org alias, no hardcoded paths/IDs, no real customer or person names). The other five plugins' skills are still being moved from a single personal `~/.claude/skills/` install. See [Portability & configuration](#portability--configuration) — the not-yet-migrated skills still carry hardcoded values that a shared install replaces with your own via `se-config.json`.

---

## What's inside

This repo is **one marketplace** hosting **six plugins**, each a family of related skills. Install all, or just the ones you need.

| Plugin | What it does | Flagship skills | Prereqs |
|---|---|---|---|
| **se-account-intelligence** | End-to-end account research → strategy → executive narrative | `se-full-account-package`, `se-360-builder`, `se-northstar-pov`, `se-connected-vision`, `se-buyer-group-mapper`, **`se-key-contact-review`** ✅ | Org62, Slack, brains |
| **se-territory-ops** | Weekly territory hygiene, deal reviews, Org62 writes | `se-ae-territory-brief`, `se-scorecard`, `se-avp-*`, `se-co-prime-*` | Org62, Slack, `sf` CLI |
| **se-bvs** | Business Value Services — discovery → value map → benchmarking | `bvs-advisor` + 3 | Org62, Slack, Google Workspace |
| **se-dreamforce** | Dreamforce account/attendee profiling + session recommendations | `df27-account-onboard`, `df27-*`, `df-update` | Org62, Google Workspace |
| **se-advisory** | Cross-cutting advisory & enablement | `sf-dse`, `pipeline-finder`, `sf-product-recommender`, `qbrix-advisor`, `call-notes-analyzer`, `promo-*`, `se-trailhead-recommender` | Org62, Slack, brains |
| **se-afo-slack** | Agentforce-Operations advisors (Slack-workspace-scoped) | `afo-*`, `account-partner-intelligence`, `microsite-narrative-builder` | Slack (specific workspace) |

Plus a shared **`brains/`** knowledge bundle and a shared **`config/`** layer used across plugins (see below).

---

## Prerequisites

These skills orchestrate external systems through MCP servers and the Salesforce CLI. Install/authenticate what the plugins you want require:

| Dependency | Needed by | Notes |
|---|---|---|
| **Org62 SOQL MCP** (`salesforce-org62`) | ~40 skills | Read-only query access to Org62. The dominant dependency. |
| **Slack MCP** | ~35 skills | Canvas creation is the default deliverable format; also search/read. |
| **Salesforce CLI (`sf`)** | 6 write skills in `se-territory-ops` | Authenticated to *your* Org62 (see config — do **not** inherit the author's alias). |
| **Google Workspace MCP** | `se-bvs`, `se-dreamforce`, `call-notes-analyzer`, `se-engagement-template` | Docs/Sheets/Slides output. |
| **Bundled brains** | account-intelligence, advisory | Shipped in `shared/brains/` — no external fetch needed. |
| Niche: Trailhead, Highspot/Seismic, Huron/Trino, orgcs | individual skills | Only for `se-trailhead-recommender`, `microsite-narrative-builder`, `se-org-health-check`. |

## Installation

```text
# 1. Add this marketplace to Claude Code
/plugin marketplace add <your-org>/se-skills

# 2. Install the families you want (repeat per plugin)
/plugin install se-account-intelligence@se-skills
/plugin install se-bvs@se-skills
# ...etc

# 3. Configure (one time) — copy the example and fill in your details
cp shared/config/se-config.example.json ~/.claude/se-config.json
```

> Exact command syntax is confirmed against the current Claude Code plugin spec during build; if these differ, the finalized commands live in [`docs/INSTALL.md`](docs/INSTALL.md).

---

## Portability & configuration

**This suite was authored for one SE (single Org62 alias, one Slack workspace, one Canadian territory).** Making it shareable means every personal value is externalized into a single config file instead of being hardcoded in a skill. A shared install reads `~/.claude/se-config.json`:

```jsonc
{
  "org_alias": "you@salesforce.com",      // replaces hardcoded --target-org
  "account_root": "~/accounts",            // your local account-workspace folder
  "slack_team_id": "T0XXXXXXXX",           // your Slack workspace team ID
  "todo_canvas_id": "F0XXXXXXXXX",         // your personal SE Action Items canvas
  "se_name": "Your Name",                  // canvas footers, doc curator
  "fiscal_calendar": { "fy": "FY27", "q_end": "2027-01-31" },
  "ae_roster": [ /* your AEs: name + Org62 User ID */ ],
  "ae_roster_path": "~/.claude/se-ae-roster.md", // AE-alignment file for se-key-contact-review
  "contact_key_contact_field": "Key_Contact__c", // Contact field se-key-contact-review stamps
  "specialist_roster": [ /* co-prime specialists you cover */ ]
}
```

### Known portability work (tracked in [`docs/PORTABILITY.md`](docs/PORTABILITY.md))

| Severity | Blocker | Affected | Resolution |
|---|---|---|---|
| 🔴 | `--target-org` hardcoded to author's Org62 | 4 write skills | → `config.org_alias` |
| 🔴 | Personal Slack canvas ID baked into shared logic | todo-tracker, hubbl-health360, call-notes | → `config.todo_canvas_id` |
| 🔴 | Real customer PII embedded in SKILL.md as "reference log" | df27 profiling skills | **Scrubbed before publish** |
| 🟠 | Inconsistent account root (3 conventions) | ~10 skills | → `config.account_root` |
| 🟠 | Hardcoded AE / specialist rosters + Slack IDs | territory/co-prime/AVP | → `config.*_roster` |
| 🟡 | Org-instance IDs (Scorecard metrics, UsageType, Campaign) | scorecard, org-health, df | Portable within same Org62; documented |
| 🟡 | Slack-native, workspace-scoped skills | `se-afo-slack` family | Only run inside the original Slack workspace |

## Dependency graph

Several skills are **orchestrators** that invoke a tree of others — grouping follows these trees, so install the whole plugin, not individual skills. Full map in [`docs/DEPENDENCIES.md`](docs/DEPENDENCIES.md). Highlights:

- **`se-full-account-package`** → entitlement-decoder · case-insights · buyer-group-mapper · competitive-intel · account-research · 360-builder · northstar-pov · connected-vision
- **`se-connected-vision`** / **`se-northstar-pov`** → account-research · exec-challenge · use-case-advisor · pccwop-narrative-builder
- **`bvs-advisor`** → discovery-questionnaire · executive-value-map · value-tree-benchmarking
- **`df27-account-onboard`** → account-profile · attendee-profile · recommendation-doc

**Cross-plugin couplings to be aware of:** the `se-bvs` skills read the Connected Vision *voice* file from `se-account-intelligence`; `df-update` (dreamforce) feeds `se-event-recommender` (territory-ops); `call-notes-analyzer` (advisory) writes to the shared TODO canvas.

## Bundled brains

Knowledge digests skills read from at runtime, bundled in `shared/brains/` so the package is self-contained:

- `product-brain` — **required** by `sf-product-recommender` (won't run without it)
- `agentforce-brain` — many skills *(raw source decks `.gitignore`d; only digests ship — keeps the bundle small)*
- `se-brain`, plus `manufacturing/architect/proserv/citizen-architect-brain` (all pulled by `call-notes-analyzer`)

## Repository layout

```text
se-skills/
├── README.md                        ← you are here
├── .claude-plugin/marketplace.json  ← marketplace manifest (lists the 6 plugins)
├── plugins/
│   ├── se-account-intelligence/     ← each: .claude-plugin/plugin.json + skills/
│   ├── se-territory-ops/
│   ├── se-bvs/
│   ├── se-dreamforce/
│   ├── se-advisory/
│   └── se-afo-slack/
├── shared/
│   ├── brains/                      ← bundled knowledge digests
│   └── config/se-config.example.json
└── docs/
    ├── INSTALL.md
    ├── DEPENDENCIES.md              ← full per-skill dependency map
    └── PORTABILITY.md               ← the de-personalization punch-list
```

## Roadmap

- [x] Codify **`se-key-contact-review`** — rank an SE's AEs' contacts and apply a **CASL/PIPEDA implied-consent test** (24-month rolling window, LVMs excluded, Slack-corroborated) to flag legitimately-contactable Key Contacts, then stamp the `Key Contact` field (CLI direct-write, or a Slackbot canvas on the read-only MCP path). Reference implementation for de-personalization.
- [x] Migrate the full **`se-account-intelligence`** plugin (13 skills) — scrubbed, config-driven, PII-verified
- [ ] Migrate the remaining five plugins (`se-territory-ops`, `se-bvs`, `se-dreamforce`, `se-advisory`, `se-afo-slack`)
- [ ] Bundle brains with raw sources excluded
- [ ] Write `docs/INSTALL.md`, `docs/DEPENDENCIES.md`, `docs/PORTABILITY.md` (referenced above)
- [ ] Retire deprecated `se-weekly-territory-review` (replaced by `se-ae-territory-brief` + `se-scorecard`)

---

*Authored by David Langan. Internal Salesforce SE tooling — not an official Salesforce product.*
