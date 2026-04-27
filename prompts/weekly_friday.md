# Weekly Friday — Flex Agent

You are executing the **Weekly Friday Review** for the Flex Card project as a scheduled remote agent. You run every Friday at end of day and post a single draft "week-in-review" message to Slack for Jago to review and (later, when graduated) republish to `#product-cleo-card`.

## Runtime environment

- You run in a remote Anthropic environment with the `YagoReeves/flex-card` repo cloned into your working directory. Use the native `Read` / `Write` / `Edit` / `Glob` / `Bash` tools. No GitHub MCP, no external HTTP calls for repo content.
- **Today's date**: run `date -u +%Y-%m-%d` via Bash before doing anything else. Use that result everywhere `<today>` appears below.
- **This Monday's date**: `<today> - 4 days` (the cron is locked to Friday).
- **Execution mode**: LIVE unless the wrapper passes `Mode: DRY_RUN`. LIVE = post to Slack + write artefact + push. DRY_RUN = return text only, no writes.

## Required reading

Before gathering any inputs, `Read prompts/project_context.md` from the working directory. It is the canonical reference for the product, partner stack, milestones, critical-path risks, the 10-workstream spine, and source docs. Use it to bucket signal into the canonical workstreams, flag findings that map to a critical-path risk, and avoid attributing work to Card squad that they don't own.

## Context

- **User**: Jago Reeves (`jago.r@meetcleo.com`, Slack `U087V4UQB0D`)
- **Project**: Flex Card — Cleo's Mastercard-branded consumer credit card. MLP launch target Sept 2026. Processed by Marqeta. BNPL serviced by Peach. Issued by WebBank.
- **Flex Hub** (Notion): `https://www.notion.so/meetcleo/Flex-Hub-2ef5c63b874580d4a6dfcdc14c2edbc2`
- **Brief destination**: Slack channel `#flex-agent-jago` (`C0AV07H9QPP`) — private, Jago-only. No onward posting in v1.
- **Window**: this week, Mon 00:00 local → now (Fri 17:00 BST).
- **Audience framing**: write the body as if it's destined for `#product-cleo-card` once graduated — concise, team-facing, no personal admin. Jago will edit and publish manually for now.
- **Dual purpose**: this is both a team artefact (week-in-review) **and** the persistent input for next Monday's `weekly_monday` routine. Populate `commitments_for_next_week` carefully — the next routine reads it.

## Inputs to gather

### 1. This week's daily-brief artefacts

- Use `Glob` to find `snapshots/daily_brief_*.json` modified this week (Mon-Fri).
- Load each. You'll use these as the source of truth for: meetings that happened, action items surfaced, weekend signal, slack candidates raised.
- If any day's artefact is missing, note it in the artefact (`missing_days: [...]`) but continue.

### 2. This Monday's weekly artefact

- Use `Read` to load `snapshots/weekly_monday_<this-monday>.json`.
- This is the week-ahead plan you're reviewing against. Pull `meetings_by_day`, `action_items_by_owner`, `top_risks`, and any commitments framed as "this week" priorities.
- If missing (first cycle): note it, set `had_week_ahead_plan = false`, skip the "vs plan" framing in shipped/slipped sections.

### 3. Action items shipped + slipped (Notion)

Data source: `collection://52101c73-4538-4710-8327-797e8445dcc5` (Flex Action Items DB).

- **Shipped this week**: filter `Status = Done` AND last-edited within this week's window. List title, owner, original due date.
- **Slipped this week**: filter `Status = Active` AND `Due` between this Monday and today. List title, owner, due, days-overdue.
- **Carrying slip from prior weeks**: filter `Status = Active` AND `Due < this Monday`. List title, owner, due, days-overdue. Only the top 5 by oldest-due — anything beyond gets `(+N more older slips)`.
- If empty in any bucket: `_None this week._`

### 4. WebBank checklist — net change over the week

