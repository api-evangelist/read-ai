---
name: sales-call-review
description: Weekly sales review skill that analyzes the past week's customer calls from the Read AI connector and produces a single self-contained interactive HTML report with two primary views — account-centric deal health (every account that had calls in the window with universal + optional framework deal-health signals, enriched with CRM data when a HubSpot or Salesforce MCP is connected) and per-rep coaching (every internal rep on calls with universal + optional framework coaching dimensions, talk-time analysis from Read AI metrics, and concrete moments to coach on). Reps are auto-identified by internal email domain or CRM user list — no manager-vs-rep mode toggle needed. The HTML report has filterable / sortable account and rep cards, inline Chart.js visualizations (talk-time bars, dimension scores, sentiment trajectories), and stable anchors for deep-linking. CRM is optional; the skill falls back to email-domain grouping and runs without deal-stage context when no CRM is connected. Designed to run unattended on a Friday-afternoon cron in the user's local timezone but safe to run on-demand at any time. Use this skill whenever the user mentions a weekly sales-call review, a coaching report from the past week's calls, account / deal-health rollups from meeting transcripts, or asks "how did this week's customer calls go and what should I coach my reps on?"
---

# Sales Call Review

## What this skill does

Every Friday afternoon (or whenever the user runs it on demand), this skill converts the past week's customer calls into a single self-contained interactive HTML report. The report has two primary surfaces:

- **Per-account deal health** — every account that had calls in the window, scored on universal dimensions (stakeholder breadth, sentiment trend, urgency, budget signals, next-step clarity, days since last touch, risk flags, strength signals) and any framework dimensions the user has configured. When CRM is connected, accounts also show deal stage, amount, owner, and close date.
- **Per-rep coaching** — every internal rep on the call set, scored on universal dimensions (talk-time ratio, question density, discovery quality, objection handling, next-step clarity, filler / hedging, listening signals) plus any framework dimensions. Each dimension includes a specific moment to coach on (a transcript quote with a timestamp and a one-line "what to try instead").

The report file is the only output — no calendar writes, no chat posts, no CRM writebacks. Designed to be opened in a browser, navigated, and acted on.

## Required connectors

This skill is a multi-connector demo. It needs:

- **Read AI MCP** (required) — meeting metadata, action items, transcripts, and the `metrics` payload that supplies talk-time data. The skill cannot run without it.
- **CRM MCP** (optional, pluggable, recommended):
  - **HubSpot** — implemented day one.
  - **Salesforce** — implemented day one.
  - See `references/crm_adapters.md`.

When no CRM is connected, the skill falls back to email-domain grouping and runs without deal-stage context. The report's header explicitly notes the absence so the user understands the trade-off.

## Configuration

Stored in the working directory at:

```
./.sales-call-review-config.json
```

Expected shape:

```json
{
  "identity": {
    "email": "user@example.com",
    "display_name": "User Name",
    "internal_domains": ["yourcompany.com"],
    "timezone": "America/Los_Angeles"
  },
  "filter": {
    "lookback_days": 7,
    "title_include_patterns": [],
    "title_exclude_patterns": ["interview", "recruiting", "internal", "1:1", "weekly", "all-hands"],
    "exclude_domains": ["greenhouse.io", "lever.co"]
  },
  "crm": {
    "source": null,
    "hubspot": { "portal_id": null },
    "salesforce": { "org_id": null, "default_pipeline": null }
  },
  "coaching": {
    "framework_file": null
  },
  "deal_health": {
    "framework_file": null
  },
  "accounts": {
    "domain_to_name_overrides": {}
  },
  "outputs": {
    "reports_dir": "reports",
    "file_mode": "0600"
  },
  "cache": {
    "cache_file": ".sales-call-review-cache.json"
  }
}
```

Field notes:

