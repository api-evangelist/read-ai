---
name: friday-followups
description: A scheduled weekly planner skill. Pulls the past week's meetings from the Read AI connector and direct messages plus @-mentions from the user's connected messaging app (Slack on day one, Microsoft Teams via the Microsoft 365 connector also supported), LLM-judges which action items and messages are actually actionable for the user, estimates duration per todo using bounded buckets, groups todos by topic/project across both sources, and books tentative calendar blocks for the upcoming workweek on Google Calendar (Outlook Calendar via the Microsoft 365 connector also supported). Produces a dated local markdown digest alongside the calendar writes. Designed to run unattended on a Friday-AM cron in the user's local timezone, with strong safety properties — every event is tentative + private + tagged, daily caps prevent calendar takeover, double-booking is prevented, and a one-command undo deletes everything the most recent run created. Use this skill whenever the user mentions a weekly follow-ups planner, a Friday-AM meeting-action rollup, automatic calendar blocking from meeting transcripts, or asks "what do I owe people from this week's meetings and when am I going to do it."
---

# Friday Followups

## What this skill does

Every Friday morning (or whenever the user runs it on demand), this skill converts the past week's accumulated commitments — meeting action items, direct messages, channel mentions — into a structured, topic-grouped plan for the upcoming workweek, and books tentative calendar blocks so the work has time reserved before Monday starts.

Every run does these things in order:

1. **Plan phase (in memory):**
   1. Fetch meetings + action items from the Read AI connector for the past 7 days.
   2. Fetch DMs + @-mentions to the user from the configured messaging adapter (Slack or Teams) for the past 7 days.
   3. LLM-judge each candidate as a relevant todo for *this* user (`yes` / `no` / `maybe`).
   4. LLM-estimate duration per todo, clamped to bounded buckets (default 15/30/60/90/120 min).
   5. LLM-group todos by topic/project, across both sources.
   6. Enrich each group with related messaging context from the past 14 days (top 1–2 threads).
   7. Fit groups into open calendar slots under all constraints (working hours, no double-book, daily cap).
2. **Apply phase:**
   - Always write a dated markdown digest in `./digests/`.
   - Unless `--dry-run`, create tentative calendar events (one per group).
   - Append an entry to the run log.

The skill is built to run unattended on a schedule. It is also safe to run interactively — in that mode it shows the plan first and asks before creating calendar events.

## Required connectors

This skill is a multi-connector demo. It needs:

- **Read AI MCP** (required) — meeting metadata, action items, transcripts. The skill cannot run without it.
- **Calendar MCP** (required, pluggable):
  - **Google Calendar** — implemented day one.
  - **Outlook Calendar via Microsoft 365 connector** — see `references/calendar_adapters.md` for the documented adapter mapping.
- **Messaging MCP** (optional, pluggable; recommended):
  - **Slack** — implemented day one.
  - **Microsoft Teams via Microsoft 365 connector** — see `references/messaging_adapters.md`.

The first-run wizard (Step 0) detects which of these are connected and adapts accordingly. If no messaging connector is wired, the skill runs in meeting-only mode and notes that in the digest.

## Configuration

Stored in the working directory at:

```
./.friday-followups-config.json
```

Expected shape:

```json
{
  "identity": {
    "email": "user@example.com",
    "aliases": ["FirstName", "Initials"],
    "timezone": "America/Los_Angeles"
  },
  "working_hours": {
    "start": "09:00",
    "end": "17:00",
    "days": ["mon", "tue", "wed", "thu", "fri"]
  },
  "planning": {
    "lookback_days": 7,
    "plan_horizon_days": 5,
    "max_block_minutes_per_day": 180,
    "duration_buckets_minutes": [15, 30, 60, 90, 120],
    "min_block_minutes": 15
  },
  "calendar": {
    "source": null,
    "google": { "calendar_id": "primary" },
    "outlook": { "calendar_id": "primary" },
    "event_defaults": {
      "title_prefix": "[Followups] ",
      "status": "tentative",
      "transparency": "opaque",
      "visibility": "private",
      "extended_property_tag": "friday-followups"
    }
  },
  "messaging": {
    "source": null,
    "lookback_days": 7,
    "enrichment_lookback_days": 14,
    "include_dms": true,
    "include_mentions": true,
    "enrich_with_related_context": true,
    "slack": {},
    "teams": {}
  },
  "outputs": {
    "digests_dir": "digests"
  },
  "relevance": {
    "cache_file": ".friday-followups-cache.json"
  }
}
```

