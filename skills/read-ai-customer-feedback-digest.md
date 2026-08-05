---
name: customer-feedback-digest
description: Pulls sales-style meetings (discovery, demo, customer) from the Read AI connector for a recent window, extracts structured product feedback (user needs, product quality signals, feature gaps, general feedback), optionally tags each item against a user-supplied persona framework and a coarse vertical taxonomy, and publishes a weekly digest to a configurable set of destinations — a dated markdown snapshot in the working directory, a rolling wiki page (Confluence, Notion, SharePoint, or OneNote), and a chat-channel post (Slack or Microsoft Teams). The wiki and chat outputs are pluggable adapters, so teams on either Atlassian/Notion/Slack stacks or Microsoft 365 stacks can use the same skill. Reads the prior 8 weeks of snapshots and a running trends file to surface recurring themes and emerging signals. Use this skill whenever the user wants to summarize sales discovery or demo calls for the product team, set up a weekly customer-feedback rollup, identify trends in prospect feedback, or asks "what are prospects saying about us this week". On first run, the skill launches an interactive setup wizard that detects the user's connected MCPs and walks them through configuration. The skill is also designed to run unattended on a schedule — when invoked from a scheduled task, it runs end-to-end in auto-publish mode without prompting.
---

# Customer Feedback Digest

## What this skill does

Turns a window of sales-style meetings into a structured, actionable feedback digest for the Product team. Every run does five things in order:

1. Fetches meetings from the Read AI connector for the requested window (default: past 7 days) whose title matches the configured patterns (typically discovery or demo calls).
2. Pulls each call's summary, action items, key questions, topics, and (when needed) transcript.
3. Extracts product feedback into four categories — user needs, product quality, feature gaps, general feedback — and optionally tags each item with persona and vertical.
4. Reads the prior `N` weeks of snapshots and the running trends file, then synthesizes what's recurring vs new.
5. Writes outputs to the destinations the user has enabled: a dated markdown snapshot in the working directory, a single rolling wiki page (one of Confluence, Notion, SharePoint, or OneNote), and a short chat digest (Slack or Microsoft Teams).

The skill is built to run unattended on a schedule. It is also safe to run interactively — in that mode it shows the local snapshot first and asks before posting to chat or updating the wiki.

## Required connectors

This skill is a connector-integration demo. It requires at minimum:

- **Read AI MCP** (required) — the source of meeting metadata and transcripts. The skill cannot run without it.

Optional, enabled per-output role:

- **Wiki adapter** (one of):
  - **Atlassian / Confluence MCP** — implemented day one.
  - **Notion MCP** — implemented day one.
  - **Microsoft 365 connector → SharePoint Pages** — documented adapter; writes require a future write-capable M365 connector or a third-party M365 MCP.
  - **Microsoft 365 connector → OneNote** — documented adapter; same caveat as SharePoint.
  - See `references/wiki_adapters.md`.
- **Chat adapter** (one of):
  - **Slack MCP** — implemented day one.
  - **Microsoft 365 connector → Teams** — documented adapter; same write caveat as the M365 wiki adapters.
  - See `references/chat_adapters.md`.

The wiki and chat roles each accept exactly one adapter per run, configured in `outputs.wiki.target` and `outputs.chat.target`.

The first-run wizard (Step 0) detects which of these are connected and adapts the available output choices accordingly. See `references/connector_setup.md` for detection patterns and setup pointers.

## Configuration

The skill keeps its persistent settings in a config file in the current working directory:

```
./.feedback-digest-config.json
```

Expected shape:

