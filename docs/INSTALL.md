# Installing SE Skills

This is a Claude Code **plugin marketplace**. You add the marketplace once, install the plugin families you want, and set up a single config file. Skills are then invoked as `/<plugin>:<skill>` or triggered conversationally.

> **Current state:** only `se-key-contact-review` (in `se-account-intelligence`) is migrated and shippable. Installing other plugins today gives you the manifest but few/no skills until migration completes. See the [roadmap](../README.md#roadmap).

---

## 1. Prerequisites

These skills orchestrate external systems through **MCP servers** and the **Salesforce CLI**. Install/authenticate only what the plugins you want require.

| Dependency | Needed by | How to get it |
|---|---|---|
| **Org62 SOQL MCP** (`salesforce-org62`) | most skills | Read-only SOQL access to Org62. Configure as an MCP server in Claude Code. |
| **Slack MCP** | most skills | Canvas creation + search/read. The default deliverable format. |
| **Salesforce CLI (`sf`)** | write skills (territory-ops; `se-key-contact-review` write-back) | `npm i -g @salesforce/cli`, then `sf org login web` against **your** Org62. |
| **Google Workspace MCP** | `se-bvs`, `se-dreamforce`, `call-notes-analyzer` | Docs/Sheets/Slides output. |
| **Bundled brains** | account-intelligence, advisory | Ship in `shared/brains/` — no external fetch. |

### Connection modes for Org62 (matters for write-back)

Some skills **write** to Org62 (e.g. `se-key-contact-review` stamps a `Key Contact` field). How the write happens depends on how you're connected:

- **Salesforce CLI present** (`sf` authenticated to `org_alias`) → skills write directly via `sf data`.
- **MCP only** (read-only Org62 MCP) → skills cannot write; they produce a **Slack Canvas** with instructions for Slackbot to apply the update.

You don't configure this explicitly — the skill detects what's available — but if you want direct writes, install and authenticate the `sf` CLI.

---

## 2. Add the marketplace

```text
/plugin marketplace add dlangan/se-skills
```

## 3. Install the families you want

Repeat per plugin — install only what you need:

```text
/plugin install se-account-intelligence@se-skills
/plugin install se-territory-ops@se-skills
/plugin install se-bvs@se-skills
/plugin install se-dreamforce@se-skills
/plugin install se-advisory@se-skills
/plugin install se-afo-slack@se-skills
```

## 4. Configure (one time)

Copy the example config into your Claude Code home and fill in your own values:

```bash
cp shared/config/se-config.example.json ~/.claude/se-config.json
```

Then edit `~/.claude/se-config.json`. See [PORTABILITY.md](PORTABILITY.md) for what each key controls. Key ones:

| Key | Purpose |
|---|---|
| `org_alias` | Your `sf` CLI / Org62 target (never inherit the author's). |
| `account_root` | Your local account-workspace folder. |
| `slack_team_id` | Your Slack workspace team ID. |
| `se_name` | Your name (canvas footers, doc curator). |
| `ae_roster_path` | Path to your AE-alignment `.md` file (used by `se-key-contact-review`). |
| `contact_key_contact_field` | Contact field API name to stamp for Key Contacts (default `Key_Contact__c`). |

> **Never commit your `se-config.json`.** It lives in `~/.claude/`, not the repo, and is `.gitignore`d here as a backstop.

---

## 5. Verify

```text
/plugin
```

You should see the installed plugins listed. Try a migrated skill:

```text
Run a key contact review for <AE name>
```

If it can't find an AE roster and you didn't name an AE, `se-key-contact-review` will offer to create the roster file or take a one-off AE name — that's expected first-run behavior.

---

## Troubleshooting

- **A skill says a brain file is missing** → confirm the brain shipped under `shared/brains/`; a few skills won't run without their brain (e.g. `sf-product-recommender` needs `product-brain`).
- **A write-back didn't happen** → you're likely on the read-only MCP path; look for the Slack Canvas it produced instead, and confirm `sf` is authenticated if you want direct writes.
- **404 on `docs/` links or `shared/`** → some referenced files are still being written; check the [roadmap](../README.md#roadmap).