Field notes:

- `identity.aliases` lets the relevance judgment catch references like initials or first-name-only mentions.
- `calendar.source` is `"google"` or `"outlook"`. `messaging.source` is `"slack"`, `"teams"`, or `null`. If a `source` is null and exactly one matching MCP is connected, the skill auto-resolves to it.
- `event_defaults.extended_property_tag` is the marker the skill writes on every event so undo can find them even if the run log is lost.
- `planning.duration_buckets_minutes` is configurable — e.g., a team that prefers 25/50-minute Pomodoro blocks can set it to `[25, 50, 75, 100, 125]`.
- `planning.min_block_minutes` is the threshold at which the skill bothers booking; smaller todos still appear in the digest but get bundled into their group's block.

## Inputs

The skill accepts these arguments (all optional):

| Arg | Purpose | Default |
| --- | --- | --- |
| `--start` | Window start (ISO 8601, inclusive) | `now - planning.lookback_days` |
| `--end` | Window end (ISO 8601, inclusive) | now |
| `--mode` | `interactive` or `auto` | `interactive` if a human is in the loop, `auto` if invoked from a scheduled task |
| `--dry-run` | Skip calendar writes (markdown still written) | false |
| `--config` | Path to config file | `./.friday-followups-config.json` |
| `--reconfigure` | Re-run the first-run wizard against an existing config | false |
| `--undo` | Delete events from the most recent run | false |
| `--by-tag` | (with `--undo`) Broader fallback: delete every event in the planning horizon with the extended-property tag | false |

When invoked from a scheduled task, default to `--mode auto`.

## Step 0 — First-run setup wizard

Fires when the config file is missing, when required fields are unset, or when the user passes `--reconfigure`. Skipped in `--mode auto`: scheduled runs must fail loudly rather than prompt.

### 0.1 — Greet and explain

Tell the user what the skill does in two sentences. Note that setup will write a config file to the current directory.

### 0.2 — Connector check

Inspect the available tools in the current session. For each of the following, report `connected` or `missing` (see `references/connector_setup.md` for the detection patterns):

- Read AI MCP (required).
- Calendar MCPs: Google Calendar, Microsoft 365 (covers Outlook Calendar).
- Messaging MCPs: Slack, Microsoft 365 (covers Teams).

Hard stops:

- **Read AI missing** — stop, show setup pointer, do not proceed.
- **No calendar MCP connected** — stop, show setup pointer.

Soft cases:

- **No messaging MCP connected** — warn the user that the skill will run meeting-only, continue.
- **Multiple calendar or messaging MCPs connected** — ask the user to pick a source for each role.

### 0.3 — Identity

Ask for:

- `identity.email` — the user's primary email (used to find them in attendee lists and for relevance judgment).
- `identity.aliases` — short list of names/initials the user is addressed by (e.g., first name, two-letter initials).
- `identity.timezone` — default to the system timezone; confirm or override (must be a valid IANA tz like `America/Los_Angeles`).

### 0.4 — Working hours

- `working_hours.start` and `working_hours.end` — default `09:00`–`17:00`.
- `working_hours.days` — default `["mon", "tue", "wed", "thu", "fri"]`.

Confirm or override. These bound every calendar slot the skill considers.

### 0.5 — Caps and buckets

- `planning.max_block_minutes_per_day` — default 180 (3 hours). Anything beyond rolls to the digest's Overflow section.
- `planning.duration_buckets_minutes` — default `[15, 30, 60, 90, 120]`. Confirm or override.

### 0.6 — Adapter selection

For each pluggable role (calendar, messaging), if multiple MCPs are connected, ask the user to pick:

- `calendar.source` ← `"google" | "outlook"`. Also confirm `calendar.<source>.calendar_id` (default `"primary"`).
- `messaging.source` ← `"slack" | "teams" | null`.

If only one MCP is connected for a role, pre-fill the source and confirm.

### 0.7 — Write config

