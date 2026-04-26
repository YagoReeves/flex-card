# Flex Agent — Spec (v1)

**Last updated**: 2026-04-26 · **Owner**: Jago Reeves · **Status**: Daily Brief + WebBank Sync routines live; WebBank Mirror DB reconciled to canonical 94 rows after seed-parser bug recovery; Mon 27 Apr is first natural fire of new parser + push validation; Job 4 (Weekly Cycle) prompts written, routines pending creation in UI

## Pick up here next session

**What's live and validated:**
- Daily Brief routine `trig_0168M4zWyESpqBw57f9PDXfp` — fires `30 6 * * 1-5` (07:30 BST nominal; `next_run_at` shows ~07:36 BST due to scheduler jitter). Re-cronned via API 2026-04-26.
- WebBank Sync routine `trig_01V1a8NQ1SayN7kpnq1Q7Eh2` — fires `0 6 * * 1-5` (07:00 BST nominal; ~07:07 actual). **Parser rewritten 2026-04-26** (handles multi-line cells, parties from Excel col 20 with canonical→Other mapping, upsert-by-code on Notion writes, sanity-check abort if <80 rows). Validated locally against 24 Apr cached + 26 Apr live Box outputs (94 rows each, matches openpyxl truth code-for-code). End-to-end remote validation deferred to Mon's natural fire because idempotent-exit blocks today's test. Re-cronned via API 2026-04-26.
- Weekly Monday routine `trig_01CdWfPUFDnn2Eg64SruRszZ` — fires `30 7 * * 1` (08:30 BST nominal; ~08:36 actual). Created via API 2026-04-26 with `allow_unrestricted_git_push: true` set at creation. Reads `prompts/weekly_monday.md`. MCPs: Slack, Notion, Google-Calendar, Gmail.
- Weekly Friday routine `trig_015kvdNZvkNCL8CFgJP4299X` — fires `0 16 * * 5` (17:00 BST nominal; ~17:01 actual). Created via API 2026-04-26 with push toggle at creation. Reads `prompts/weekly_friday.md`. MCPs: Slack, Notion.
- **WebBank Mirror DB at canonical 94 rows** (was 80 from buggy seed → 82 after routine added DP11.1/.2 → 94 after manual recovery on 2026-04-26 added the 12 rows the seed missed: AP4, AQ5, BSA17–20, DD2, DP2, OR2, SV1, UW5, UW11). All have correct WebBank Status + Party.
- **26 Apr snapshot committed** (`snapshots/webbank_checklist_2026-04-26.json`) with all 94 rows, parties populated. This is the baseline Mon's fire diffs against.
- DST reminder routine `trig_01EreY3zZT6e7sVapd6g9LVU` — fires Mon 19 Oct 2026.

**Next session priorities (in order):**

1. **Verify Mon 27 Apr ~07:07 BST WebBank Sync fire** — first natural fire of the new parser + the toggle-cycle 403 fix attempt. Expected: 94 rows parsed, ~0 changes vs 26 Apr snapshot, **`git push origin main` succeeds**, Slack digest posts, Mirror DB unchanged. If push still 403s: the Sync routine was *updated* (not recreated) so push toggle was set post-creation per the original failure mode — disable the existing trigger and recreate via API with toggle at creation time. (Weekly Mon + Fri were created with toggle at creation, so are the canonical test of whether create-time toggle setting fixes the 403.)
2. **Verify Mon 27 Apr ~07:36 BST Daily Brief fire** — first natural fire post re-cron. Should reference today's freshly-pushed WebBank diff and write `snapshots/daily_brief_<today>.json` (new artefact step added 2026-04-26). Brief was also *updated* not recreated, so its push behaviour follows Sync's. Confirm Slack post lands cleanly + JSON artefact pushes to `main`.
3. **Verify Mon 27 Apr ~08:36 BST Weekly Monday fire** — first natural fire of `weekly_monday`. Should read the Daily Brief artefact (or fall back to its own scan if Brief's push 403'd) and produce a draft week-ahead. First-cycle behaviour expected: `had_week_ahead_plan = false`, no carry-over from prior Friday. **This is the canonical test of create-time push-toggle setting** — if Weekly Monday's push to `snapshots/weekly_monday_<today>.json` succeeds while Sync/Brief 403, the working hypothesis is confirmed and we should recreate Sync + Brief via the same API path.
4. **Verify Fri 1 May ~17:01 BST Weekly Friday fire** — first cycle. Reads daily-brief artefacts written Mon-Fri this week + this week's Weekly Monday artefact. Writes `commitments_for_next_week` for Mon 4 May's continuity loop.
5. **Build Job 2 — Meeting Action Item Extractor**. Manual-trigger routine (Jago kicks off after a meeting). Reads Granola transcript for the named meeting, extracts candidate action items with strict rules, posts draft to `#flex-agent-jago`. Approval flow per spec §"Job 2".
6. **End-to-end workflow verification** after one full Mon-Fri cycle of Brief + Sync + both Weekly routines. Capture friction, missed signals, false positives.

