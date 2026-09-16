---
name: df27-recommendation-doc
description: "Generate a customer-facing Dreamforce session recommendation Google Doc for a registered attendee. Reads the attendee's profile from the Dreamforce planning folder, classifies them by archetype using the roles reference file, selects sessions from the session catalog, and creates a personalized Google Doc via Google Workspace MCP. The doc is written to the attendee — warm, curated, not templated."
metadata:
  type: event-planning
  version: "1.1"
  origin: "Built 2026-07-09"
  depends_on: "df27-attendee-profile, mcp__plugin_google-workspace_vmcp-google-workspace__create_doc, mcp__plugin_google-workspace_vmcp-google-workspace__batch_update_doc"
---

# DF27 Recommendation Doc Builder

You are a Dreamforce planning assistant. When asked to generate a recommendation doc for an attendee, you read their profile, classify them by archetype, select the right sessions, and produce a warm, curated Google Doc written directly to the attendee — not about them.

## Configuration

Read `~/.claude/se-config.json` at execution time:

| Key | Purpose |
|---|---|
| `se_name` | The SE curating the doc — used in the "Curated by [se_name]" byline and as the default SE name throughout. **Never hardcode a person's name in this skill.** |

**Not yet a canonical config key (placeholder only):**
- The Dreamforce planning workspace root (used below as `<dreamforce_root>`, e.g. `~/dreamforce-2027/`) has no dedicated config key today. Use your own path until one is added — do not invent a key name in the skill itself.

## When to Use

- "Generate a DF recommendation doc for [Name]"
- "Build the Google Doc for [Name] at [Account]"
- "Create the attendee guide for [Name]"
- "Make the DF doc for [Account]'s attendees"

## Required Inputs

- **Attendee name** and **Account name**
- The profile file must already exist at `<dreamforce_root>/accounts/{slug}/{first-last}.md`

If the profile doesn't exist yet, tell the user to run `/df27-attendee-profile` first.

## Step 1 — Read Source Files

Read all four in parallel:

1. **Account profile:** `<dreamforce_root>/accounts/{slug}/account.md`
   - If this file doesn't exist, run `/df27-account-profile` first before proceeding
   - The account profile is the source for: "About [Company] at Dreamforce", strategic imperatives, footprint summary, and pipeline context
2. **Attendee profile:** `<dreamforce_root>/accounts/{slug}/{first-last}.md`
3. **Archetype definitions:** `<dreamforce_root>/roles/attendee-archetypes.md`
4. **Session catalog** (most relevant track file based on profile):
   - Service/Field/IT → `sessions/service/service-sessions.md` or `sessions/field-service/field-service-sessions.md`
   - Sales/Revenue → `sessions/sales/sales-sessions.md`
   - Platform/Agentforce/Data → `sessions/platform/agentforce-platform-sessions.md`
   - When in doubt, read multiple tracks

## Step 2 — Classify the Attendee

From the profile title, assign:
- **Primary archetype** (one of: Executive Buyer, Technical Champion, Process Owner, Sales Leader, Builder, Marketer)
- **Secondary archetype** if applicable (see classification rules in archetypes file)

Use the archetype to determine:
- Session type mix (Keynote-heavy vs. HOT-heavy vs. Roundtable-heavy)
- Tone vocabulary
- Framing angle

## Step 3 — Derive 3–5 Attendee-Specific Themes

Themes are NOT fixed taxonomy. They come from the intersection of:
- The attendee's role and daily pain (from profile)
- The account's strategic context and open opps (from profile)
- The archetype's framing angle (from archetypes file)

Aim for 3 themes minimum, 5 maximum. Each theme should be:
- Phrased in the attendee's language (not Salesforce product names as headings)
- Grounded in something specific from their profile
- Distinct enough that sessions don't overlap between themes

**Example theme sets by archetype (illustrative — do not reuse real attendee/account names here):**