Save `./.friday-followups-config.json`.

### 0.8 — Schedule pointer

Print, but do not install, the recommended cron expression for the user's interface:

```
To run Friday-followups every Friday at 07:00 in your timezone:

  In Claude Code:
    /schedule "0 7 * * 5" "/skill friday-followups --mode auto"

  In Cowork (Scheduled Agents):
    Create a scheduled agent that fires Friday 07:00 in your timezone
    Prompt: invoke the friday-followups skill in auto mode
```

### 0.9 — Smoke test offer

Offer to run `--dry-run` against the past 7 days as a validation step. This exercises Read AI ingestion and (if connected) the messaging adapter without touching the calendar.

## Step 1 — Fetch candidates

**Meetings (Read AI).**

- `list_meetings` with `start_datetime_gte = now − planning.lookback_days`, `start_datetime_lte = now`. Loop with cursor pagination until `has_more` is false.
- For each meeting where `identity.email` is in the attendee list, call `get_meeting_by_id` with `expand: ["summary", "action_items", "key_questions"]`. Skip `metrics` and `recording_download`.
- **Tier-2 transcript fallback** (`expand: ["transcript"]`) only when an action item's owner or intent is genuinely unclear from the structured layers. Process meetings one at a time when fetching transcripts; loading multiple in parallel risks context overflow.

**Messaging (if `messaging.source` is not null).**

Through the configured adapter (see `references/messaging_adapters.md`):

- DMs to the user in the past `messaging.lookback_days` days.
- Channel messages where the user is @-mentioned in the past `messaging.lookback_days` days.
- For each match, fetch the thread contents and the permalink.

Skip this step entirely when `messaging.source` is null.

**Output of Step 1** — a flat list of candidates. Each carries:

- `source_type`: `meeting_action_item` | `messaging_dm` | `messaging_mention`
- `source_id`: stable upstream ID
- `source_link`: clickable permalink (Read AI `report_url` or messaging permalink)
- `text`: the raw candidate content
- `context`: surrounding context the LLM will need for relevance judgment (meeting summary, prior messages in thread)
- `attendees` or `participants`: for grouping

## Step 2 — LLM-judge relevance

For each candidate, the LLM judges whether it represents an actionable todo for the user. Inputs to the prompt: `identity.email`, `identity.aliases`, the candidate text, the candidate context.

Output schema per candidate:

```json
{
  "source_id": "...",
  "relevant": "yes" | "no" | "maybe",
  "reason": "<one-line reasoning>",
  "todo_summary": "<one-sentence refined todo>"
}
```

- `yes` → enters the planning pipeline.
- `maybe` → logged in the digest under "Review by hand", not booked.
- `no` → dropped silently (but counted in the run log).

**Cache verdicts** in `./.friday-followups-cache.json` keyed by `source_id` so re-runs against the same week don't re-judge unchanged items. The cache stores verdicts + IDs + reasoning, *not* raw message bodies (see Privacy section).

## Step 3 — Estimate duration and group by topic

Single LLM pass over the `yes` todos. For each todo, output:

- `duration_minutes` — one of `planning.duration_buckets_minutes`. Clamp anything off-bucket to the nearest bucket value.
- `group_name` — a short topic/project label. Grouping is cross-source by design: a meeting action item and a Slack mention on the same project must land in the same group.

If a group has only one todo, the group name equals the todo summary.

Reshape the output into groups, each with name and `[ (todo, duration_minutes, source) ]`.

## Step 4 — Enrich with messaging context

Skipped when `messaging.source` is null or `messaging.enrich_with_related_context` is false.

For each group:

