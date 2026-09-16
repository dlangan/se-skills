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
| 🔴 | `--target-org` hardcoded to author's Org62 | 4 write skills | → `config.org_alias` | pending migration |
| 🔴 | Personal Slack canvas ID baked into shared logic | todo-tracker, hubbl-health360, call-notes | → `config.todo_canvas_id` | pending migration |
| 🔴 | Real customer PII embedded in `SKILL.md` as a "reference log" | df27 profiling skills | **Manual scrub before publish** — `.gitignore` will NOT catch this | pending migration |
| 🟠 | Inconsistent account root (3 conventions) | ~10 skills | → `config.account_root` | pending migration |
| 🟠 | Hardcoded AE / specialist rosters + Slack IDs | territory / co-prime / AVP | → `config.*_roster` | pending migration |
| 🟡 | Org-instance IDs (Scorecard metrics, UsageType, Campaign) | scorecard, org-health, df | Portable within the same Org62; documented, not externalized | accepted |
| 🟡 | Slack-native, workspace-scoped skills | `se-afo-slack` family | Only run inside the original Slack workspace | accepted / documented |
| ✅ | — | `se-key-contact-review` | Fully config-driven; no hardcoded org, roster, or IDs | **done** |

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
