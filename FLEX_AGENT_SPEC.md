# Flex Agent — Spec (v1)

**Last updated**: 2026-04-25 · **Owner**: Jago Reeves · **Status**: Daily Brief + WebBank Sync routines both live and configured for Mon 27 Apr first fires

## Pick up here next session

**What's live:**
- Daily Brief routine `trig_0168M4zWyESpqBw57f9PDXfp` — fires `53 7 * * 1-5` (08:53 BST). Wrapper-pattern prompt fetches `prompts/daily_brief.md` from working-dir clone of `YagoReeves/flex-card`.
- WebBank Sync routine `trig_01V1a8NQ1SayN7kpnq1Q7Eh2` — fires `7 7 * * 1-5` (08:07 BST). Same wrapper pattern, fetches `prompts/webbank_sync.md`. Dry-run verified successful 2026-04-25.
- WebBank Mirror DB populated (80 rows, initial state).
- DST reminder routine `trig_01EreY3zZT6e7sVapd6g9LVU` — fires Mon 19 Oct 2026. Updated to remind on bumping both routine crons (Brief + Sync) by +1 UTC hour.
- GitHub repo `https://github.com/YagoReeves/flex-card` — private, attached to both production routines as a working-directory source. Holds spec, prompts, snapshot history.

**First fire to monitor: Mon 27 Apr 2026.**
- 08:07 BST: Sync runs first. Should parse Excel, diff vs `snapshots/webbank_checklist_2026-04-24.json`, update Mirror DB, write today's snapshot + diff JSONs, `git push origin main`, post digest to `#flex-agent-jago`.
- 08:53 BST: Brief fires. Should consume today's diff JSON from working-dir clone, summarise WebBank slice, post to `#flex-agent-jago`.

**Open for Jago (manual):**
- (In Notion UI) group the two Agent-owned DBs into a toggle section under Flex Hub's Central Memory — Notion API was blocking the auto-insert.
- Spot-check a handful of Mirror DB rows against the Excel — we parsed 80 kept + 62 dropped, but WebBank's own Grand Total is 100; worth a sanity pass.
- Watch Monday's first live fires; report any anomalies.

**Outstanding jobs to build:**
- Job 2: Meeting Action Item Extraction (manual trigger).
- Job 4: Weekly Cycle (Mon 09:00 + Fri 17:00).
- Approval Sweep — **blocked** by remote-routine 1-hour minimum; needs design revisit. Lower priority now that Sync auto-writes (no approval gate needed for it per locked spec).

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
- [ ] Build **Approval Sweep** — 15-min interval per spec, but remote routines have a 1-hour minimum. Design revisit needed: hourly sweep, local CronCreate (session-bound), or webhook trigger. Deferred — lower priority now that Sync auto-writes (no approval gate per locked option-c design).

### Routine management

- **Wrapper-prompt pattern** (adopted 2026-04-25): both production routines carry a tiny ~6-line wrapper that fetches the canonical prompt from `prompts/<file>.md` in the attached repo's working directory. To change a prompt, edit the file in the repo, commit, push to `main`. Next routine fire automatically picks up the new version — no `RemoteTrigger` update needed.
- **Caveats on `RemoteTrigger` PUTs**: when calling `update`, send the **full** `session_context` object (including `sources` with `allow_unrestricted_git_push: true`). Partial updates replace the object whole and clobber attached repo state. Always `RemoteTrigger get` first if you're unsure of the current shape.
- All routines visible + editable at https://claude.ai/code/routines

### Phase 2 — Core jobs
- [x] **WebBank Mirror DB initial population** — 80 rows seeded from Excel (2026-04-24 parse); 62 rows filtered as `Not Required`/`Existing - Not Req.`. Parser in Python, kept at `staging/webbank_parsed.json`. Schema extended with `DD`, `DP`, `IP` category options after first-pass revealed missing codes.
- [x] **WebBank Checklist Daily Sync** routine — live. Cron `7 7 * * 1-5` (08:07 BST). Routine ID `trig_01V1a8NQ1SayN7kpnq1Q7Eh2`. Model: `claude-sonnet-4-6`. MCP connections: Slack, Notion, Box. Repo source: `YagoReeves/flex-card` with `allow_unrestricted_git_push: true`. Design: option-c (auto-write DB + digest to `#flex-agent-jago` only, no `#product-cleo-card`, no approval gate). On fire: Read prior snapshot from working dir, parse Excel via Box MCP, diff, update Mirror DB, write today's snapshot+diff JSONs, `git push origin main`, post digest. Dry-run completed 2026-04-25.
- [ ] Build **Post-Meeting Action Item Extractor** (manual trigger)
- [ ] Build **Weekly Cycle** (Mon 09:00, Fri 17:00)

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
