# Granola Meeting → Action Items Extraction

Canonical prompt for the **manual-interactive** post-meeting extraction flow. Used when Jago says "extract action items from this Granola meeting: <url>" or similar in any Claude Code session that has the Granola + Notion MCPs attached.

This is the same logic the future conversational Slack agent will run — keeping it canonical here means we don't duplicate it later.

## Trigger

Jago references a Granola meeting by URL (e.g. `https://notes.granola.ai/t/<uuid>-<slug>`) or by ID directly. Extract the UUID before the trailing slug — that's the `meeting_id` for the Granola MCP.

## Step 1 — Pull the transcript

Call `mcp__claude_ai_Granola__get_meeting_transcript` with `meeting_id`. Capture the meeting `title` and full `transcript`.

If the fetch fails, stop and tell Jago — don't fabricate.

## Step 2 — Frame the meeting

In one line, summarise what the meeting was (title + ~6 words on topic). This is shown back to Jago so he can spot if you've pulled the wrong meeting.

## Step 3 — Extract candidate action items

Apply **strict** criteria — false positives are worse than false negatives. A candidate qualifies if it's one of:
- **Explicit first-person commitment**: "I'll do X by Y", "I'll own X", "I'll send this tomorrow", "I'll add that to the agenda".
- **Direct ask of a named person**: "@alice please send X", "Bob, can you update the doc by Friday".
- **Agreed next-step that names an owner**: "we agreed Charlie will write up X".
- **Future commitment to a deliverable** with a discoverable owner — even if owner is implicit ("I'll write up the doc and share Friday" → owner = speaker).
- **Partner deliverable commitment**: an external partner agrees, is asked, or commits to deliver something to Cleo — e.g. "WebBank will send the Reg E template Friday", "Marqeta to provide sandbox access by 14 May", "Idemia to confirm plastics design feedback this week". Owner here is the **partner**, not a Cleo person — these get captured as **partner-owned** in Step 6.

Reject:
- Hypothetical / aspirational ("we should probably look at X").
- Discussion of past actions that already happened.
- "We need to think about X" without a named owner or deliverable.
- In-progress work being narrated without a new commitment.

For each surviving candidate, rephrase into a **short imperative actionable** — concrete, not a quote.

**Partner-owned vs internally-owned framing** (matters for Step 6):
- *"WebBank to send Reg E template"* — partner-owned (subject of the imperative is the partner; deliverable is theirs).
- *"Get Reg E template from WebBank"* — internally-owned (subject is Cleo; action is a chase or follow-up).
- The verb subject determines which side owns it. When partner-owned, **lead the imperative with the partner name** so the framing is unambiguous downstream.

## Step 4 — Propose to Jago

Output a markdown table with columns: `#`, `Actionable`, `Owner side` (`Cleo` / partner name e.g. `WebBank`), `Suggested owner` (Cleo person if internally owned; `—` if partner-owned), `Due`, `Source quote`. Number 1–N.

Below the table, list:
- **Items considered but excluded** — borderline candidates with a one-line reason for exclusion. Lets Jago override if he disagrees.
- **Uncertainties** — names you might have transcribed wrong, ambiguous owners, dates you've inferred.

End with the approval-syntax line:
> Reply with `approve all` / `approve N, M, ...` / `approve all except N, M` / edits like *"approve all, change #3 owner to X, drop #5"* / `reject all`.

**Do not write anything to Notion yet.** Wait for Jago's reply.

## Step 5 — Apply approvals

When Jago replies, parse:
- Which numbered items to keep
- Owner edits (apply to Notes field per below)
- Title edits, due-date edits, drops

If anything is unclear, ask once before writing.

## Step 6 — Write to Action Items DB

Data source: `collection://52101c73-4538-4710-8327-797e8445dcc5`. Use `mcp__claude_ai_Notion__notion-create-pages` with one page per approved item. Schema (verified live):

- **Title** (text): the approved imperative actionable
- **Status** (select): `Proposed`
- **Source** (select, exact value): `Meeting`
- **Workstream** (select): infer from content — one of `[Programme, Card Product, BNPL Product, Bank & Compliance, Platform Foundations, Money Movement, Servicing & Operations, Card Issuance & Fulfilment, Credit Reporting, Fraud & Risk]`. Default to `Programme` when ambiguous.
- **Source Link** (url): the Granola URL Jago provided
- **Source Context** (text): `<Meeting title>, <YYYY-MM-DD> — "<short verbatim quote>"` (quote under ~120 chars)
- **Notes** (text): `Owner: <name>.` for internally owned, or `Partner contact: <name @ partner>.` for partner-owned (where applicable), followed by any context worth carrying (one short sentence).
- **Owner** (person): leave blank — Jago tags the Notion person manually for internally-owned items.
- **Partner Owner** (single-select): set to the partner name when the item is **partner-owned** per the framing rule in Step 3 — exact value from `[WebBank, Marqeta, Mastercard, Peach, Indebted, TabaPay, Pinwheel, Idemia, IC Payments, I2C]`. Leave blank for internally-owned items. **Mutually exclusive with `Owner` in the auto-flow** — set exactly one (or neither, if truly untriaged).
- **Priority** (select): leave blank
- **Due** (date `date:Due:start`): populate only if approved with a due date
- **Created**: auto-set by Notion

**Ownership convention**: every action item should reflect either (a) `Owner` set + `Partner Owner` null = internally owned, or (b) `Owner` null + `Partner Owner` set = partner-owned (Cleo waiting on partner). Both blank is allowed only for genuinely-untriaged items.

## Step 7 — Confirm back

Return a short confirmation: count of items written, with markdown links to each new Notion page. No preamble, no meta-commentary.

## Constraints

- **Confidentiality**: never quote sensitive Box/WebBank content verbatim into Notion. The Notes/Source Context fields stay above board for general circulation in Cleo.
- **Quote brevity**: Source Context quotes capped ~120 chars. The point is traceability, not a transcript dump.
- **Tone**: imperative actionables, not narrated descriptions. "Do X" not "We should consider doing X".
- **Don't dedup against the existing DB on the manual flow** — Jago is reviewing each item explicitly, dedup adds noise to the proposal step. (This differs from the Daily Brief auto-write, which does dedup.)
