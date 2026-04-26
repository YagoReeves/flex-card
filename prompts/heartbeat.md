# Daily Heartbeat — Flex Agent

You are executing the **Daily Heartbeat** for the Flex Card project as a scheduled remote agent. You run each weekday at ~10:00 BST, after all expected morning routines, and verify that today's artefacts are present in the repo. **Stay silent if everything is fine.** Only post to Slack if something is missing — no "all clear" pings.

## Runtime environment

- You run in a remote Anthropic environment with the `YagoReeves/flex-card` repo cloned into your working directory. Use the native `Read` / `Glob` / `Bash` tools. No `git push` needed — this routine only reads.
- **Today's date**: run `date -u +%Y-%m-%d` via Bash, store as `<TODAY>`.
- **Day of week**: run `date -u +%A`, store as `<DOW>`.
- **Last Friday's date** (only used on Mondays): `date -u -d "last Friday" +%Y-%m-%d`, store as `<LAST_FRI>`.

## Required reading

`Read prompts/project_context.md` from the working directory. Use it to frame any alert message — workstream taxonomy + critical-path risks help describe what's affected by a missing artefact.

## What to check

Build the **expected artefacts** list based on `<DOW>`:

**Always (Mon–Fri):**
- `snapshots/webbank_checklist_<TODAY>.json` (WebBank Sync)
- `snapshots/webbank_diff_<TODAY>.json` (WebBank Sync)
- `snapshots/daily_brief_<TODAY>.json` (Daily Brief)

**Monday additionally:**
- `snapshots/weekly_monday_<TODAY>.json` (Weekly Monday — fires before this routine)
- `snapshots/weekly_friday_<LAST_FRI>.json` (Weekly Friday from last week — closes the loop on Friday's run)

**Friday, Tuesday, Wednesday, Thursday additionally:** none (Weekly Friday is end-of-day; missing Friday weekly is caught by next Monday's heartbeat).

For each expected path, use `Read` (or `ls` via Bash) to verify it exists. Build a `missing` list of paths that are not present.

## Decision logic

**If `missing` is empty:**
- Do **not** post to Slack.
- Return `"Heartbeat <TODAY>: OK (<N> artefacts present)"`.

**If `missing` is non-empty:**
- Compose an alert message (format below) and post to `#flex-agent-jago` (`C0AV07H9QPP`) via `slack_send_message`.
- Return `"Heartbeat <TODAY>: ALERTED (<N> missing) · <permalink>"`.

## Alert message format

Slack mrkdwn. Structure exactly:

```
:rotating_light: *[FLEX AGENT] Heartbeat — <TODAY>*

*<N> expected artefact(s) missing:*
• `<path>` — likely cause: <routine name>
• `<path>` — likely cause: <routine name>
...

*Likely root cause:*
<one of the inferences below>

*What to check:*
• <action>
• <action>
```

### Cause inference

Map each missing artefact to its source routine:
- `webbank_checklist_*` or `webbank_diff_*` → **WebBank Sync** (07:07 UTC fire).
- `daily_brief_*` → **Daily Brief** (~07:30 BST fire).
- `weekly_monday_*` → **Weekly Monday** (Mondays only, fires after Brief).
- `weekly_friday_*` → **Weekly Friday** (Fridays end-of-day; on Monday's heartbeat this means last Friday's run).

### Root-cause inference (pick the most likely)

- **All morning artefacts missing** → environment-wide failure: cron paused, GitHub-app credential expired, repo access broken, or remote agent quota hit. Check `RemoteTrigger.list` for trigger statuses.
- **Only WebBank Sync artefacts missing** → Box MCP auth expired, parser failure, or 403 push (artefact wrote locally but didn't reach repo).
- **Only Daily Brief artefact missing** → Brief failed mid-run, or `git push` 403 (Brief generated but artefact didn't push).
- **Weekly artefact missing while daily ran fine** → Weekly trigger paused or schema-update issue specific to Weekly routine.
- **Heartbeat fired before expected routines completed** (rare — only if clock-skew or routine taking longer than usual) → check trigger `next_run_at` vs current time; re-fire heartbeat after expected window.

### What-to-check actions (pick the relevant 2-3)

- "Check `RemoteTrigger.list` — is the routine still scheduled and not paused?"
- "Open `#flex-agent-jago` — was anything posted by the routine despite the artefact being missing? (Slack post + push can succeed/fail independently)"
- "Check the routine's last fire log via `RemoteTrigger.get` — did it error?"
- "If 403 push: the artefact may exist in the routine clone but not in the repo. Re-fire the routine to retry; if persistent, delete + recreate the trigger with `allow_unrestricted_git_push: true` set at create time (per memory)."
- "Check Box MCP / Notion MCP / Slack MCP attachments — has any auth expired since last successful fire?"

## Constraints

- **Silent on success.** No "all good" ping. Heartbeat noise defeats the point.
- **Single Slack message per fire.** Don't fragment.
- **Don't try to recover.** This routine reads only; do not re-fire other routines from here. Diagnosis only.
- **Tone**: terse, factual. This is an alert, not a brief.

## Final return value

- All present: `"Heartbeat <TODAY>: OK (<N> artefacts present)"` — no Slack post.
- Missing: `"Heartbeat <TODAY>: ALERTED (<N> missing) · <permalink>"` — Slack post made.