- `identity.internal_domains` is load-bearing — it powers both the call filter (external attendees) and the fallback rep identification when no CRM is connected.
- `filter.title_exclude_patterns` defaults generously. The cost of analyzing a wrongly-included internal sync is real (LLM cost + clutter). Better to exclude too aggressively and let the user widen the net.
- `filter.exclude_domains` is the safety valve for external attendees who aren't sales prospects (ATSes, payroll vendors, lawyers, agencies).
- `crm.source` is `"hubspot" | "salesforce" | null`. Auto-resolves when exactly one CRM MCP is connected. Per-adapter config blocks are minimal — most CRM identity comes from OAuth context.
- `coaching.framework_file` and `deal_health.framework_file` are paths relative to the config file (e.g., `./coaching-framework.md`). Null by default; user copies `references/coaching_framework_example.md` (or the deal-health equivalent) to a path of their choice, adapts it, and points config at the copy.
- `accounts.domain_to_name_overrides` is only meaningful when CRM is missing — pretty-names domains like `acme.com` → `Acme Corp` for the fallback grouping. When CRM is connected, the CRM account name is used.
- `outputs.file_mode` is the POSIX file mode applied via `chmod` after the HTML is written. `0600` (owner only) by default — the report contains sensitive coaching critiques and deal data.

## Inputs

The skill accepts these arguments (all optional):

| Arg | Purpose | Default |
| --- | --- | --- |
| `--start` | Window start (ISO 8601, inclusive) | `now - filter.lookback_days` |
| `--end` | Window end (ISO 8601, inclusive) | now |
| `--mode` | `interactive` or `auto` | `interactive` if a human is in the loop, `auto` if invoked from a scheduled task |
| `--dry-run` | Stop after Step 3, print preview, skip analyses + render | false |
| `--config` | Path to config file | `./.sales-call-review-config.json` |
| `--reconfigure` | Re-run the first-run wizard against an existing config | false |

When invoked from a scheduled task, default to `--mode auto`.

## Step 0 — First-run setup wizard

Fires when the config file is missing, when required fields are unset, or when the user passes `--reconfigure`. Skipped in `--mode auto`: scheduled runs must fail loudly rather than prompt.

### 0.1 — Greet and explain

Tell the user what the skill does in two sentences. Note that setup writes a config file and that the rendered HTML report contains sensitive coaching critiques and deal data (`0600` file mode is enforced).

### 0.2 — Connector check

Inspect the available tools in the current session. For each of the following, report `connected` or `missing` (see `references/connector_setup.md`):

- Read AI MCP (required).
- CRM MCPs: HubSpot, Salesforce.

Hard stop on missing Read AI.

If no CRM is connected, warn the user that the skill will run in no-CRM mode (email-domain grouping, no deal-stage context) and continue. Multiple CRMs connected → ask the user to pick `crm.source`.

### 0.3 — Identity

Ask for:

- `identity.email` — the user's primary email.
- `identity.display_name` — name to use in the report.
- `identity.internal_domains` — domains belonging to the user's organization. Required. Examples should be concrete (e.g., `["yourcompany.com", "yourcompany.io"]`).
- `identity.timezone` — default to system tz; confirm.

### 0.4 — Filter

- `filter.lookback_days` — default 7, confirm.
- `filter.title_include_patterns` — patterns that force-include a meeting even without an external attendee. Default empty.
- `filter.title_exclude_patterns` — patterns that force-exclude. Default `["interview", "recruiting", "internal", "1:1", "weekly", "all-hands"]`.
- `filter.exclude_domains` — external domains that aren't sales prospects. Default `["greenhouse.io", "lever.co"]`.

### 0.5 — Coaching framework

Ask: "Do you want to extend the universal coaching dimensions with your own framework (MEDDIC, BANT, SPICED, custom)?"

- **No** → leave `coaching.framework_file` null.
- **Yes** → walk the user through copying `references/coaching_framework_example.md` to a path of their choice (suggest `./coaching-framework.md`), adapting it to their org, then set `coaching.framework_file` to that path. Don't try to write the framework for them — that's the user's call.

### 0.6 — Deal-health framework

Same pattern with `references/deal_health_framework_example.md`.

### 0.7 — Account naming overrides (no-CRM only)

If no CRM is connected, optionally collect `accounts.domain_to_name_overrides` for any domains the user wants pretty-named. Skip when CRM is connected — the CRM provides account names.

### 0.8 — Write config and schedule pointer

Save `./.sales-call-review-config.json`. Print, but do not install, the recommended cron:

```
sales-call-review is configured. Recommended schedule: Friday 16:00 in your local timezone.

  In Claude Code:
    /schedule "0 16 * * 5" "/skill sales-call-review --mode auto"

  In Cowork (Scheduled Agents):
    Schedule: every Friday at 16:00 in your timezone
    Prompt: invoke the sales-call-review skill in auto mode
```