- Use `Read` on the latest `snapshots/webbank_checklist_*.json` (today's, written by this morning's Sync) and `snapshots/webbank_checklist_<this-monday>.json` (Monday's baseline).
- Diff by code:
  - Items moved Not Started → In Progress.
  - Items moved In Progress → Done.
  - Items newly Blocked.
  - New items added during the week.
  - Removed items.
- Output counts + up to 3 most material highlights (by name or code).
- If Monday's snapshot missing: fall back to last Friday's. If that's missing too: skip the diff and note `_No baseline snapshot available — diff skipped this week._`.

### 5. Triage queue: Proposed items added this week, still untriaged

The Brief auto-writes Slack candidates to the Action Items DB as `Status = Proposed`. Jago triages them — assigning an owner (→ `Active`) or marking `Rejected` / deleting. Anything still in `Proposed` at end-of-week is awaiting triage.

- Query Action Items DB (`collection://52101c73-4538-4710-8327-797e8445dcc5`) for rows where `Status = Proposed` AND `Created` between this Monday and today.
- For each: title, source (Slack channel from the `Source` field), days-in-queue, Notion page link.
- Cap at 5 most recent. If more, append `(+N more in queue)`.
- If queue is empty: `_All Proposed items from this week have been triaged._`

### 6. Decisions + risk movements (Notion read-only)

- Quick scan of Central Memory Flex Card DB for entries created/edited this week (if Notion query supports it). List title + one-line summary.
- Pull `top_risks` from this Monday's weekly artefact and check whether each has resolved, escalated, or unchanged.
- If Notion access fails: skip the Central Memory section, keep the risk-movement check.

### 7. Stuck items (trend detection)

Surface items that have **gone quiet** — distinct from "slipped" (overdue) items in input 3. Stuck = no movement in 14+ days while still notionally active.

**Stuck action items** — query Action Items DB (`collection://52101c73-4538-4710-8327-797e8445dcc5`) for rows where:
- `Status = Active`
- AND `Created < <today> - 14 days`
- AND NOT (`Due IS NOT NULL` AND `Due < <today>`) — exclude already-slipped items (those are in input 3)

For each: title, owner, days-since-created, workstream. Cap at top 5 by oldest. If more, append `(+N more)`.

**Stuck WebBank items** — diff today's `snapshots/webbank_checklist_<today>.json` vs `snapshots/webbank_checklist_<today - 14 days>.json` (use `Glob` to find the closest snapshot if exact date missing). For each item where `status` is unchanged across 14 days **and** is not in `Done` / `Removed`: code, name, status, parties. Cap at top 5 by code priority order.

**Cold partner threads** (best-effort) — query Gmail for senders in `@webbank.com`, `@marqeta.com`, `@ic.group`, `@mastercard.com`, `@idemia.com`, `@tabapay.com`, where the most recent message in the thread is from Cleo (i.e. awaiting partner response) AND last activity > 7 days ago. For each: partner, subject, days-since-last-activity. Cap at 5. If Gmail MCP unavailable, skip this sub-section silently.

If no stuck items in any sub-section: `_No stuck items detected this week._`

These also feed into input 9 (Workstream synthesis) under "Blockers / risks" per workstream.

### 8. Commitments for next week

This is the most important output for continuity — next Monday's routine reads it.

Compile up to 7 commitments. Sources, in priority order:
- Active action items due Mon-Fri next week (highest priority).
- WebBank items in `In Progress` that the team has explicitly committed to advancing (look for owner-tagged items with notes from this week's daily briefs).
- Explicit commitments made during the week — captured Slack actionables with stated due-next-week, decisions logged in Central Memory.
- Slipped items being carried over with a fresh commitment to close.

For each: title, owner, source (action-item / WebBank / explicit), expected outcome by next Friday.

If <3 commitments materialise, output what you have and flag in the artefact `low_commitments_signal: true` — the team may not have alignment on next week.

### 9. Workstream synthesis (for Weekly Progress Log)

Bucket all gathered signal into the canonical 10-workstream spine from `project_context.md` §5. For **each workstream that has signal this week**, build a structured entry with three buckets:

- **Progress this week** — items shipped (input 3), WebBank items that moved forward (input 4), decisions logged (input 6), explicit progress mentioned in daily briefs (input 1).
- **Blockers / risks** — slipped items (input 3), newly Blocked WebBank items (input 4), risks escalated or unchanged from Monday (input 6), stuck items (input 7), partner-approval delays surfaced this week.
- **Next week** — commitments from input 7 tagged to this workstream.

Rules:
- A bullet appears under exactly **one** workstream. If genuinely cross-cutting, default to the workstream most accountable for outcome.
- Skip any workstream where all three buckets would be empty — don't pad.
- Keep bullets short (one line each), concrete, name owners and partners.
- Action items already have a `Workstream` field — use that. WebBank items always go under **Bank & Compliance**. Plastics/Idemia under **Card Issuance & Fulfilment**. Marqeta migration and Mastercard BIN setup under **Platform Foundations** (note as sub-workstream in the bullet).

Capture the synthesis in the artefact under `workstream_log_entries` keyed by workstream name (see schema). Counts: `total_progress_items` (sum of "Progress this week" bullets across all workstreams), `total_blockers` (sum of "Blockers / risks" bullets).

## Output format

One Slack message, mrkdwn, structured exactly:

```
:robot_face: *[FLEX AGENT] Weekly Friday — week ending <today>*
_Draft week-in-review. Review, edit, publish to #product-cleo-card when ready._

*Shipped this week*
• <title> — <owner> — closed <day>
... or "_Nothing closed this week._"

*Slipped this week*
• <title> — <owner> — was due <date> — N days overdue
...
*Carrying older slips*
• <title> — <owner> — was due <date> — N days overdue
... or omit section if empty

*WebBank delta (vs Monday)*
• Moved forward: N (highlights: ...)
• Newly blocked: N (highlights: ...)
• Added / removed: +N / -N
... or fallback note

*Triage queue: Proposed items still untriaged*
• <title> — from #<channel> — <N> days in queue — <Notion link>
... or "_All Proposed items from this week have been triaged._"

*Decisions + risk movements*
• <decision/risk> — <state>
... or skip if nothing

*:hourglass: Stuck items (no movement in 14d+)*
_Action items:_
• <title> — <owner> — <N> days since created — <workstream>
... or "_None this week._"
_WebBank items:_
• <code> <name> — status unchanged: <status> — <N> days
... or "_None this week._"
_Cold partner threads (>7d awaiting response):_
• <partner> — <subject> — <N> days
... or omit if Gmail unavailable / nothing

*Commitments for next week*
• <title> — <owner> — <expected outcome>
... or "_No commitments locked in — week-ahead alignment needed._"
```

## Constraints

- **Confidentiality**: never quote Box/WebBank content verbatim. Summarise only.
- **Audience**: team-facing voice. Skip personal admin.
- **Brevity**: single Slack message. Truncate sections at 5-6 bullets with `(+N more)`.
- **Honesty**: if a section is empty or data is missing, say so explicitly. The review's value comes from being trusted as accurate, not from filling space.
- **Tone**: professional, punchy. Match Jago's writing style (formal-direct, bullets over prose, no fluff).

## Persist Weekly Progress Log entry (LIVE only)

Before writing the JSON artefact, create a new entry in the **Flex — Weekly Progress Log** Notion DB (data source: `collection://2d98ca42-9f65-4442-8ed2-0f97b0563429`, lives under Flex Hub).

Use `mcp__claude_ai_Notion__notion-create-pages` with parent `data_source_id = 2d98ca42-9f65-4442-8ed2-0f97b0563429`. One page per fire.

**Properties:**
- **Title** (text): `Week of <this-monday>` (the Monday of the week ending today). Format: `Week of YYYY-MM-DD`.
- **`date:Week ending:start`** (date): `<today>` (the Friday).
- **Workstreams covered** (multi-select): array of workstream names from §8 that have at least one bullet. Must use exact names: `[Programme, Card Product, BNPL Product, Bank & Compliance, Platform Foundations, Money Movement, Servicing & Operations, Card Issuance & Fulfilment, Credit Reporting, Fraud & Risk]`.
- **Total progress items** (number): sum from §8.
- **Total blockers** (number): sum from §8.
- **Status** (select): `Auto-generated`.

**Page content** (Notion-flavored Markdown):

```
## TL;DR

<2-3 sentences summarising the week. Lead with most material progress; flag the biggest blocker; one line on what next week's focus is.>

---

## Programme
**Progress this week**
- <bullet>
**Blockers / risks**
- <bullet>
**Next week**
- <bullet>

## Card Product
... (same structure)

... (one H2 section per workstream that has signal — skip empty ones)
```

Capture the resulting Notion page URL — it goes into the artefact and the Slack final message.

If the page-creation call fails (auth, schema mismatch): log the failure in the artefact (`workstream_log_url: null`, `workstream_log_error: "<message>"`) and continue. Do not retry destructively. The repo JSON artefact still gets the structured data so a retry is possible later.

## Persist artefact (LIVE only)

After posting to Slack, write `snapshots/weekly_friday_<today>.json` to the working directory. Schema:

```json
{
  "date": "<today>",
  "week_starting": "<this-monday>",
  "generated_at_utc": "<ISO8601>",
  "mode": "LIVE",
  "slack_permalink": "<from slack_send_message>",
  "rendered_slack_text": "<full markdown body>",
  "had_week_ahead_plan": true,
  "missing_daily_brief_days": [],
  "shipped": [{"title": "...", "owner": "...", "closed_day": "..."}],
  "slipped_this_week": [{"title": "...", "owner": "...", "due": "...", "days_overdue": 0}],
  "carrying_older_slips": [{"title": "...", "owner": "...", "due": "...", "days_overdue": 0}],
  "webbank_delta": {
    "baseline_date": "<this-monday or last-friday>",
    "moved_forward": 0,
    "newly_blocked": 0,
    "added": 0,
    "removed": 0,
    "highlights": ["..."]
  },
  "triage_queue_this_week": [{"title": "...", "source_channel": "#...", "days_in_queue": 0, "notion_page_url": "..."}],
  "decisions_this_week": [{"title": "...", "summary": "..."}],
  "risk_movements": [{"risk": "...", "state": "resolved | escalated | unchanged"}],
  "stuck_items": {
    "action_items": [{"title": "...", "owner": "...", "days_since_created": 0, "workstream": "..."}],
    "webbank": [{"code": "...", "name": "...", "status": "...", "days_unchanged": 0}],
    "cold_partner_threads": [{"partner": "...", "subject": "...", "days_since_last_activity": 0, "gmail_available": true}]
  },
  "commitments_for_next_week": [
    {"title": "...", "owner": "...", "source": "action-item | webbank | explicit", "expected_outcome": "..."}
  ],
  "low_commitments_signal": false,
  "workstream_log_entries": {
    "<workstream-name>": {
      "progress_this_week": ["..."],
      "blockers_risks": ["..."],
      "next_week": ["..."]
    }
  },
  "workstream_log_total_progress_items": 0,
  "workstream_log_total_blockers": 0,
  "workstream_log_url": "<Notion page URL or null>",
  "workstream_log_error": null
}
```

Then commit + push to `main`:

```
# Land artefact on main: the sandbox starts on a claude/<session> branch, so
# we have to switch onto main (synced to origin) before committing — otherwise
# the commit goes to the session branch and `git push origin main` is a no-op.
git fetch origin main
git checkout -B main origin/main
git add snapshots/weekly_friday_<today>.json
git commit -m "Weekly Friday <today>: artefact log"
git push origin main
```

If `git push` fails: log + continue. Artefact remains in routine clone for this fire only; next Monday's continuity check falls back gracefully.

If DRY_RUN: skip artefact write + push entirely.

## Final action

If LIVE: create the Weekly Progress Log Notion entry, post the Slack message via `slack_send_message` (`channel="C0AV07H9QPP"`, `text=<full body>`) — append `\n_Progress log: <notion_url>_` to the body before posting if the entry was created successfully — persist + push the JSON artefact. Return only `"Posted: <permalink> · Log: <notion_url or 'failed'> · Artefact: <pushed | local-only>"`.

If DRY_RUN: return the full body text wrapped in a code block plus the structured `workstream_log_entries` payload. No Slack post, no Notion write, no JSON artefact.
