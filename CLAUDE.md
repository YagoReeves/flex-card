# Flex Card — Project Context

Working directory for **Flex Card launch PMO + Flex Agent automation**. Pillar-level context lives one level up at `~/Documents/cleo/credit_pillar/CLAUDE.md`. Don't duplicate it.

## What lives here

- **Repo**: `YagoReeves/flex-card` (private). Local clone is this directory.
- **Scheduled routines** (cloud-hosted via `/schedule`):
  - WebBank Sync (07:00 BST, Mon–Fri)
  - Granola Sweep (06:45 BST, Mon–Fri)
  - Daily Brief (07:30 BST, Mon–Fri)
  - Weekly Monday (08:30 BST, Mon)
  - Weekly Friday (17:00 BST, Fri)
  - Heartbeat (10:00 BST, Mon–Fri)
- **Agent-managed Notion DBs** (under "Agent-managed Databases" toggle on Flex Hub):
  - Flex Action Items — `collection://52101c73-4538-4710-8327-797e8445dcc5`
  - WebBank Mirror — `collection://a31ff0cd-3aaa-434a-815d-0c8dcc8ba53f`
  - Flex — Weekly Progress Log — `collection://2d98ca42-9f65-4442-8ed2-0f97b0563429`
  - Central Memory Flex Card — `collection://33e5c63b-8745-8199-ab41-000bbbcaaf18` (under its own "Central Memory (Agent-managed)" sub-page)
  - Flex Card Implementation Checklist (Banking Ops) — manual

## Canonical references

- **`prompts/project_context.md`** — single-source-of-truth product reference (workstream spine, partner stack, critical-path risks, milestones). Read by every routine on every fire. Edit here when product context changes.
- **`FLEX_AGENT_SPEC.md`** — full spec of the automation: routine list, schemas, pickup-here-next-session pointer, audit backlog. Update at the end of each session that materially changes the system.

## Conventions

- **Push-to-main pattern** for all routine prompts: `git fetch origin main && git checkout -B main origin/main && git add ... && git commit ... && git push origin main`. The cloud sandbox starts on a `claude/<session>` branch; without the explicit checkout, `git push origin main` is a no-op.
- **Live Notion DB is source of truth for action items.** Snapshots are pointer indexes via `notion_page_id`. Routines re-resolve by ID before rendering, never by title-match (titles change in Notion; IDs are stable).
- **Action Items vs Central Memory routing.** Trackable tasks with an owner → Action Items DB. Risks / Issues / Decisions / Standing partner commitments / Forward-looking ideas → Central Memory DB (`Status = Logged`, Jago triages forward). One source can produce both; don't drop either.
- **WebBank Mirror DB write contract** — see the authoritative mapping table in `prompts/webbank_sync.md` step 4. Box-updated properties (overwritten on each fire): `Item Code` (create-only), `Document Name`, `Bank Guidance`, `WebBank Status`, `Bank Due`, `Bank Draft`, `Party`, `Category`. **Internal-only properties the routine never touches**: `Internal Notes`, `Internal Priority`, `Internal Status`, `Cleo Team`, `Internal Owner`, `Internal Due`, `Internal Draft`. Anything outside the Box-updated set is invisible to the routine and persists across runs. Don't reuse Box-updated property names for manual purposes — collisions overwrite.
- **WebBank Mirror archive lifecycle** — rows that WebBank flags `Not Required` / `Existing - Not Req.`, or that disappear from the checklist entirely, are **archived** (Notion `archived: true`), not status-flagged. The upsert searches both active and archived pages by code, so if WebBank re-flags a row as required the original page is un-archived and refreshed (no duplicate). Don't manually delete rows from the Mirror DB — archive instead, or the next sync will recreate them as new pages and lose the audit trail.

## Memory

Auto-memory directory has been chained through symlinks across two folder moves. The real store lives at `~/.claude/projects/-Users-jago-r-Documents-flex-card`. Symlinks point at it from `…-credit-pillar-flex-card` (after the flex_card → credit_pillar move) and `…-cleo-credit-pillar-flex-card` (after the credit_pillar → cleo move). New sessions opened in this folder resolve to the same memory store. Hard-rename later if cleanup is wanted.