### 0.9 — Smoke test offer

Offer to run `--dry-run` against the past 7 days. Validates Read AI access, filter behavior, CRM lookup, and rep+account identification — without paying for the full analyses.

## Step 1 — Fetch and filter calls

Use the Read AI MCP's `list_meetings` tool with `start_datetime_gte` and `start_datetime_lte` set to the window. Loop with cursor pagination until `has_more` is false.

**Default include rule:** at least one attendee email NOT in `identity.internal_domains` AND NOT in `filter.exclude_domains`.

**Title overrides:**

- If meeting title matches any pattern in `filter.title_include_patterns` (case-insensitive substring), force-include even without an external attendee.
- If meeting title matches any pattern in `filter.title_exclude_patterns`, force-exclude regardless of attendees.

Carry exclusion reasons forward — they render in the report's "All calls" audit table.

If the resulting list is empty, write a "no sales calls in window" HTML report using the minimal template in `references/output_templates.md` and stop.

## Step 2 — Pull rich content per call (tiered fetch)

For each kept meeting, fetch in two tiers.

**Tier 1 — structured layers + metrics.** Call `get_meeting_by_id` with `expand: ["summary", "action_items", "key_questions", "topics", "metrics"]`. Note `metrics` is required here — Read AI's speaker-time data is the source-of-truth for talk-time analysis (the LLM doesn't compute talk-ratio; it contextualizes Read AI's numbers).

**Tier 2 — transcript fallback.** Re-fetch with `expand: ["transcript"]` only when the per-rep coaching analysis needs to quote specific moments (objection handling examples, listening-signal evidence, next-step exchanges). Most calls will need this. Process meetings one at a time when fetching transcripts to keep context manageable.

**Cache analyses** in `./.sales-call-review-cache.json` keyed by meeting ID. Re-runs against the same week skip already-analyzed calls. Invalidate cache entries when the meeting ID disappears upstream or when the cached entry is older than 30 days.

## Step 3 — Identify reps and group by account

### Rep identification

For each call's attendees:

- **CRM connected:** resolve each attendee's email against the CRM user list (`list_users` operation from `crm_adapters.md`). Matches → reps.
- **CRM absent:** any attendee whose email domain matches `identity.internal_domains` → rep.

Record reps as `(email, display_name)`. Dedupe across calls. Multiple reps on one call all get coaching credit for that call.

### Account grouping

For each call:

- **CRM connected:** call `find_opportunity_by_emails` against the call's external attendees. Take the most recently-touched matching opportunity. The opportunity's account (HubSpot company / Salesforce account) is the grouping key. If no CRM match for a specific call, fall back to email-domain grouping for that call (the report flags which calls used the fallback).
- **CRM absent:** primary external email domain on the call is the grouping key. Display name = `accounts.domain_to_name_overrides[domain]` if set, otherwise the domain itself.

**Output of Step 3** — a graph of `{accounts: [...], reps: [...], calls: [...]}` with bidirectional refs.

**Dry-run stops here.** In `--dry-run`, print the Step 3 output as a preview and exit. Show: total calls matched, calls included vs excluded with reasons, accounts identified, reps identified, CRM source (or "none"). Skip Steps 4–6.

## Step 4 — Deal-health analysis (per account)

**One LLM pass per account.** Each account's full bundle of calls in the window goes to one LLM call producing structured JSON for all universal dimensions and (if configured) framework dimensions.

The LLM input includes:

- Account name + CRM context (if available): deal stage, amount, owner, close date, days in stage.
- All calls' summaries, action items, key questions, topics.
- Transcript excerpts where needed for specific quote evidence.
- The framework-file contents (when configured) inlined into the prompt.

The LLM output schema per account:

```json
{
  "account_id": "...",
  "account_name": "...",
  "health": "green" | "yellow" | "red",
  "universal_dimensions": {
    "stakeholder_breadth": {"count": 4, "delta_vs_prior": "+1", "note": "..."},
    "sentiment_trend": {"direction": "increasing" | "flat" | "decreasing", "note": "..."},
    "urgency_signals": {"score": 0-3, "evidence": [{"quote": "...", "speaker": "...", "timestamp": "...", "call_id": "..."}]},
    "budget_signals": {"score": 0-3, "evidence": [...]},
    "next_step_clarity": {"score": 0-3, "agreed_next_step": "...", "evidence": [...]},
    "days_since_last_touch": <int>,
    "risk_flags": [{"flag": "...", "evidence": [...]}],
    "strength_signals": [{"signal": "...", "evidence": [...]}]
  },
  "framework_dimensions": { /* present only when framework_file is configured */ },
  "crm_enrichment": { /* present only when CRM is connected */
    "deal_stage": "...",
    "amount": <number>,
    "owner": "...",
    "close_date": "YYYY-MM-DD",
    "days_in_stage": <int>,
    "open_opportunities_count": <int>
  }
}
```