```json
{
  "context": {
    "product_name": "<your product>",
    "product_one_liner": "<short description used in extraction prompt>",
    "audience_team": "Product",
    "feedback_focus": "<what kinds of feedback to surface — e.g. integrations, pricing, UX gaps>"
  },
  "ingestion": {
    "source": "read_ai",
    "title_patterns": ["disco", "demo"],
    "exclude_internal_only": true,
    "internal_domains": ["yourcompany.com"]
  },
  "tagging": {
    "personas_file": null,
    "verticals": null
  },
  "outputs": {
    "local": {
      "enabled": true,
      "snapshots_dir": "snapshots",
      "trends_file": "rolling-trends.md"
    },
    "wiki": {
      "target": null,
      "confluence": {
        "space_key": null,
        "page_id": null,
        "page_title": null
      },
      "notion": {
        "parent_page_id": null,
        "page_id": null,
        "page_title": null
      },
      "sharepoint": {
        "site_id": null,
        "page_id": null,
        "page_title": null
      },
      "onenote": {
        "section_id": null,
        "page_id": null,
        "page_title": null
      }
    },
    "chat": {
      "target": null,
      "slack": {
        "channel": null
      },
      "teams": {
        "team_id": null,
        "channel_id": null
      }
    }
  },
  "trend_window_weeks": 8
}
```

Field notes:

- `context.*` is fed into the extraction prompt so the LLM understands what product is being evaluated and what kinds of feedback matter to this team.
- `ingestion.title_patterns` are matched case-insensitively at the start of the meeting title — e.g., `"disco"` matches `Disco - Acme - Q3 sync` and `Disco call with Beta Inc`. Keep matching loose at the front of the string to avoid false positives from titles like "Discovery Channel kickoff".
- `tagging.personas_file` is a path (relative to the config file) to a markdown file the user wrote describing their personas. If null, persona tagging is skipped. See `references/personas_example.md` for the expected shape.
- `tagging.verticals`, if set, replaces the default vertical taxonomy. The defaults are listed in `references/extraction_schema.md`.
- `outputs.wiki.target` is one of `"confluence" | "notion" | "sharepoint" | "onenote"` (or null to disable the wiki output entirely). Pick one — the skill writes to exactly one wiki destination per run. See `references/wiki_adapters.md` for what each target needs in its config block.
- `outputs.chat.target` is one of `"slack" | "teams"` (or null to disable the chat post entirely). Pick one. See `references/chat_adapters.md` for adapter-specific notes.
- The SharePoint, OneNote, and Teams adapters depend on the Microsoft 365 connector. As of this writing, the Anthropic-distributed M365 connector is read-only; the M365 write paths require either a future write-capable M365 connector or a third-party M365 MCP. The skill ships the adapter logic anyway so the moment writes become available, the configuration already exists.

## Inputs

The skill accepts these arguments (all optional):

| Arg | Purpose | Default |
| --- | --- | --- |
| `--start` | Window start (ISO 8601, inclusive) | 7 days before now |
| `--end` | Window end (ISO 8601, inclusive) | now |
| `--mode` | `interactive` or `auto` | `interactive` if a human is in the loop, `auto` if invoked from a scheduled task |
| `--dry-run` | Skip wiki and chat writes | false |
| `--config` | Path to config file | `./.feedback-digest-config.json` |
| `--reconfigure` | Re-run the first-run wizard against an existing config | false |

When invoked from a scheduled task, default to `--mode auto`.

## Step 0 — First-run setup wizard

Fires when the config file is missing, when required fields are unset, or when the user passes `--reconfigure`. Skipped in `--mode auto`: scheduled runs must fail loudly rather than prompt.

Walk the user through the following sections. Ask one section at a time, confirm answers, then move on.

### 0.1 — Greet and explain

Tell the user what the skill does in two sentences: it pulls meetings from Read AI, extracts product feedback, and publishes a weekly digest. Note that the setup will take about 5 minutes and will write a config file to the current directory.

### 0.2 — Connector check

Inspect the available tools in the current session. For each of the following, report `connected` or `missing` (see `references/connector_setup.md` for detection patterns):

- Read AI MCP (required).
- Wiki MCPs: Atlassian / Confluence, Notion, Microsoft 365 (covers SharePoint Pages and OneNote).
- Chat MCPs: Slack, Microsoft 365 (covers Teams).