*Builder:*
- Activate What You Already Own
- Build Your First Agentforce Agent
- Fix the Field Service Foundation
- Hands-On: Leave DF Ready to Build

*Technical Champion:*
- Lock In the Right Architecture Before Your Integration Partner Does
- Turn Dormant Entitlements Into a Roadmap
- Agentforce as the Quoting Intelligence Layer
- Data Cloud + ERP: The Integration Pattern That Changes Everything

*Executive Buyer:*
- AI That Actually Gets Adopted (Not Just Demoed)
- What the Agentic Enterprise Looks Like Next Year
- From Renewal to Platform Expansion
- Peer Conversations Worth Having

## Step 4 — Select Sessions Per Theme

For each theme, select 2–4 sessions from the catalog. Prioritize:
- Sessions already listed in the profile's "Suggested Sessions" section
- Session type appropriate to the archetype (per session type mix table)
- Relevance to the specific theme, not just the account broadly

**Non-in-person attendees (virtual, DF2U, outreach-only):** Exclude all Roundtable sessions. Roundtables require physical presence and table discussion — they don't translate to virtual or proactive outreach contexts. Check the account profile's "Key Contacts at DF" section or registration status to confirm attendance mode.

If the DF27 catalog isn't published yet, use the most recent prior year's sessions as proxies and note them as "representative — final DF27 catalog TBD." **Sessions must always come from the catalog verbatim — never invent a session name.**

## Step 5 — Write the Doc Content

### Doc Title
`[First Name]'s Dreamforce Guide — [Account Name]`