The overall `health` is the LLM's holistic judgment across all dimensions. The report renders this as a green/yellow/red badge on the account card.

## Step 5 — Coaching analysis (per rep)

**Per-call LLM pass.** A rep's score on "discovery quality" depends on whether the call was a discovery call; coaching makes sense per-call, then aggregates per-rep.

For each call, for each rep on the call, run one LLM analysis pass.

The LLM input:

- The call's metadata (title, attendees, duration, call type if classifiable).
- The full transcript (tier 2) when available; structured layers otherwise.
- Read AI `metrics` for this rep (speaking time, etc.).
- The user's coaching framework file (when configured) inlined.

The LLM output schema per (call, rep):

```json
{
  "call_id": "...",
  "rep_email": "...",
  "universal_dimensions": {
    "talk_time_ratio": {
      "rep_seconds": <int>,
      "prospect_seconds": <int>,
      "ratio": <float>,
      "appropriate_for_call_type": true | false,
      "note": "..."
    },
    "question_density": {"questions_per_10min": <float>, "examples": [{"quote": "...", "timestamp": "..."}]},
    "discovery_quality": {
      "applicable": true | false,
      "score": 0-3,
      "covered": ["problem", "impact", "urgency", "decision_process"],
      "missed": ["budget"],
      "note": "..."
    },
    "objection_handling": {
      "score": 0-3,
      "examples": [{"objection": "...", "rep_response": "...", "evaluation": "addressed" | "deflected", "timestamp": "...", "suggestion": "..."}]
    },
    "next_step_clarity": {"score": 0-3, "agreed_next_step": "...", "note": "..."},
    "filler_hedging": {"samples": [{"phrase": "I think maybe", "timestamp": "..."}], "note": "..."},
    "listening_signals": {"score": 0-3, "evidence": [...]}
  },
  "framework_dimensions": { /* present only when framework_file is configured */ },
  "headline_strength": "<one line>",
  "headline_improvement": "<one line>"
}
```

**Aggregation per rep across calls:**

- Average numeric scores.
- Pick the 1–2 most useful coaching examples (specific call, specific moment, specific transcript quote, suggested improvement).
- Compute overall talk-time ratio across all the rep's calls.

## Step 6 — Render HTML report

Build the single self-contained file using the template and rendering recipe in `references/html_template.md`. The recipe deliberately uses Bash file concatenation for the Chart.js inlining step rather than reading the 200KB asset through LLM context.

**File layout of the rendered HTML:**