The M365 connector is a single connector that simultaneously enables the SharePoint, OneNote, and Teams adapters — detection treats one M365 install as enabling all three options in the wizard.

If Read AI is missing, stop the wizard. Show the user the setup pointer from `references/connector_setup.md` and ask them to connect Read AI before re-running the skill.

If any optional MCPs are missing, note them — the user can still pick an output that uses a missing MCP, but the matching `outputs.wiki.target` or `outputs.chat.target` will be left null until the MCP appears.

### 0.3 — Context

Ask for the product context that drives extraction. Suggested defaults are placeholders; encourage the user to write something concrete.

- `context.product_name` — name of the product the feedback is about.
- `context.product_one_liner` — short description (one sentence).
- `context.audience_team` — who the digest is for (default: "Product").
- `context.feedback_focus` — free text describing the kinds of feedback most important to this team. This goes into the extraction prompt.

### 0.4 — Ingestion

- `ingestion.title_patterns` — title prefixes to match. Default `["disco", "demo"]`. Confirm or change.
- `ingestion.internal_domains` — list of email domains belonging to the user's own organization. Meetings where all attendees match these domains are dropped as internal.

### 0.5 — Tagging

Ask: "Do you want persona tagging on extracted items?"

- If **no**: leave `tagging.personas_file: null` and move on.
- If **yes**: walk the user through copying `references/personas_example.md` to a path of their choice (suggest `./personas.md`), adapting it to their org, then set `tagging.personas_file` to that path. Don't try to write their personas for them — that's the user's call and the file is plain markdown.

Optionally ask whether they want to override the default vertical taxonomy. Most users won't.

### 0.6 — Outputs

Local markdown is always on. The other two output roles — wiki and chat — each accept exactly one adapter.

**Wiki role** (`outputs.wiki.target`). Ask which wiki, if any, to enable. If multiple compatible MCPs are connected, ask the user to pick one. Then gather adapter-specific config:

- **Confluence** — `space_key` and either `page_id` (existing page to update) or `page_title` (create new page on first run).
- **Notion** — `parent_page_id` and `page_title` for new-page creation, or `page_id` to update an existing page.
- **SharePoint** — `site_id` (resolvable via `/sites?search=...` against the M365 connector) and `page_title` for create, or `page_id` to update.
- **OneNote** — `section_id` (resolvable via `/me/onenote/sections`) and `page_title` for create, or `page_id` to update.

If the user picks SharePoint or OneNote, warn that writes depend on M365 connector write capability (currently read-only) and persist the config anyway. The wiki target will be ready the moment a write-capable M365 connector or third-party M365 MCP is wired up.

**Chat role** (`outputs.chat.target`). Ask which chat target, if any, to enable.

- **Slack** — `channel` (channel name like `#product` or a channel ID). Confirm before saving.
- **Teams** — `team_id` and `channel_id` (both Graph IDs the user can find via `/me/joinedTeams` and `/teams/{id}/channels`).

If the user picks Teams, warn about M365 writes as above.

Wiki and chat targets are each a single choice. If the user wants more than one wiki or chat output, explain that the skill writes to one per run and ask them to pick. (Adding a second target later is a future enhancement; see the "Adding a fifth/third adapter" sections in the adapter files.)

### 0.7 — Write config and offer smoke test

Write `./.feedback-digest-config.json`. Then offer to run a dry-run against the past 7 days as a smoke test — this validates Read AI ingestion and the extraction pipeline without touching chat or the wiki.

## Workflow (after setup is complete)

### Step 1 — Fetch matching meetings

Use the Read AI MCP's `list_meetings` tool with `start_datetime_gte` and `start_datetime_lte` set to the window. The tool caps `limit` at 10 and returns `has_more: true` with cursor-based pagination, so loop until the page is empty or `has_more` is false. The cursor is the `id` of the last meeting in the previous page.

For each page, keep meetings whose lowercased `title` starts with any of `ingestion.title_patterns`.