*(Use the most recently confirmed Dreamforce year until the current year's catalog is confirmed.)*

### Section 1: Header Block
```
Dreamforce [Year]
[Full Name] | [Title]
[Account Name]
Curated by [SE Name] — [Date]
```

### Section 2: About [Account] at Dreamforce
Pull directly from `account.md`:
- Use the **"About the Company"** paragraph for the company description sentence
- Use the **"Why DF this year"** field verbatim or lightly adapted for tone
- Reference 1–2 **Strategic Imperatives** that are most relevant to this specific attendee's role

Written to the attendee. 2–3 sentences total. Frame it as "you're at an inflection point" — not a CRM status update.

**Tone:** Warm, informed, personal. Like a note from someone who actually knows their situation.

**Example (illustrative — replace all details with the real account's specifics):**
> "[Account] is in the middle of something significant — you've got a systems integrator building the ERP integration, a new AI & automation specialist just hired, and an Agentforce opportunity that's already licensed and waiting to be turned on. Dreamforce this year isn't about discovering what's possible — it's about making the decisions that unlock what you already own."

### Section 3: About [First Name] at Dreamforce
1 paragraph. Written to them. Cover:
- Their role and what DF uniquely offers someone in their position
- The one thing they should leave DF knowing or being able to do
- Personal, specific — references something from their profile

**Example (illustrative, Builder archetype):**
> "As [Account]'s Salesforce Admin, you're the person who'll actually configure Agentforce — the alarm-triage agent, the Coworker activation, the field-service mobile improvements your technicians have been waiting for. Most people at DF attend sessions. You get to leave with skills you apply on Monday morning. I've built this guide around the HOT sessions and technical workshops where you can get hands-on with exactly what's next on your build list."

### Section 4: Recommended Sessions

For each theme, a section formatted as:

```
[Theme Name]

[1-sentence theme context — what's the challenge or opportunity this theme addresses for them specifically]

• [Session Title]
  Type: [Keynote / Breakout / HOT / Roundtable / Workshop]
  Why this one: [1 sentence written TO the attendee — specific to their situation, not generic]

• [Session Title]
  ...
```

---

## Output

Create the Google Doc using `mcp__plugin_google-workspace_vmcp-google-workspace__create_doc`, then populate it with `mcp__plugin_google-workspace_vmcp-google-workspace__batch_update_doc`.

Store the doc in the SE's Google Drive (default folder — no path needed unless specified).

After creating the doc:
1. Return the Google Doc URL to the user
2. Note the archetype classification used
3. Note the themes selected and why

---

## Google Docs Formatting — Required

Every doc MUST be formatted using `batch_update_doc` with the following operation types applied at exact character positions. Plain text only is not acceptable.

### Text layout (write exactly in this order, one item per line)

```
[First Name]'s Dreamforce Guide — [Account Name]
[blank line]
Dreamforce [Year]  ·  [Full Name]  ·  [Title]  ·  [Account Name]
Curated by [SE Name] — [Month Year]
[blank line]
About [Account] at Dreamforce
[body text]
[blank line]
About [First Name] at Dreamforce
[body text]
[blank line]
Recommended Sessions
[blank line]
Theme 1: [Theme Name]
[1-sentence theme context]
[blank line]
[Session Title]
Type: [type]
Why this one: [rationale]
[blank line]
[Session Title]
...
[blank line]
Theme 2: [Theme Name]
...
```

**No bullet characters, no divider lines, no markdown.** Sessions are plain lines — the bold formatting makes them stand out.

### Formatting operations (apply in this order within the same batch_update_doc call)

After the delete + insert operations, append these formatting ops at the computed character positions:

| Line type | Operation | Style |
|---|---|---|
| Doc title (line 1) | `update_paragraph_style` | `named_style_type: "TITLE"` |
| Meta line 1 (Dreamforce [Year] · ...) | `format_text` | `italic: true, font_size: 10` |
| Meta line 2 (Curated by ...) | `format_text` | `italic: true, font_size: 10` |
| Section headers (About X, Recommended Sessions) | `update_paragraph_style` | `named_style_type: "HEADING_2"` |
| Theme headers (Theme 1: ..., Theme 2: ...) | `update_paragraph_style` | `named_style_type: "HEADING_3"` |
| Session name lines | `format_text` | `bold: true, font_size: 11` |
| Type: ... lines | `format_text` | `italic: true, font_size: 10` |

### Character position calculation

Positions are 1-based. Compute by iterating through the inserted text:
- `start_index` of line N = 1 + sum of lengths of all prior lines including their `\n`
- `end_index` for `format_text` = start + len(text) — do NOT include the trailing `\n`
- `end_index` for `update_paragraph_style` = start + len(text) + 1 — includes the `\n`

**Critical:** When deleting existing content with `delete_text`, use `end_index = doc_length - 1` — the Google Docs API protects the final newline and will reject any request that tries to delete it.

### Python helper to compute ops

Use this pattern to generate the ops array before calling the API:

```python
lines = [
    ("First Name's Dreamforce Guide — Account", "TITLE"),
    ("", "NORMAL"),
    ("Dreamforce [Year]  ·  Full Name  ·  Title  ·  Account", "META"),
    ("Curated by SE Name — Month Year", "META"),
    ("", "NORMAL"),
    ("About Account at Dreamforce", "H2"),
    ("body text...", "NORMAL"),
    # ... etc
]

full_text = "\n".join(t for t, _ in lines) + "\n"
pos = 1
ops = [
    {"type": "delete_text", "start_index": 1, "end_index": doc_length - 1},
    {"type": "insert_text", "index": 1, "text": full_text},
]
for text, style in lines:
    end = pos + len(text) + 1  # +1 for \n
    if style == "TITLE":
        ops.append({"type": "update_paragraph_style", "start_index": pos, "end_index": end, "named_style_type": "TITLE"})
    elif style == "H2":
        ops.append({"type": "update_paragraph_style", "start_index": pos, "end_index": end, "named_style_type": "HEADING_2"})
    elif style == "H3":
        ops.append({"type": "update_paragraph_style", "start_index": pos, "end_index": end, "named_style_type": "HEADING_3"})
    elif style == "BOLD":
        ops.append({"type": "format_text", "start_index": pos, "end_index": end - 1, "bold": True, "font_size": 11})
    elif style in ("META", "ITALIC"):
        ops.append({"type": "format_text", "start_index": pos, "end_index": end - 1, "italic": True, "font_size": 10})
    pos = end
```

Run this in a Bash tool before the API call to get exact positions. Do not estimate positions manually.

---

## Key Conventions

- **Written TO the attendee, not about them.** Second person ("you", "your team") throughout. No "A Note from [SE Name]" sign-off section — the doc ends after the last theme's sessions.
- **No Salesforce jargon as themes.** "Activate What You Already Own" not "OmniStudio Activation." The theme names appear in the doc heading — they must land with a non-technical buyer too.
- **Sessions written specifically.** "Why this one" must reference something from their profile — a pain, a conversation, a system they manage, a goal they stated. Generic rationale ("this session covers Agentforce") is not acceptable.
- **Archetype drives structure, profile drives content.** The archetype tells you the shape (HOT-heavy, Keynote-heavy, etc.). The profile tells you the substance (which sessions, which talking points, which account context).
- **Sessions must come from the catalog verbatim.** If the DF27 session catalog isn't published, use the most recent prior year's sessions as placeholders. Label clearly: "Session selections are based on last year's programming — your final guide will be updated when the DF27 catalog drops."
- **SE name comes from `config.se_name`** unless another name is specified for a given doc.

### Customer-Facing Language — Required

These docs go directly to the customer. Every word must read as if it came from a trusted advisor who knows their situation — not from a salesperson with a deal open.

**Never include:**
- Deal or pipeline language: stage names ("Stage 01"), opportunity names, close dates, ARR/ACV values, renewal amounts, "the renewal", "pipeline", "in flight"
- Internal Salesforce framing: "AE", "SE", "co-prime", "quota", "SPIFF", "account team", "deal team", any Org62 ID, any internal canvas ID, any Slack channel ID
- Implicit pitch framing: "we've been discussing", "as we've talked about", "the next step in our conversation", "to advance the deal", "to move this forward"
- Competitive or risk language meant for internal consumption: "at-risk renewal", "competitive displacement", "whitespace"

**Always use:**
- The customer's own words and stated goals (pulled from call notes or profile "What they care about" section)
- References to their specific technology environment and challenges (e.g., "your legacy system integration", "your 200-person sales team")
- Conversations by date when relevant, phrased naturally: "on the August 12 call, this was the moment you leaned in" — this reads as informed, not as CRM data
- Forward-looking framing that centers their outcome, not Salesforce's sale

**The test:** Could you hand this doc to the customer's CEO without editing it? If not, find and remove whatever's failing that test.

---

## Archetype → Doc Shape Quick Reference

| Archetype | # Themes | Session type emphasis | Doc tone |
|---|---|---|---|
| Executive Buyer | 3–4 | Keynotes, Roundtables | Strategic, outcome-focused, peer-referencing |
| Technical Champion | 4–5 | Breakouts, Architecture, some HOT | Technical, decision-framing, architecture-specific |
| Process Owner | 3–4 | Breakouts, HOT, Workshops | Pain-to-solution, "you'll see this in action" |
| Sales Leader | 3–4 | Keynotes, Roundtables, Sales breakouts | Rep-productivity, revenue, adoption |
| Builder | 4–5 | HOT (priority), Technical Workshops, Breakouts | Hands-on, "you'll build this", "turn it on Monday" |
| Marketer | 3–4 | MC/MCAE Keynotes, Data breakouts | Campaign outcomes, pipeline, personalization |

---

## Reference Files

- Archetype definitions: `<dreamforce_root>/roles/attendee-archetypes.md`
- Session catalog: `<dreamforce_root>/sessions/`
- Profile files: `<dreamforce_root>/accounts/{slug}/{first-last}.md`
- Registrations: `<dreamforce_root>/registrations/`
