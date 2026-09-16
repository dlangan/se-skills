# Portability & De-personalization

This suite was authored for **one SE** — a single Org62 alias, one Slack workspace, one Canadian territory. Making it shareable means every personal or org-specific value is **externalized into `~/.claude/se-config.json`** instead of being hardcoded inside a skill, and any embedded customer data is **scrubbed before the skill is published**.

This doc is the de-personalization punch-list: what has to change, where it lives, and how it's resolved. It's also the gate every skill passes through on its way from `~/.claude/skills/` into a plugin here.

---

## The config contract

Skills read `~/.claude/se-config.json` at runtime. Copy the example and fill it in:

```bash
cp shared/config/se-config.example.json ~/.claude/se-config.json
```

| Key | Replaces (hardcoded original) | Used by |
|---|---|---|
| `org_alias` | Author's `--target-org` / Org62 alias | All Org62/`sf` skills |
| `account_root` | Author's local `~/accounts` convention (3 variants existed) | ~10 account skills |
| `slack_team_id` | Author's Slack workspace ID | Slack skills |
| `todo_canvas_id` | Author's personal SE Action Items canvas | todo-tracker, call-notes, hubbl-health360 |
| `se_name` | Author's name in canvas footers / doc curator | Output-producing skills |
| `fiscal_calendar` | Hardcoded FY/quarter dates | territory / scorecard |
| `ae_roster_path` | Path to the SE→AE alignment `.md` | `se-key-contact-review` |
| `ae_roster` | Hardcoded AE names + Org62 User IDs | territory / co-prime / key-contact |
| `specialist_roster` | Hardcoded co-prime specialist AE list | buyer-group-mapper, co-prime |
| `contact_key_contact_field` | Assumed Contact field API name | `se-key-contact-review` |

---

## Punch-list (severity-ranked)

| Severity | Blocker | Affected skills | Resolution | Status |
|---|---|---|---|---|
| 🔴 | `--target-org` hardcoded to author's Org62 | write skills (territory-ops) | → `config.org_alias` | ✅ done |
| 🔴 | Personal Slack canvas ID baked into shared logic | call-notes-analyzer, todo/health skills | → `config.todo_canvas_id` | ✅ done |
| 🔴 | Real customer PII embedded in `SKILL.md` as a "reference log" | df27 profiling skills | Manually scrubbed before publish — every real customer/person name genericized to `[Account]`/`[Stakeholder]` placeholders | ✅ done |
| 🔴 | Real internal Slack channel/canvas IDs left as "reference only" | afo-slack family, promo-collector, promo-matcher | Stripped to placeholder tokens (`<…_CHANNEL_ID>` / `<…_CANVAS_ID>`); word-bounded scan confirms zero `F0…`/`C0…`/`T…` real IDs remain | ✅ done |
| 🟠 | Inconsistent account root (3 conventions) | ~10 skills | → `config.account_root` | ✅ done |
| 🟠 | Hardcoded AE / specialist rosters + colleague Slack IDs | territory / co-prime / AVP | → `config.*_roster`; AVP/RVP identities + source channel/DM IDs genericized | ✅ done |
| 🟡 | Org-instance IDs — Scorecard Metric/Type (`aJC…`/`aJD…`), Dreamforce Campaign (`701…`), Q-Identity Workspace org (`00D…`) | scorecard, territory-org62-sync, deal-coach, df-update, bvs | **Kept** — portable within the same Org62 and not personal data; documented so an installer knows they're author-org-specific and may need swapping | ✅ accepted / documented |
| 🟡 | Slack-native, workspace-scoped skills | `se-afo-slack` family | Only run inside the original Slack workspace | ✅ accepted / documented |
| ✅ | — | `se-key-contact-review` | Fully config-driven; no hardcoded org, roster, or IDs | **done (reference impl)** |

### Kept org-instance IDs (documented, not PII)

These opaque IDs are Org62 *metadata*, not personal or customer data. They're kept verbatim because they identify org objects the skill queries; an installer on a different Org62 swaps them:

| ID pattern | What it is | Where |
|---|---|---|
| `aJC3y…` / `aJD3y…` | Scorecard Metric / Type record IDs | `se-scorecard`, `se-territory-org62-sync`, `se-deal-coach` |
| `701ed…` | Dreamforce Campaign IDs | `df-update` (and df27 registration sync) |
| `00D8c000003ECzNEAW` | Q-Identity Solutions Workspace org | `bvs-discovery-questionnaire` (BVS plug-in point) |

### Known residual gaps (workspace-scoped, flagged for the installer)

Config keys and behaviours that couldn't be fully generalized and are documented in the affected skill's Configuration section:

- **`se-afo-slack` dual-workspace assumption** — the family was authored across two Slack workspaces; canvas/channel targets are placeholder tokens an installer must map to their own workspace. These skills only run inside the Slack workspace they were authored for.
- **AVP/RVP source channels & DMs** — `se-avp-*` / `se-rvp-*` read a leader's Monday post and DM a canvas link; the source channel/DM and leader identity are placeholders, not canonical config keys.
- **GTM-leader names (promo skills)** — no canonical `se-config.json` key exists for a GTM-leader roster; `promo-collector`/`promo-matcher` use `[GTM Leader Name]` placeholders rather than inventing a key.
- **`tech_exec_user_id`** — the default tech-exec on SE opportunity field updates has no canonical key yet; skills that need it use a `<default_tech_exec_user_id>` placeholder pending a config-key decision.

---

## PII scrubbing — read before migrating any skill

The `.gitignore` protects against committing `se-config.json`, the AE roster, and brain raw sources. **It does not and cannot protect against PII written inside a `SKILL.md` body.** Several source skills (notably the df27 family) contain real customer names, deal details, and account data captured as "reference logs" during development.

**Every skill must be manually read and scrubbed before it is added to this repo.** The bar: a skill's `SKILL.md` should contain only *methodology* and *placeholders* — no real customer names, no real deal data, no personal IDs. This repo is **public**.

---

## Migration checklist (per skill)

Use this when moving a skill from `~/.claude/skills/` into a plugin:

- [ ] Read the whole `SKILL.md` — remove any real customer/PII/reference-log content.
- [ ] Replace every hardcoded org alias, canvas ID, team ID, roster, and path with a `config.*` lookup.
- [ ] Document any org-instance IDs that stay (🟡 accepted) so an installer knows they're author-org-specific.
- [ ] Confirm brain dependencies ship under `shared/brains/` (digests only).
- [ ] Update `DEPENDENCIES.md` with the skill's skill-deps and integrations.
- [ ] Flip its punch-list row to ✅.

`se-key-contact-review` is the reference implementation — every value it needs comes from `se-config.json`, and its `SKILL.md` carries methodology only.