If `ingestion.exclude_internal_only` is true, drop meetings where every attendee's email matches one of `ingestion.internal_domains`. These are internal practice or training calls and won't have prospect feedback.

If the resulting list is empty, write a snapshot that says "No matching calls in window" and stop. Do not produce a chat/wiki update for an empty week — silence is signal.

### Step 2 — Pull rich content per meeting (tiered fetch)

For each kept meeting, fetch in two tiers. The structured layers (summary, action_items, key_questions, topics) are dense enough to support extraction in most cases, and full transcripts are large (often >100k characters per call). Fetching transcripts you don't need wastes API budget and context.

**Tier 1 — structured layers.** Call `get_meeting_by_id` with `expand: ["summary", "action_items", "key_questions", "topics"]`. This is your default. Skip `metrics` and `recording_download` — they don't carry product feedback signal.

**Tier 2 — transcript fallback.** Re-fetch with `expand: ["transcript"]` only when one of these is true for a specific call:
- A summary line or topic clearly mentions a product feedback signal but the wording is too compressed to be useful as a verbatim quote (e.g., "discussed integration concerns" without specifics).
- An action item references prospect frustration or a specific product gap, but the source moment isn't otherwise captured.
- Persona inference (if enabled) is genuinely unclear from titles + structured layers and you need behavioral signal from the conversation.

Process meetings one at a time when fetching transcripts; loading multiple in parallel risks context overflow.

**When the source isn't transcript, label it.** Each extracted item carries a `quote_source` field naming which layer the quote came from: `transcript`, `summary`, `topics`, `key_questions`, or `action_items`. This matters because the structured layers are themselves AI-generated — a "quote" from `summary` is the meeting tool's synthesis, not the speaker's literal words. The Product team should know whether to trust the wording verbatim. See `references/extraction_schema.md`.

If the transcript is unavailable for a meeting (very short call, upload failed), proceed with the structured layers and set `quote_source` accordingly. Mark `transcript_unavailable: true` on the meeting's run-log entry so the next week's run knows not to retry.

### Step 3 — Extract feedback into the four categories

For every meeting, walk the transcript and summary and produce structured feedback items. The schema and worked examples are in `references/extraction_schema.md` — read that file before extracting if you're unsure how to fill any field.

Use `context.product_name`, `context.product_one_liner`, and `context.feedback_focus` to focus the extraction. You are extracting feedback **about the configured product** as expressed by prospects in these calls — not random sales chatter.

The four categories:

- **User needs** — the actual job-to-be-done they're trying to do. What problem is this prospect trying to solve, in their own words? Not "they want feature X"; rather "they're trying to maintain accurate notes across many concurrent client engagements because mixing up clients is a real risk." The category answers "why are they here at all?"
- **Product quality** — explicit signals about whether the product is good or has concerns. Praise, complaints, comparisons to competitors, things that broke, things that worked, NPS-shaped statements.
- **Feature gaps** — specific functionality they want that we don't have today. Be precise: "save reusable prompts in the chat sidebar" is a feature gap; "make the assistant better" is too vague — file that as general feedback.
- **General feedback** — anything else relevant: workflow observations, integration requests with named tools, pricing/packaging objections, onboarding friction, security/privacy constraints, accessibility, perceived missing roadmap items.

Two items can fall into more than one category. Don't force exclusivity — emit both and let trend analysis dedupe. Always quote the prospect verbatim where possible and capture the speaker's name and timestamp from the transcript so the Product team can verify.

### Step 4 — Tag persona and vertical

**Persona.** If `tagging.personas_file` is null, skip persona tagging — items have no `persona` field.

If a personas file is configured, load it and use the rules inside to tag each item. The expected structure (routing rules, title heuristic, behavioral axes) is documented in `references/personas_example.md`. Apply the rules in the order they appear in the user's file.

If a persona can't be inferred confidently, mark `unknown` rather than guessing. Trend analysis tolerates `unknown` better than wrong tags.