- Build a search query from the group name, any named people across the group's source items, and any company/project names visible in the candidate text or context.
- Search the messaging source for matching threads in the past `messaging.enrichment_lookback_days` days (default 14).
- Take the top 1–2 results per todo (rank by adapter's native relevance score). Attach permalinks + a one-line excerpt as `related_context` on the todo.

Bounded: at most ~2 messaging searches per group, capped further by the daily block cap downstream.

## Step 5 — Fit groups into calendar slots

- Read availability from the configured calendar adapter (`freebusy` over the planning horizon, `planning.plan_horizon_days` days from the upcoming Monday).
- For each day, compute candidate slots = `working_hours` window minus confirmed events on the user's primary calendar.

For each group, in descending size order:

- `total_duration = sum(todo.duration for todo in group)`.
- If a todo's individual duration is below `planning.min_block_minutes`, it doesn't get its own event but its time still counts in the group total.
- Find the **earliest** contiguous slot in the workweek that fits `total_duration`, doesn't overlap existing events, and respects the daily cap. Earliest-fit is the deliberate default — users expect Monday-morning slots to fill before Friday-afternoon ones.
- If no contiguous slot fits, attempt a split across days; record the split in the group's metadata.
- If still won't fit anywhere this week, mark the group `overflow`.

## Step 6 — Apply

### Always

Write the digest to `./<outputs.digests_dir>/YYYY-MM-DD-followups.md` (date is the day the skill runs). Format in `references/output_templates.md`. The digest includes:

- TL;DR (counts: groups, hours planned, overflow).
- The plan: every group, every todo, durations, source links, related context, the slot it landed in.
- "Review by hand" section listing `maybe` verdicts.
- "Overflow" section listing groups that didn't fit.
- "Notes on this run" with which adapter was used and any errors.

### Unless `--dry-run`

For each non-overflow group, create a tentative calendar event via the configured adapter:

- **Title:** `{calendar.event_defaults.title_prefix}{group_name}` (default prefix `[Followups] `).
- **Description:** bulleted checklist of todos with source-meeting / source-message permalinks, plus the related-context block.
- **Status:** `tentative` (Google) / `showAs: "tentative"` (Outlook).
- **Visibility:** `private`.
- **Extended property:** `{calendar.event_defaults.extended_property_tag}: "friday-followups"`.

### Pre-write idempotency check

Right before writes, re-read the calendar over the planning window for events carrying the configured extended-property tag. For each new planned block:

- (group, start) matches existing tagged event → reuse the event ID, no write.
- existing tagged event not in new plan → delete it.
- new plan has no existing match → create.

Net effect: running the skill twice in the same week converges to the same calendar state instead of stacking duplicates.

## Step 7 — Run log

Append one JSON line per run to `./.friday-followups-runs.jsonl`:

```json
{
  "run_at": "2026-05-22T07:00:00-07:00",
  "mode": "auto",
  "config_snapshot": {
    "calendar_source": "google",
    "messaging_source": "slack",
    "max_block_minutes_per_day": 180
  },
  "candidates": {"meeting_action_item": 18, "messaging_dm": 6, "messaging_mention": 9},
  "verdicts": {"yes": 14, "maybe": 4, "no": 15},
  "groups": 5,
  "blocks_created": [
    {"event_id": "abc123", "calendar": "google", "calendar_id": "primary",
     "group": "Q3 launch", "start": "2026-05-25T10:00-07:00", "end": "2026-05-25T11:30-07:00"}
  ],
  "blocks_skipped_overflow": [{"group": "Vendor review", "total_duration": 90}],
  "digest_path": "digests/2026-05-22-followups.md",
  "errors": []
}
```

For `--dry-run`, replace `blocks_created` with `dry_run_blocks` so a follow-up apply can run without re-planning.

For `--undo`:

```json
{
  "run_at": "2026-05-22T09:14:00-07:00",
  "mode": "undo",
  "undid_run_at": "2026-05-22T07:00:00-07:00",
  "deleted_event_ids": ["abc123", "def456"]
}
```

The run log is the rollback path. Keep it append-only; never rewrite it.

## Mode behavior

**Interactive mode** (default when a human is asking):

1. Run Step 0 (wizard) if config is missing or incomplete.
2. Run Steps 1–5 silently. Write the markdown digest.
3. Show the user the digest path and a one-line summary: "5 groups, 6 hours planned, 2 in overflow."
4. Ask: "Book the N calendar blocks?" Wait for explicit confirmation.
5. On yes → run Step 6 calendar writes + Step 7 run log. On no → markdown stays; nothing booked.

**Auto mode** (scheduled task):

1. Fail loudly if config is missing or invalid. Do **not** start the wizard from a scheduled run.
2. Run Steps 1–6 end-to-end without prompts. Tentative events created directly.
3. Always write the run log, even on failure.
4. If any step fails, write a failure entry to the run log with `errors[]` populated and the partial outputs that did succeed.

**Dry-run** (`--dry-run`, valid in interactive or auto):

1. Run Steps 1–5 normally.
2. Write the markdown digest.
3. Skip Step 6 calendar writes; record would-be events under `dry_run_blocks` in the run log.

**Reconfigure** (`--reconfigure`): force the wizard against existing config. Preserves identity history but lets the user change adapters, caps, working hours.

**Undo** (`--undo`):

1. Read the most recent successful (non-undo) run-log entry.
2. For each `event_id` in `blocks_created`, delete the event via the calendar adapter.
3. Append an undo line to the run log.
4. Report a summary: "Deleted N events from run <iso_timestamp>."

**Undo by tag** (`--undo --by-tag`, fallback when the run log is unrecoverable):

1. Find every event on the configured calendar over the next `plan_horizon_days` carrying the `friday-followups` extended-property tag.
2. In interactive mode: confirm with the user before deletion. In auto mode: refuse (too broad to run unattended).
3. Delete confirmed events. Append an undo line to the run log.

## Calendar safety

All booking constraints enforced in code, every run:

1. **Working hours only** — never book outside `working_hours.start`–`working_hours.end` in `identity.timezone`. Restrict to `working_hours.days`.
2. **No double-book** — `freebusy` query before placement. Skip any slot overlapping a confirmed event. Tentative events from prior runs of *this skill* are overwritable; tentative events from any other source are protected.
3. **Daily cap** — sum of placed blocks per day ≤ `planning.max_block_minutes_per_day`. Excess → Overflow.
4. **Min block** — todos under `planning.min_block_minutes` bundled into a group block.
5. **Tentative status** — every event created tentative.
6. **Title + property tagging** — every event carries `title_prefix` and the `friday-followups` extended property. Two independent identifiers for undo.

## Privacy + safety

- **Tentative + private** events by default. Calendar-sharing colleagues see "busy", not contents.
- **No external posts.** The skill never writes to Slack/Teams. Messaging access is read-only.
- **No PII to LLM beyond what's already in Read AI / Slack / Teams.** No external enrichment services.
- **Cache contains verdicts + IDs + reasoning, not raw message bodies.** Treat at the same trust level as the run log.
- **Thread contents stay in-process.** Pulled into context for the LLM judgment, then dropped; only the verdict and a permalink make it into the digest and run log.
- **Never delete or overwrite digest files.** Re-runs that produce a new digest for the same date write `YYYY-MM-DD-followups-rev2.md` rather than overwriting.

## When this skill should fail loudly vs gracefully

**Fail loudly** (raise/return an error, do not write calendar):

- Read AI MCP returns auth error or 5xx.
- Calendar MCP returns auth error or 5xx.
- `freebusy` returns inconsistent results (e.g., overlapping confirmed events that the API both lists and hides — indicates a permissions issue we shouldn't paper over).
- Configured `calendar.source` adapter isn't connected at run time (in `--mode auto`).
- `--mode auto` and config is missing or invalid.

In each case: write a markdown digest with what *would* have been booked, populate `errors[]` in the run log, exit non-zero, no events created.

**Fail gracefully** (write what we can, log the gap):

- Messaging MCP fails — proceed meeting-only, log the gap.
- Individual transcripts fail to expand — proceed with summary-only for those meetings, mark the items.
- Individual calendar event create fails — record in `errors[]`, continue with remaining groups.
- LLM returns malformed output for a single candidate — log and skip that candidate.

## Reference files

- `references/connector_setup.md` — Wizard-side reference. How to detect each MCP (Read AI, Google Calendar, Slack, Microsoft 365), and what to tell the user when one is missing.
- `references/calendar_adapters.md` — The pluggable calendar abstraction. Google Calendar mapping (implemented day one) and Outlook via Microsoft 365 mapping (documented; tool names get filled in at wire-up time).
- `references/messaging_adapters.md` — The pluggable messaging abstraction. Slack mapping (implemented day one) and Teams via Microsoft 365 mapping (documented).
- `references/output_templates.md` — Markdown digest template and the run-log JSON shape. Read once at the start of Step 6 and apply consistently across runs.