1. `<head>` with title, meta, inline `<style>` block.
2. `<script>` block containing Chart.js (sourced from the skill's `assets/chart.umd.min.js`).
3. `<script type="application/json" id="report-data">` containing the analysis output from Steps 4–5.
4. `<body>` with the four main sections: header / TL;DR, per-account, per-rep, all-calls table.
5. `<script>` block with the vanilla render JS (filter / sort / expand / chart-init).

Write to `./<outputs.reports_dir>/YYYY-MM-DD-sales-call-review.html`, then `chmod` to `outputs.file_mode` (default `0600`).

## Step 7 — Run log

Append one JSON object per run to `./.sales-call-review-runs.jsonl`:

```json
{
  "run_at": "2026-05-22T16:00:00-07:00",
  "mode": "auto",
  "window": ["2026-05-15", "2026-05-22"],
  "calls_matched": 12,
  "calls_analyzed": 11,
  "calls_excluded": [{"id": "...", "title": "Internal sync", "reason": "title_exclude_match"}],
  "accounts": 5,
  "reps": 3,
  "crm_source": "hubspot",
  "report_path": "reports/2026-05-22-sales-call-review.html",
  "cache_hits": 8,
  "errors": []
}
```

`cache_hits` is the count of calls whose analysis was reused from a prior run.

The run log is append-only. Never rewrite it.

## Mode behavior

**Interactive mode** (default for human invocation):

1. Run Step 0 (wizard) if config is missing or invalid.
2. Run Steps 1–6 silently.
3. Show a one-line summary: "Analyzed 14 calls across 5 accounts, 3 reps. Report at `reports/YYYY-MM-DD-sales-call-review.html`."
4. Offer to open the report (`open` on macOS, `xdg-open` on Linux, print path otherwise).

**Auto mode** (scheduled, Friday afternoon):

1. Fail loud if config is missing or invalid. Do **not** start the wizard from a scheduled run.
2. Run Steps 1–7 end-to-end without prompts.
3. No notification side effects — the local HTML file is the only output. The user is expected to know it's there as part of their weekly routine.

**Dry-run** (`--dry-run`):

1. Stop after Step 3.
2. Print preview: counts of calls matched / excluded with reasons, accounts identified, reps identified, CRM source.
3. Skip Steps 4–6 (the expensive analyses + render).
4. Cheapest path to "did my filter / grouping config produce sensible results?"

**Reconfigure** (`--reconfigure`):

Force the wizard against existing config. Preserves identity history but lets the user change CRM source, framework files, exclusion lists, working dir.

**No `--undo` / `--apply` / `--clear-cache` in v1.** The only artifact is a local HTML file the user can delete. The cache file (`.sales-call-review-cache.json`) can be manually deleted if the user wants to force re-analysis.

## Calendar safety

(N/A — the skill does not touch the calendar.)

## Privacy + safety

- **`0600` file mode** enforced on the rendered HTML report. The skill `chmod`s after writing; doesn't rely on umask.
- **Report footer reminder** about file sensitivity on every render.
- **No external network requests in the rendered HTML.** Chart.js is inlined; all data is inlined; no CDN dependencies. Opens fully offline.
- **No PII to external services** beyond what's in Read AI / the configured CRM / the LLM context. No third-party enrichment.
- **Cache at the same trust level as the run log.** Contains analysis results keyed by meeting ID. Does NOT contain raw transcripts (those stay in-process during the LLM analysis call).
- **Never delete or overwrite report files.** Re-runs that produce a report for the same date write `YYYY-MM-DD-sales-call-review-rev2.html` rather than overwriting.

## When this skill should fail loudly vs gracefully

**Fail loudly** (raise/return an error, do not write the report):

- Read AI MCP returns auth error or 5xx.
- A configured CRM MCP returns auth error or 5xx mid-run (treat as broken connection, not "no CRM").
- `--mode auto` and config is missing or invalid.

**Fail gracefully** (degrade, log, still produce a report):

- CRM MCP missing entirely → run in no-CRM mode using email-domain grouping. Note this in the report header and the run log.
- Individual transcript expand fails → analyze that call from structured layers only. Mark affected scores as "structured-data only" in the report.
- LLM returns malformed output for one call → log it, skip the call from per-rep aggregation, surface the failure in the report's "Notes on this run" section.
- Framework file is missing or malformed → fall back to universal-default dimensions only. Log the framework error in the report.

## Reference files

- `references/connector_setup.md` — Wizard-side reference. How to detect each MCP (Read AI, HubSpot, Salesforce) and what to tell the user when one is missing.
- `references/crm_adapters.md` — The pluggable CRM abstraction (`list_users`, `find_opportunity_by_emails`, `list_account_open_opportunities`, etc.). Per-adapter mappings for HubSpot (implemented day one) and Salesforce (implemented day one).
- `references/coaching_framework_example.md` — Template the user copies and adapts to extend the universal coaching dimensions with their own framework (MEDDIC, BANT, SPICED, custom). Fictional placeholder example so the shape is obvious.
- `references/deal_health_framework_example.md` — Template for the deal-health framework. Mirrors the coaching template's structure.
- `references/html_template.md` — The HTML report skeleton + the rendering recipe. Step 6 reads this to build the output. Includes the Bash-based Chart.js inlining pattern to avoid loading 200KB through LLM context.
- `references/output_templates.md` — Run log JSON shape and the minimal "no sales calls in window" report template.

The `assets/chart.umd.min.js` file (~200KB) is the bundled Chart.js library inlined into every rendered report. Sourced from `chart.js@4.4.7` on jsdelivr.
