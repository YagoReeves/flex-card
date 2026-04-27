# Granola Meeting → Action Items + Central Memory Extraction

Canonical prompt for the **manual-interactive** post-meeting extraction flow. Used when Jago says *"extract action items from this Granola meeting: <url>"*, *"process this transcript"*, or similar in any Claude Code session that has the Granola + Notion MCPs attached.

The flow extracts **two parallel candidate streams** from the same transcript:
1. **Action Items** — trackable tasks with an owner and a near-term outcome. Written to the Flex Action Items DB as `Status = Proposed`.
2. **Central Memory entries** — risks, issues, material decisions, standing commitments to partners, and forward-looking ideas. Written to the Central Memory Flex Card DB as `Status = Logged`.

A single source quote can produce **both** (e.g. *"We agreed 2-policy structure + Jago to draft by Friday"* → Decision in Central Memory + Action Item to draft). Don't drop one to capture the other — the two DBs serve different cognitive loads.

This is the same logic the cron-driven Granola Sweep applies in its auto-flow, and the same logic the future conversational Slack agent will reuse — keeping it canonical here means we don't duplicate it later.

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

## Step 3b — Extract Central Memory candidates

Distinct from action items: items that aren't trackable tasks but *are* worth remembering for the future. Apply the same strict bar — false positives hurt more than false negatives. A candidate qualifies if it fits one of these five categories:

- **Risk** — a concern raised about something that *might* go wrong, with impact on Flex Card outcomes. Cues: *"the concern is…"*, *"we're worried that…"*, *"if X happens we're stuck…"*, *"this could push UAT…"*. Don't conflate with Issue (Risk = forward-looking; Issue = currently happening).
- **Issue** — a problem that's actively impacting work right now. Cues: *"we're blocked by…"*, *"X isn't working…"*, *"<partner> hasn't responded in N weeks…"*. If a Cleo-side person can resolve it with a discrete action, **also** capture an Action Item; the Issue stays in Central Memory for the broader context.
- **Decision** — a material choice that's been made or agreed. Cues: *"we agreed…"*, *"we decided…"*, *"the call is…"*, *"locked in as…"*. Decisions are typically `Severity = High` if they affect product construct, partner posture, or timeline; `Medium` if they affect implementation detail. **Skip purely procedural decisions** ("let's reschedule to Tuesday") — those aren't memory-worthy.
- **Commitment** — a standing promise to a partner or stakeholder that doesn't have a near-term deliverable but *does* shape future behaviour. Cues: *"we promised WebBank we'll…"*, *"we committed to never…"*, *"we always give X notice…"*, *"agreed with Mastercard that…"*. **Distinct from a near-term action**: if there's a Friday deliverable, capture the Action Item AND optionally a parent Commitment if the standing rule is non-obvious.
- **Idea** — a future-relevant thought, exploration, or hypothesis raised that isn't actionable now but worth remembering. Cues: *"in future we should…"*, *"worth exploring…"*, *"if we ever…"*, *"one option down the line…"*. Ideas are deliberately low-frequency — most discussion noise is not a real Idea.

Reject:
- Hypothetical / aspirational with no clear category fit.
- Past-tense narration of things that already happened (those are facts, not memory items).
- Procedural meeting admin (scheduling, agenda-setting).
- Anything already covered as an Action Item where the CM entry would be a near-duplicate.