**Vertical.** Infer from email domain and company name in the transcript. Use the taxonomy in `tagging.verticals` if configured, otherwise the coarse default from `references/extraction_schema.md`. Don't invent fine-grained subcategories — coarse buckets make trend analysis stable across weeks.

### Step 5 — Synthesize trends

Read the last `trend_window_weeks` (default 8) snapshots from `<cwd>/<outputs.local.snapshots_dir>/` and the running `<cwd>/<outputs.local.trends_file>`. Then produce three things:

- **Recurring themes** — items that appear across multiple weeks. Promote any feature gap mentioned in 3+ weeks within the window to a "recurring" call-out at the top of the digest. Note the count and the personas that raised it (if persona tagging is enabled).
- **Emerging signals** — items raised this week that match wording or intent of items raised in 1–2 prior weeks. Don't promote yet, but flag for watch.
- **New this week** — items with no prior match.

The trends file (`rolling-trends.md`) is the durable accumulator. Update it in place: append new themes, increment counts on recurring ones, and demote themes that haven't appeared in the trend window. The format is in `references/output_templates.md`.

### Step 6 — Produce outputs

Produce artifacts in this order based on which outputs are enabled in config. Use the templates in `references/output_templates.md` — they handle formatting consistently across runs, which matters for diffing snapshots over time.

**1. Local markdown snapshot** (when `outputs.local.enabled`). Write to `<cwd>/<outputs.local.snapshots_dir>/YYYY-MM-DD-feedback.md` where the date is the end of the window. This is the canonical output and the source of truth. Include: window dates, count of calls processed, the four categories with structured items, persona/vertical breakdowns (if tagging is enabled), and a trends section.

**2. Wiki page** (when `outputs.wiki.target` is set). Dispatch on the target and route to the matching adapter in `references/wiki_adapters.md`. The page is rolling — its content is replaced each run with the most recent week's digest plus a "Trends" section at the top. All four wiki targets preserve version history (Confluence revisions, Notion edit history, SharePoint version history, OneNote edit history), so prior weeks remain inspectable.

If a page ID is missing in config:
- **Interactive mode** — ask the user for the page ID or URL, then write it back to config.
- **Auto mode** — create a new page under the adapter's configured parent (`space_key`, `parent_page_id`, `site_id`, or `section_id` respectively) titled per the adapter's `page_title`, and write the new ID back to config.

Never overwrite an existing `page_id` silently.

**3. Chat digest** (when `outputs.chat.target` is set). Render a concise summary and post it via the configured chat adapter (see `references/chat_adapters.md`). The chat message is short — a TL;DR up top, the top 3 recurring themes, the top 3 new feature gaps this week, and a link to the full wiki page and the local markdown. Keep it under 30 lines so the Product team can scan it. Template in `references/output_templates.md`; per-adapter formatting (Slack mrkdwn vs Teams HTML) lives in `chat_adapters.md`.

### Step 7 — Write a run log

Append to `<cwd>/.feedback-digest-runs.jsonl` (one JSON line per run):

```json
{"run_at":"2026-05-06T18:30:00Z","mode":"auto","window":["2026-04-29","2026-05-06"],"calls_processed":7,"items_extracted":34,"snapshot":"snapshots/2026-05-06-feedback.md","outputs_used":["local","wiki","chat"],"wiki":{"target":"confluence","page_id":"1234567890","version":42},"chat":{"target":"slack","channel":"#product","message_ref":"1714923600.001"}}
```

Why this matters: scheduled runs happen unattended. The run log is the path to rollback. If a chat post turns out to be wrong, `chat.message_ref` lets you delete the right message via the chat adapter. If a wiki update was bad, `wiki.version` (Confluence) or the prior page state (other adapters) tells you what to revert to.

## Mode behavior