**Open for Jago (manual, anytime):**
- (In Notion UI) group the two Agent-owned DBs into a toggle section under Flex Hub's Central Memory — Notion API was blocking the auto-insert.

**Audit backlog (post-2026-04-26 strategic review):**

*High leverage — build next:*
1. **Job 2 — Meeting Action Item Extractor** (manual trigger). Reads Granola transcript for a named meeting, extracts action items + decisions with strict rules, auto-writes Proposed rows to Action Items DB (mirror Brief's pattern). Highest functional uplift remaining; fills the 'extract action items without effort' goal that's currently leaning entirely on Slack scanning.
2. **Heartbeat / failure-detection routine.** Daily ~09:30 BST check — reads today's `daily_brief_<date>.json` + `webbank_diff_<date>.json`; if either missing, posts `:rotating_light: Routine failure detected` to `#flex-agent-jago`. Cheap insurance against silent failures (current biggest unaddressed risk).
3. **Trend detection in Weekly Friday.** Compare this Friday vs last Friday. Flag stuck items: action items In-Progress > 14 days, WebBank items unchanged > 14 days, partner threads with no inbound > 7 days. Turns review from 'what happened' to 'where are we stuck'.
4. **Granola summaries in Daily Brief prep pointers.** Currently mentioned but not actually pulled — wire 1-line summary of most recent prior transcript per recurring meeting. Prompt edit only, no new routine.
5. **OOO/holiday suppression.** `config/skip_dates.json` in repo; each routine reads as first step and exits cleanly if today's listed.

*First-week calibration (after Mon 27 Apr – Fri 1 May fires):*
6. **Calibrate Slack candidate extraction.** Review Proposed queue: false-positive rate (deleted items), missed commitments, dedup over/under-aggressiveness. Tune strict criteria + dedup heuristics.
7. **Audit Calendar filter coverage.** Likely false-negatives — meetings without 'Flex' in title. Compare actual Flex meetings vs what the Brief surfaced; expand keyword/attendee-domain list.

*Lower priority — revisit when scale demands:*
8. **KPI tracking** — capture rate, false-positive rate, triage latency, routine fire success rate. Likely a Friday aggregate or monthly routine.
9. **Escalation path for urgent items** — distinguish routine from urgent partner signal. Options: marked-priority section, Slack DM, or `:rotating_light:` keyword-triggered routine.
10. **Notion retry / partial-write handling.** Current prompt has no recovery if Notion is flaky mid-Brief and only 3 of 5 candidates write. Options: all-or-nothing transactional, idempotency key per candidate, or surface partial state in artefact.

*Identified but not actioned (low leverage at current scale):*
- Notification fatigue calibration (3-5 messages/day in `#flex-agent-jago` — monitor signal:noise; calibrate by tuning extraction strictness + content density).
- Snapshot directory growth (revisit when >100 files, ~5 months out).
- Compliance audit logging (theoretical — disproportionate effort vs current scale).
- Multi-user contingency / handoff (not relevant — single-operator workflow).
- MCP credential rotation lifecycle (unknown; investigate when first auth issue surfaces).
- Prompt staging/canary (no A/B for routine prompts; bad edit affects all next fires).

**Deferred (revisit triggers noted):**
- **2026-05-08**: human-in-loop graduation review. Brief is auto-post + auto-writes Slack candidates as `Proposed` to Action Items DB (Jago triages in Notion). Sync is auto-write (option-c locked). Weekly Mon + Fri are draft-only to `#flex-agent-jago` (Jago manually edits + publishes onward to `#product-cleo-card`). Review evaluates whether Weekly outputs can publish directly to `#product-cleo-card`.
- **Approval Sweep — obsoleted.** Original design had a 15-min sweep reading Slack thread replies for `capture N` / `approve` / `reject`. Brief now auto-writes (no approval needed) and Weekly drafts are manually edited+published by Jago. Only Job 2 (Action Items extractor, manual trigger) might want it — evaluate when Job 2 is built. If kept simple (extractor also auto-writes Proposed rows), no sweep ever needed.
- **Dashboard** — revisit after 4 weeks of usage. Possible Claude-Code-built web app in this repo.
- **AD1–AD6 placeholder rows** — currently dropped by parser (empty name in Excel). Bring them in once WebBank populates a deliverable; parser will auto-include them on next fire.

## TL;DR

An AI-assisted PMO workflow for the Flex Card launch (UAT target Sept 2026), built on scheduled Claude agents. Four cron jobs produce daily/weekly briefs, action-item extraction, and a WebBank checklist mirror. All publishing is human-gated via Slack threads in `#flex-agent-jago`, processed by a 15-min approval sweep. Canonical source-of-truth stays in the Flex Hub Notion workspace; the agent writes only to a purpose-built Action Items DB and a WebBank Mirror DB. No custom dashboard in v1.

## Goals

1. Extract, assign, and chase action items across meetings, Slack, and email without manual effort.
2. Maintain a live internal mirror of the WebBank implementation checklist with daily diffs.
3. Surface partner inbox signal (email + Slack) fast, without watching every channel.
4. Produce consistent Monday/Friday cycle updates.

Non-goals for v1: custom dashboard, conversational Slack bot with its own identity, writes into existing human-owned Notion DBs (Central Memory, Timelines, Key Decision Log).

## Principles

- **Human-in-loop by default.** Every outbound Slack post and Notion DB write is a draft until Jago approves. No graduation to auto-send without explicit sign-off. Reminder to revisit: **2026-05-08**.
- **Slack is the review surface.** All drafts land in `#flex-agent-jago`. No Notion review pages.
- **Notion hub stays human-curated.** The agent writes only to two new DBs it owns (Action Items, WebBank Mirror). Existing DBs (Central Memory, Timelines, Meeting Notes, Key Decision Log, Banking Ops Checklist) are read-only from the agent's perspective.
- **Confidential content stays confidential.** Box/WebBank content summarised in-session only — never copied verbatim into Slack, Gmail, or unrestricted Notion pages.
- **Stateless runs, shared state.** Scheduled jobs persist nothing between runs; all state lives in Notion + a local `staging/` folder.

## Architecture

**Two faces of one agent:**
- **Cron agent** (via `/schedule` → CronCreate). Runs unattended. Each fire is a fresh Claude session with MCP access (Notion, Slack, Box, Gmail, Granola, Google Calendar).
- **Interactive session** (Claude in terminal). Same MCP access. Used for ad-hoc questions, investigations, and occasional manual-trigger commands (e.g. post-meeting action-item extraction).

**Shared state:**
- **GitHub repo** `https://github.com/YagoReeves/flex-card` (private, owner `YagoReeves`) — single source of truth for prompts, spec, and WebBank snapshot history. Both production routines have it attached as a working-dir source with `allow_unrestricted_git_push: true`, so they read prompts via native `Read` tool and the Sync `git push`es daily snapshot/diff JSONs to `main` directly. Local clone at `/Users/jago.r/Documents/flex_card/`.
- **Notion** — Flex Hub workspace; Action Items DB + WebBank Mirror DB owned by the agent. Mirror DB is the live queryable state; Notion MCP is the routine's read/write surface for it.
- **Local-only directories** (not used by routines, kept for human-side reference): `staging/` holds the original 2026-04-24 seed parse artifacts.
- **Slack** — `#flex-agent-jago` (private channel) as the draft/review surface and the Sync's daily digest destination.

**Identity note:** scheduled agent posts to Slack *as Jago* via the user-auth MCP. Agent posts are distinguished by a marker prefix (`🤖 [FLEX AGENT] …`) and/or fenced code blocks, not by sender. A true bot identity is a v2 consideration.

## The four scheduled jobs

### Job 1 — Daily Morning Brief
- **When**: 09:00 Mon-Fri.
- **Output**: Posted to `#flex-agent-jago`. The Brief itself needs no approval; it enables a capture flow for Slack-originated action items (see below).
- **Inputs**:
  - WebBank checklist diff highlights (from Job 3 snapshot).
  - Action Items DB: overdue + due today.
  - Priority inbox hits since yesterday's brief (email domains + Slack channels — see Priority Inbox below).
  - Today's Flex-relevant meetings from Google Calendar with prep pointers (Granola history, Central Memory, linked Notion pages).
  - **Candidate Slack action items**: top 5 explicit commitments spotted in watched channels since last Brief. Strict extraction criteria — `I'll do X by Y`, verb + deliverable `@`-mentions, any agreed emoji/prefix conventions. Each listed with numeric index + thread link.
- **Format**: single message in `#flex-agent-jago`, short headers, links over long quotes.
- **Capture flow**: reply in thread with `capture 1, 3` to flip specified candidates into the Action Items DB draft pipeline (processed by the approval sweep like any other draft). Unreplied candidates expire after 24h; anything not captured by Friday rolls into the Week-in-Review list for a last chance.

### Job 2 — Meeting Action Item Extraction
- **When**: manual `/run` after any Flex-relevant meeting. Triggered by Jago in an interactive session.
- **Scope**: any meeting in Granola — not limited to the 3 weeklies. Slack-originated action items are handled via the Daily Brief capture flow, **not** here, to avoid duplicating the Slack scan.
- **Flow**:
  1. Pull Granola transcript for the specified meeting.
  2. Extract candidate action items (title, owner if stated, due date if stated, source quote).
  3. Flag any action item missing a clear owner — explicit ownership-gap surface.
  4. Post draft table to `#flex-agent-jago` in a new thread: `🤖 [FLEX AGENT] Draft action items — <meeting> <date>`.
  5. On approval (`approve` / `approve with edits: …`), sweep writes to Flex Action Items DB + posts summary to the appropriate project Slack channel tagging owners.

### Job 3 — WebBank Checklist Daily Sync
- **When**: 08:00 Mon-Fri (runs before Daily Brief so the brief can reference fresh diff).
- **Source**: WebBank checklist Excel file in Box, at `https://app.box.com/file/2101567683726` (stable).
- **Flow**:
  1. Download the Excel file from Box.
  2. Parse checklist rows.
  3. Diff against yesterday's snapshot in `snapshots/webbank_checklist_YYYY-MM-DD.json`.
  4. Produce diff summary: new items, status changes, new notes, removed items.
  5. Post draft diff digest to `#flex-agent-jago`: `🤖 [FLEX AGENT] Draft WebBank digest — <date>`.
  6. On approval, sweep posts digest to `#product-cleo-card` and updates WebBank Mirror DB in Notion.
  7. Save today's parse as the new snapshot.
- **Confidentiality**: Excel contents summarised only. Raw cell contents never re-posted to Slack or Notion verbatim beyond status/label-level detail needed for diff clarity.

### Job 4 — Weekly Cycle
- **Monday 09:00** — Week-ahead summary: upcoming Flex deliverables, at-risk items, priority meetings of the week, outstanding action items by owner. Draft to `#flex-agent-jago`; on approval, published to Flex Hub + relevant Slack channel.
- **Friday 17:00** — Week-in-review: what shipped, what slipped, WebBank checklist net change over the week, action items closed / still open. Same approval flow.

## Approval flow

### Slack draft pattern (for posts destined for project channels)
Initially: agent DMs Jago the draft text; Jago edits and sends to the destination channel manually. When graduated (post-2026-05-08 review), agent posts directly to destination with no sweep needed.

### Notion DB writes (for Action Items DB + WebBank Mirror DB)
Thread-based approval via `#flex-agent-jago`:

1. Job posts a draft to a new thread, prefixed `🤖 [FLEX AGENT]`, with structured proposed rows.
2. Draft payload also staged as JSON in `staging/<YYYY-MM-DD>_<job>.json`.
3. **Approval sweep cron** runs every 15 min during working hours (Mon-Fri, 09:00-19:00):
   - Reads recent thread replies from Jago in `#flex-agent-jago`.
   - Matches replies to pending staging files by thread reference.
   - `approve` → executes the Notion write, posts confirmation in thread.
   - `approve with edits: <edits>` → applies edits, executes, confirms.
   - `reject` → discards staging file, logs reason.
   - `capture <indices>` on a Daily Brief thread → flips the specified candidate Slack action items into a staging draft, then processes it as a normal action item write on the next sweep tick.
   - No reply within 48h → flags in next Daily Brief as a reminder.

## Notion structure

### Existing (read-only from agent)
- **Flex Hub** — `https://www.notion.so/meetcleo/Flex-Hub-2ef5c63b874580d4a6dfcdc14c2edbc2`
- **Central Memory Flex Card DB** — decisions/risks/issues log, Jago-curated.
- **Flex Card Timelines DB** — human-owned.
- **Meeting Notes DBs** (Partner Meetings + Card x Payments) — referenced when extracting action items.
- **Key Decision Log DB** — read-only.
- **Flex Card Implementation Checklist (Banking Ops) DB** — explicitly not used by agent per user decision 2026-04-24.

### New (agent-owned, to be created in build phase)
- **Flex Action Items DB**
  - Columns: Title · Owner · Source (meeting/Slack/email/link) · Status (Proposed/Active/Done/Blocked/Rejected) · Due · Created · Last nudged · Priority · Notes
- **WebBank Mirror DB**
  - Columns: TBD — inspect Excel structure first, then propose a column subset + Jago-added custom columns (per user decision 2026-04-24: not a direct mirror, selected columns only)

## Priority inbox

### Email (Gmail MCP)
- `@webbank.com`
- `@marqeta.com`
- `@ic.group` (IC Group — physical card provider)
- `@mastercard.com`
- *(list grows over time — add new partners here as they engage)*

Surfaced in Daily Brief; high-priority hits (e.g. unanswered > 24h from webbank.com) also flagged in Slack DM ad-hoc.

### Slack (Slack MCP)
Watched channels:
- `#product-cleo-card` (primary)
- `#collab-card-accourt`
- `#proj-flex-card-design`
- `#collab-card_ops_support`
- `#collab-compliance-credit`
- *(list grows as new channels spin up)*

Plus: any `@`-mention of Jago anywhere Flex-adjacent.

Unread messages in watched channels summarised in Daily Brief. Direct mentions escalated to self-DM.

## Build sequence

### Phase 0 — Setup
- [x] Fix CLAUDE.md typo (Marketta → Marqeta)
- [x] Write this spec
- [x] Write memory pointer
- [x] Create `staging/` and `snapshots/` directories in `/Users/jago.r/Documents/flex_card/`
- [x] Create private Slack channel `#flex-agent-jago`

### Phase 1 — Foundations
- [x] Inspect WebBank Excel structure via Box MCP (text extraction works despite `canDownload: false`)
- [x] Create **Flex Action Items DB** in Notion — `https://www.notion.so/ca353d70ea9b417ca10c10fd1c391995` · data source `collection://52101c73-4538-4710-8327-797e8445dcc5`
- [x] Create **WebBank Mirror DB** in Notion — `https://www.notion.so/f534187e1f12499bbad85bb9e52b6740` · data source `collection://a31ff0cd-3aaa-434a-815d-0c8dcc8ba53f`
- [ ] **Manual (Jago, in Notion UI)**: group the two DBs under a new "Agent-owned DBs" toggle section below Central Memory on the Flex Hub page. Notion API validator blocked the automated edit.
- [x] Build **Daily Morning Brief** routine — live end-to-end. Cron `53 7 * * 1-5` (07:53 UTC = 08:53 Europe/London BST). Routine ID `trig_0168M4zWyESpqBw57f9PDXfp`. Model: `claude-sonnet-4-6`. MCP connections attached: Slack, Notion, Granola, Google-Calendar, Gmail. Repo source attached: `YagoReeves/flex-card` with `allow_unrestricted_git_push: true`. Wrapper-pattern prompt (Reads `prompts/daily_brief.md` from working dir).
- [x] One-shot DST reminder routine to bump crons pre-GMT switch — `trig_01EreY3zZT6e7sVapd6g9LVU`, fires Mon 19 Oct 2026 (UK switches BST→GMT Sun 25 Oct). Reminds Jago to update both routine crons by +1 UTC hour.
- [~] **Approval Sweep** — obsoleted 2026-04-26. Brief now auto-writes Slack candidates as `Status = Proposed` rows in the Action Items DB; Jago triages (assigns owner → `Active`, or deletes/`Rejected`). Weekly drafts are manually edited+published by Jago. Only Job 2 (Meeting Action Item Extractor) might revisit — likely also auto-writes Proposed rows when built.

### Routine management

- **Wrapper-prompt pattern** (adopted 2026-04-25): both production routines carry a tiny ~6-line wrapper that fetches the canonical prompt from `prompts/<file>.md` in the attached repo's working directory. To change a prompt, edit the file in the repo, commit, push to `main`. Next routine fire automatically picks up the new version — no `RemoteTrigger` update needed.
- **Caveats on `RemoteTrigger` PUTs**: when calling `update`, send the **full** `session_context` object (including `sources` with `allow_unrestricted_git_push: true`). Partial updates replace the object whole and clobber attached repo state. Always `RemoteTrigger get` first if you're unsure of the current shape.
- All routines visible + editable at https://claude.ai/code/routines

### Phase 2 — Core jobs
- [x] **WebBank Mirror DB initial population** — seeded 2026-04-24 (80 rows; buggy due to seed-parser multi-line-cell handling). Reconciled to canonical **94 rows** on 2026-04-26 via local openpyxl parse + Notion MCP backfill of the 12 missed codes + 4 carryover patches. See 2026-04-26 change log entries for full recovery.
- [x] **WebBank Checklist Daily Sync** routine — live with rewritten parser (2026-04-26). Cron `7 7 * * 1-5` (08:07 BST). Routine ID `trig_01V1a8NQ1SayN7kpnq1Q7Eh2`. Model: `claude-sonnet-4-6`. MCP connections: Slack, Notion, Box. Repo source: `YagoReeves/flex-card` with `allow_unrestricted_git_push: true`. Design: option-c (auto-write DB + digest to `#flex-agent-jago` only, no `#product-cleo-card`, no approval gate). On fire: Read prior snapshot from working dir, parse Excel via Box MCP using the robust parser embedded in `prompts/webbank_sync.md`, diff, upsert Mirror DB by code, write today's snapshot+diff JSONs, `git push origin main`, post digest. **Mon 27 Apr is first natural fire of the new parser + first push attempt since the 403 toggle-cycle fix attempt.**
- [ ] Build **Post-Meeting Action Item Extractor** (manual trigger)
- [x] Build **Weekly Cycle** — prompts + routines live 2026-04-26. Weekly Monday `trig_01CdWfPUFDnn2Eg64SruRszZ` (cron `30 7 * * 1`), Weekly Friday `trig_015kvdNZvkNCL8CFgJP4299X` (cron `0 16 * * 5`). Both created via API with `allow_unrestricted_git_push: true` set at creation (canonical 403-fix path). Daily Brief gained `snapshots/daily_brief_<date>.json` artefact-write step. Friday writes `commitments_for_next_week` for Mon's continuity loop. Audience framing: drafts in `#flex-agent-jago` only for v1; Jago manually publishes onward to `#product-cleo-card` post-edit. Pending end-to-end validation Mon 27 Apr (Weekly Monday) + Fri 1 May (Weekly Friday).

### Phase 3 — Iteration
- [ ] **2026-05-08**: review human-in-loop graduation — which flows move to auto-send?
- [ ] 4-week usage review: is a dashboard worth building as a learning project in this folder?
- [ ] Expand Granola coverage beyond the 3 weekly meetings if extraction reliability proven

## Open items / deferred

- **Dashboard**: deferred; revisit after 4 weeks of usage. Possible Claude-Code-built web app in this folder as learning project.
- **True agentic auto-send**: gated on 2026-05-08 review.
- **Webhook approval trigger** (instant rather than 15-min polled): only if 15-min sweep proves too slow.
- **Standalone Slack bot identity**: only if self-DM/self-post model causes confusion.
- **Akiflow integration**: no MCP available; not in scope.

## Change log

- **2026-04-24** — Initial spec written after scoping interview with Jago. Design locked; build pending.
- **2026-04-24** — Job 2 broadened to any Granola meeting (not just the 3 weeklies) and renamed to "Meeting Action Item Extraction". Standalone Slack sweep rejected as overlapping with Daily Brief; instead, Brief now includes a "Candidate Slack action items" section with a `capture N` reply flow. Daily Brief destination moved from self-DM to `#flex-agent-jago` to unify the review surface.
- **2026-04-24** — Both Notion DBs created under Flex Hub (Flex Action Items, WebBank Mirror). Hub-toggle integration blocked by Notion API validator, deferred as a manual Jago step in Notion UI.
- **2026-04-24** — Daily Brief prompt written at `prompts/daily_brief.md`. Two dry-runs + one live post to `#flex-agent-jago` ([permalink](https://cleo-team.slack.com/archives/C0AV07H9QPP/p1777047912553659)). Prompt iterated on: candidate actionables rephrased as imperatives (not quotes); BNPL/Pay Later/Peach meetings excluded; OOO auto-responders excluded; `#collab-compliance-credit` added to watched channels.
- **2026-04-24** — Switched scheduling mechanism from local CronCreate (session-bound, 7-day expiry) to remote `/schedule` routines (cloud-hosted, persistent). Daily Brief + DST reminder routines created. Remote routines have 1-hour minimum — blocks the spec's 15-min approval sweep; deferred with design-revisit flag.
- **2026-04-24** — WebBank Mirror initial population: 80 rows seeded. Fingerprint check on Excel header row passed. Parser handles 22 category codes; schema extended to include `DD`, `DP`, `IP`, `Other` after detection of missing options. 15 pages initially remapped to `Other` then corrected to `DD`/`DP` post-schema-update.
- **2026-04-25** — Migrated project to GitHub: created private repo `YagoReeves/flex-card`, set up SSH auth on Mac, pushed initial commit. Repo holds spec, prompts, and snapshot history.
- **2026-04-25** — Adopted **wrapper-prompt + repo-attached-source** pattern. Investigated three alternatives: (a) hosted GitHub MCP — not available in routine connector picker; (b) GitHub PAT via `curl` + env var — rejected in favour of (c); (c) Anthropic-managed git source attachment with "Allow unrestricted branch pushes" toggle — selected as cleanest. Both production routines now have `YagoReeves/flex-card` attached as a working-dir source; native `Read`/`Write`/`Edit`/`Bash` tools handle all file ops; Sync `git push`es to main directly each fire.
- **2026-04-25** — WebBank Sync routine `trig_01V1a8NQ1SayN7kpnq1Q7Eh2` created with cron `7 7 * * 1-5`. Mirror DB confirmed as live runtime state (queried via Notion MCP); repo holds point-in-time snapshot history (immutable JSON commits to `main`). Dry-run successful — first live fire scheduled Mon 27 Apr 08:07 BST.
- **2026-04-25** — Daily Brief routine refactored to wrapper pattern: prompt now reads `prompts/daily_brief.md` from working-dir clone instead of carrying an inlined copy. WebBank diff slice updated to read `snapshots/webbank_diff_<today>.json` from working dir.
- **2026-04-25** — DST reminder routine updated to remind on bumping both routine crons (Brief and Sync) by +1 UTC hour at the BST→GMT switch.
- **2026-04-26** — **WebBank Sync first live fire (manual trigger) revealed two compounding bugs in the seed.** (1) Parser bug: routine produced 82 rows (80 seed carryover + DP11.1 + DP11.2) but the canonical truth from openpyxl on the binary `.xlsm` is 94 rows. Box MCP returns tab-separated text where multi-line cells (e.g. AQ5, AP4, UW11, UW12 Bank Expectation) split across lines — the naive line-by-line parser the model wrote each fire silently dropped 14 real rows because the trailing cells (Status, Notes, etc.) landed on continuation lines, not the row-start line. (2) Push bug: `git push origin main` to the routine-runtime proxy at `127.0.0.1:35923` returned 403 (`Permission to YagoReeves/flex-card.git denied to YagoReeves`) despite `allow_unrestricted_git_push: true` being set. Working hypothesis: toggle was added via API update *after* trigger creation, underlying GitHub-app credential never refreshed with broader push scope.
- **2026-04-26** — **Recovery + parser rewrite.** Local `openpyxl` parse of the `.xlsm` produced the canonical 94-row baseline. Locally regenerated `snapshots/webbank_checklist_2026-04-26.json` and `webbank_diff_2026-04-26.json`, committed + pushed via SSH from this terminal (push works fine outside the routine env). Used Notion MCP to backfill the 12 missing rows in the Mirror DB (AP4, AQ5, BSA17–20, DD2, DP2, OR2, SV1, UW5, UW11), populated WebBank Status="Not Started" + Party from Excel col 20 with canonical→Other mapping. Patched 4 carryover rows (CM4, OR8, WB1, WB2) where the new mapping diverged from the seed's quirky behaviour (substring-match on "SP - Card Ops" → "Ops"; silent drop of "R&A"). Mirror DB now at canonical 94 rows.
- **2026-04-26** — **Rewrote `prompts/webbank_sync.md` parser**. Detects row-start lines via regex `^\t\t[^\t]+\t[^\t]+\t`, coalesces continuation lines back into the parent row, then `'\n'.join(...)` + `.split('\t')` to preserve multi-line cell content while keeping trailing cells aligned. Header-row filter (`phase=='Phase'` or `code=='Process Item'`). Drops empty-name rows (per Jago — excludes AD1–AD6 placeholders that have codes but no deliverable yet). Parties parsing from col 20 (`Required Reviews`) with canonical→Other mapping (canonical: SP, Compliance, Credit, BSA, Legal, Ops, Product). Switched Notion writes from blind-create to **upsert-by-code** on the added branch (prevents duplicates if any future snapshot drift makes a code look "new"). Added sanity-check abort: if parse returns <80 rows, post a Slack failure digest and skip all writes/commits. Validated against both 24 Apr cached and 26 Apr live Box outputs (94 rows each, matches openpyxl truth code-for-code).
- **2026-04-26** — Push 403 attempted-fix: toggled "Allow unrestricted branch pushes" off → save → on → save in routines UI at 16:15Z. Triggered an `updated_at` change + populated a new `outcomes.git_repository.git_info.branches` field, which is mildly suggestive of a credential refresh. Mon 27 Apr 08:07 BST natural fire is the validation. **If push still fails on Mon: delete + recreate the trigger with toggle set at creation time.** Prompt's existing failure handling means Mirror DB updates apply even if push fails — only snapshot history advancement is at risk.
- **2026-04-26** — `RemoteTrigger update` API gotcha noted: v1↔v2 schema translator gives contradictory errors (`events[0]: event_type is required` vs `unknown field "event_type"`). Wrapper changes via API are unreliable; use the routines UI instead. `RemoteTrigger run` body `message` field is silently ignored — per-run override pattern doesn't work.
- **2026-04-26** — **Job 4 (Weekly Cycle) prompts written.** `prompts/weekly_monday.md` (Mon 08:30 BST) and `prompts/weekly_friday.md` (Fri 17:00 BST), wrapper-prompt pattern. Approval design locked as **option-a draft-only**: drafts post to `#flex-agent-jago` with framing for `#product-cleo-card`; Jago manually edits + publishes onward. No approval-sweep dependency. Continuity loop: Friday writes `commitments_for_next_week` into `snapshots/weekly_friday_<date>.json`; Monday reads last Friday's artefact for carry-over context. Weekend signal handled via Daily Brief artefact re-use, not double-scanning — Daily Brief gains a `snapshots/daily_brief_<date>.json` write step (LIVE only) which Weekly Monday reads + filters for team-relevance.
- **2026-04-26** — **All four routines wired via `RemoteTrigger` API.** Sync re-cronned `7 7 * * 1-5` → `0 6 * * 1-5` (07:00 BST). Brief re-cronned `53 7 * * 1-5` → `30 6 * * 1-5` (07:30 BST). Weekly Monday created `trig_01CdWfPUFDnn2Eg64SruRszZ` (`30 7 * * 1`). Weekly Friday created `trig_015kvdNZvkNCL8CFgJP4299X` (`0 16 * * 5`). Both new routines were created with `allow_unrestricted_git_push: true` set at creation — directly tests the 403-fix working hypothesis without disturbing Sync/Brief. **API findings**: (1) `RemoteTrigger` schema exposes only `list/get/create/update/run` — no `delete` action, so plan-A "delete + recreate" had to fall back to "update + leave existing IDs". (2) `next_run_at` consistently displays a 1-7 min jitter from the cron minute (e.g. cron `30 7` shows `next_run 07:36`); appears to be scheduler quantisation, not stale state. Cron expression is canonical. (3) Cron-only updates don't trigger the `event_type` schema gotcha that wrapper edits do — they accepted clean. (4) UI inspection still recommended for cleanup of any orphan/disabled triggers if delete-and-recreate becomes necessary.
- **2026-04-26** — **Strategic review / audit captured.** Reviewed the four-routine workflow end-to-end. Identified gaps: (a) silent-failure risk — no heartbeat or error escalation; (b) Job 2 is biggest functional gap remaining; (c) Granola data sitting unused; (d) auto-write shifts burden to Notion triage queue (extraction quality matters more); (e) no pattern/trend detection — stateless fires can't surface "stuck" items; (f) notification fatigue path; (g) no OOO suppression. Captured 10 prioritised backlog items in Audit Backlog section above. First-week pitfalls flagged: 403 push test (binary), Calendar filter false-negatives, Notion `Status` enum case sensitivity, thin first Weekly Monday output by design. Decision: ship as-is for Mon 27 Apr; address backlog post-week-1 calibration data.
- **2026-04-26** — **Brief switched from Slack-capture flow to auto-write `Proposed` rows.** Slack candidates that pass strict criteria are now auto-written to the Action Items DB with `Status = Proposed`, blank Owner/Priority/Due, Source field tagging the Slack channel. Jago triages in Notion: assign owner → `Active`, or delete/`Rejected`. Dedup before write: fuzzy-match against `Status IN (Proposed, Active)` rows created in last 7 days. Brief output replaces "Candidate Slack action items + capture N" section with "Added to Action Items DB" linking each Notion page. Action Items section gains a triage-nudge line: `:warning: N Proposed items > 3 days old — needs triage` when queue accumulates. Weekly Monday updated to read `slack_candidates_written` (was `slack_candidates`). Weekly Friday's "late-capture" section repurposed to "Triage queue: Proposed items added this week, still untriaged" — same-week visibility on uncleared queue. Approval Sweep deferred item retired: Brief no longer needs it; Weekly drafts are manually edited+published by Jago; only Job 2 might revisit.