For each surviving CM candidate, build:
- **Entry** (short title, ~6 words, leads with the noun: *"WebBank credit-policy structure (2 vs 3)"*, not *"Decision about credit policies"*)
- **Summary** (one to two sentences, the actual content of the risk/issue/decision/commitment/idea)
- **Suggested Workstream** (from `project_context.md` §5)
- **Suggested Severity** — leave blank by default. Set `High` only if the item maps to a **critical-path risk** in `project_context.md` §4 (product-classification challenge, partner approval delays). Tag `[critical-path-risk: 1]` or `[critical-path-risk: 2]` in Notes when so set.
- **Suggested Partner** — set if the entry is partner-specific (Risk about WebBank's posture, Commitment to Marqeta, etc.). Leave blank if cross-cutting or internal.
- **Source quote** (≤120 chars)

## Step 4 — Propose to Jago

Output **two markdown tables** under labelled headers — one per stream. Number each independently (`A1, A2, …` for Action Items; `M1, M2, …` for Central Memory) so approval syntax is unambiguous.

### Action Items proposal

Columns: `#`, `Actionable`, `Owner side` (`Cleo` / partner name e.g. `WebBank`), `Suggested owner` (Cleo person if internally owned; `—` if partner-owned), `Due`, `Source quote`. Number `A1` upwards.

If no Action Items qualify: state `_No action items extracted._` and skip the table.

### Central Memory proposal

Columns: `#`, `Entry`, `Category` (`Risk` / `Issue` / `Decision` / `Commitment` / `Idea`), `Workstream`, `Suggested Severity` (`High` / `—`), `Suggested Partner` (partner name or `—`), `Source quote`. Number `M1` upwards.

If no CM entries qualify: state `_No Central Memory entries extracted._` and skip the table.

Below the table, list:
- **Items considered but excluded** — borderline candidates with a one-line reason for exclusion. Lets Jago override if he disagrees.
- **Uncertainties** — names you might have transcribed wrong, ambiguous owners, dates you've inferred.

End with the approval-syntax line:
> Reply with `approve all` / `approve A1, A3, M2, ...` / `approve all except A2, M1` / edits like *"approve all, change A3 owner to X, drop M5"* / `reject all`. Identifiers `Ax` = Action Item, `Mx` = Central Memory entry — approve from either or both lists in a single reply.

**Do not write anything to Notion yet.** Wait for Jago's reply.

## Step 5 — Apply approvals

When Jago replies, parse:
- Which numbered items to keep
- Owner edits (apply to Notes field per below)
- Title edits, due-date edits, drops

If anything is unclear, ask once before writing.

## Step 6a — Write Action Items to the Action Items DB

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

## Step 6b — Write Central Memory entries to the Central Memory DB

Data source: `collection://33e5c63b-8745-8199-ab41-000bbbcaaf18`. Use `mcp__claude_ai_Notion__notion-create-pages` with one page per approved entry. Schema (verified live 2026-04-27 evening after field additions):

- **Entry** (title): the approved short name (~6 words)
- **Summary** (text): **Mandatory.** One to two sentences capturing what this is at a glance — Jago should be able to scan the DB list view and understand each entry without opening the page. Don't leave blank.
- **Next Step** (text): **Mandatory.** Either (a) the immediate next action to progress this entry (e.g. *"Reach decision at Tuesday call"*, *"Monitor WebBank reply"*, *"Escalate to Jared"*), OR (b) for entries already at `Decided` / `Resolved` status, the final decision/resolution captured in shortform. If genuinely unknown at write time, populate with `Awaiting triage` rather than leaving blank.
- **Category** (select, exact value): one of `[Risk, Issue, Decision, Commitment, Idea]`.
- **Workstream** (select): same 10-option list as Action Items DB. Default to `Programme` if genuinely cross-cutting.
- **Status** (status type, exact value): `Logged`. Always. Jago triages forward in Notion.
- **Severity** (select): `High` only if the entry maps to a critical-path risk in `project_context.md` §4 (and the Notes string includes `[critical-path-risk: 1]` or `[critical-path-risk: 2]`); otherwise leave blank for Jago to set on triage.
- **Responsible** (person): leave blank — Jago tags manually. If a clear Cleo person was named in the source as accountable, put `Responsible: <name>.` in the body of the page.
- **Partner** (select): set to the partner name if the entry is partner-specific. Exact value from `[WebBank, Marqeta, Mastercard, Peach, Indebted, TabaPay, Pinwheel, Idemia, IC Payments, I2C]`. Leave blank if cross-cutting or internal.
- **Source** (select, exact value): `Meeting`.
- **Source Link** (url): the Granola URL Jago provided.
- **Source Context** (text): `<Meeting title>, <YYYY-MM-DD> — "<short verbatim quote>"` (quote ≤120 chars).
- **Due Date** (date `date:Due Date:start`): populate only if there's an explicit decision-needed-by, risk-resolution-deadline, or commitment-due date in the source. Leave blank otherwise.
- Page body: 1–2 short paragraphs expanding on the Summary if there's nuance worth retaining (context the title and Summary can't carry). Quote ≤120 chars rule still applies. Don't paste transcript chunks.

**Don't dedup against the existing DB on the manual flow** — Jago is reviewing each entry explicitly. (The cron Sweep does dedup; manual flow doesn't.)

## Step 7 — Confirm back

Return a short confirmation: counts and links per stream. Format:

> **Action Items DB** (N written): `[<title>](url)` · `[<title>](url)` · …
> **Central Memory DB** (M written): `[<entry>](url)` · `[<entry>](url)` · …

If a stream is empty, omit it. No preamble, no meta-commentary.

## Constraints

- **Confidentiality**: never quote sensitive Box/WebBank content verbatim into Notion. The Notes/Source Context fields stay above board for general circulation in Cleo.
- **Quote brevity**: Source Context quotes capped ~120 chars. The point is traceability, not a transcript dump.
- **Tone**: imperative actionables, not narrated descriptions. "Do X" not "We should consider doing X".
- **Don't dedup against the existing DB on the manual flow** — Jago is reviewing each item explicitly, dedup adds noise to the proposal step. (This differs from the Daily Brief auto-write, which does dedup.)