**Interactive mode (default when a human is asking):**
1. Run Step 0 if config is missing or incomplete.
2. Run Steps 1–5 silently and write the local snapshot.
3. Show the user the snapshot path and a brief summary (count of calls, top themes).
4. Ask: "Ready to update the wiki and post to chat?" Wait for explicit confirmation.
5. On confirmation, run Step 6 (wiki + chat) for whichever outputs are enabled, then Step 7 (run log). On decline, stop after the local snapshot — that's still useful on its own.

**Auto mode (scheduled task):**
1. Fail loudly if config is missing or invalid. Do not start the wizard from a scheduled run.
2. Run all post-setup steps end-to-end without prompting.
3. Always write the run log regardless of success or failure.
4. If any step fails, write a failure entry to the run log with `error` and the partial outputs that did succeed. Don't partially update the wiki with a half-baked digest — either the digest is complete or the wiki isn't touched.

**Dry-run mode (`--dry-run`):**
- Run Steps 1–5 normally and write the local snapshot. Skip Step 6 wiki/chat writes and Step 7 entirely. Useful for validating the skill against a new week of data without side effects.

## Privacy and safety

These calls contain prospect names, company names, and sometimes confidential business context. Apply these rules:

- **Verbatim quotes are fine** in the local markdown and the wiki (both are internal team surfaces). Quote prospects with attribution where attribution is available.
- **Chat post must be lighter on PII.** Use first name + company only. Don't paste long verbatim quotes into the chat post — link to the wiki page instead. Chat messages (Slack threads, Teams channel posts) are shared more loosely than wiki pages.
- **Never include credentials, contracts, or pricing screenshots** even if they appear in transcripts. If a transcript captures a pricing quote, summarize as "discussed pricing" without the figure.
- **Do not name internal employees of your own org as the source of feedback.** The feedback comes from prospects. Internal participants are context but not the subject of the digest.
- **Never delete or overwrite snapshot files.** Snapshots are the historical record. If a re-run is needed for the same week, write to `YYYY-MM-DD-feedback-rev2.md` rather than overwriting.

## When this skill should fail loudly vs gracefully

Fail loudly (raise/return an error, do not write chat/wiki):
- Read AI MCP returns an auth error or 5xx.
- Wiki config exists but the page_id 404s — something has been moved or deleted, and we shouldn't silently create a new one.
- The trend file is malformed in a way that we can't parse — we'd rather flag it than silently lose history.
- Config is missing in `--mode auto`.

Fail gracefully (write what we can, log the gap):
- Individual transcripts fail to expand — proceed with summary-only for those meetings, mark the items.
- Chat post fails — the markdown and wiki are still useful; log the chat failure to the run log.
- Persona inference is uncertain — emit `unknown` and continue.
- A configured optional output is enabled but its MCP isn't connected — skip that output, log it, continue with the rest.

## Reference files

- `references/connector_setup.md` — Wizard-side reference. How to detect each MCP (Read AI, Confluence, Notion, Microsoft 365, Slack) and what to tell the user when one is missing.
- `references/personas_example.md` — Template the user copies and adapts to define their own persona framework. Includes the structure (routing rules, title heuristic, behavioral axes) with fictional placeholder personas so the shape is obvious.
- `references/extraction_schema.md` — Schema for individual feedback items, with worked examples for each of the four categories. Read when you're not sure how to structure an item.
- `references/output_templates.md` — Generic templates for the markdown snapshot, the rolling trends file, the rolling wiki page structure, and the chat message structure. Read once at the start of Step 6, then apply consistently.
- `references/wiki_adapters.md` — The pluggable wiki abstraction (`resolve_or_create_page`, `update_page`, `get_page_url`). Per-adapter mappings for Confluence and Notion (implemented day one) and SharePoint Pages and OneNote via Microsoft 365 (documented adapters; writes pending M365 write capability).
- `references/chat_adapters.md` — The pluggable chat abstraction (`post_message`). Per-adapter mappings for Slack (implemented day one) and Teams via Microsoft 365 (documented adapter; same write caveat).
