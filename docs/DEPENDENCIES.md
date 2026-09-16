# Dependencies

Two kinds of dependency matter in this marketplace:

1. **Skill → skill** (orchestration): some skills invoke a tree of others. Grouping into plugins follows these trees — **install the whole plugin, not individual skills.**
2. **Skill → integration** (MCP / CLI / brain): what external system a skill needs to run.

> This map is authoritative for the migrated skills and reflects the known orchestration structure for the rest. Per-skill entries fill in as each skill is migrated. The one fully-migrated skill today is **`se-key-contact-review`**.

---

## Orchestrator trees (skill → skill)

Orchestrators call a sequence of sub-skills; the whole tree lives in one plugin.

**se-account-intelligence**
- `se-full-account-package` → `se-entitlement-decoder` · `se-case-insights` · `se-buyer-group-mapper` · `se-competitive-intel` · `se-account-research` · `se-360-builder` · `se-northstar-pov` · `se-connected-vision`
- `se-connected-vision` / `se-northstar-pov` → `se-account-research` · `se-exec-challenge` · `se-use-case-advisor` · `pccwop-narrative-builder`
- `se-key-contact-review` → `se-buyer-group-mapper` *(reuses its title → buyer-role mapping)* ✅ migrated

**se-bvs**
- `bvs-advisor` → `bvs-discovery-questionnaire` · `bvs-executive-value-map` · `bvs-value-tree-benchmarking`

**se-dreamforce**
- `df27-account-onboard` → `df27-account-profile` · `df27-attendee-profile` · `df27-recommendation-doc`
- `df-update` (syncs the session catalog; feeds recommendations)

**se-territory-ops**
- `se-ae-territory-brief`, `se-scorecard`, `se-territory-org62-sync`, `se-avp-*`, `se-co-prime-*` (deal-review + Org62 hygiene family)

---

## Cross-plugin couplings

These couplings cross plugin boundaries — worth knowing if you install selectively:

| Consumer | Depends on | Nature |
|---|---|---|
| `se-bvs` skills | Connected Vision *voice* file from `se-account-intelligence` | Reads narrative voice; install account-intelligence too for best results. |
| `se-event-recommender` (territory-ops) | `df-update` (dreamforce) | Session catalog feeds event recommendations. |
| `call-notes-analyzer` (advisory) | shared TODO canvas (`config.todo_canvas_id`) | Writes action items to your personal canvas. |
| `se-key-contact-review` (account-intelligence) | `se-buyer-group-mapper` (same plugin) | Title → buyer-role mapping. |

---

## Integration matrix (skill → external system)

What each plugin family needs. `●` = required by most skills in the family, `○` = required by some.

| Plugin | Org62 MCP | Slack MCP | `sf` CLI | Google Workspace | Brains |
|---|---|---|---|---|---|
| se-account-intelligence | ● | ● | ○ (write-back) | ○ | ● |
| se-territory-ops | ● | ● | ● (writes) | | |
| se-bvs | ● | ● | | ● | |
| se-dreamforce | ● | ○ | | ● | |
| se-advisory | ● | ● | | ○ | ● |
| se-afo-slack | | ● (specific workspace) | | | |

**Notable single-skill dependencies:**
- `sf-product-recommender` **requires** `product-brain` — won't run without it.
- `call-notes-analyzer` pulls several brains (`se-brain`, `manufacturing`, `architect`, `proserv`, `citizen-architect-brain`).
- `se-afo-slack` family is **workspace-scoped** — only runs inside the Slack workspace it was authored for.

---

## `se-key-contact-review` — full dependency detail (reference for migration)

The migrated skill, documented in full as the template for how others should be:

- **Skill deps:** `se-buyer-group-mapper` (buyer-role mapping).
- **Integrations:** Org62 MCP (read: Account/Contact/Task/Event/OpportunityContactRole) **required**; Slack MCP (engagement corroboration + write-back canvas) **required**; `sf` CLI **optional** (enables direct field write instead of the canvas path).
- **Config keys:** `org_alias`, `ae_roster_path`, `se_name`, `slack_team_id`, `contact_key_contact_field`.
- **Brains:** none.
