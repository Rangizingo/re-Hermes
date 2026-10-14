# Hermes on ai-core: cost-claim trace, 2026-10-14

Counts only. No message content was read or printed. The box was touched read-only (`ls`, `stat`, `grep -c`, `grep -n` for flag lines, `systemctl --user is-active`, `date`, and `scp` out to scratch). No writes on ai-core, no hermes run, no model call, no service change.

## Summary

- Snapshot: local copy of `state.db` with WAL and SHM, taken 2026-10-14 11:38:59 UTC. Last recorded activity 2026-10-13 21:55:48 UTC, so the 48 h window holds about 34 h of recorded traffic (measured).
- A, background review: last completion 2026-10-12 11:52:52 UTC. Zero completions in 77 Telegram turns after the 16:15 UTC restart (measured). The plan's literal 48 h count is 1, and that row predates the switch.
- B, model calls per user turn: Telegram, 7 d, n=273 turns: median 2, p90 17, max 243 (measured).
- C, tool-search bridge: used in 28 of 273 Telegram turns, 10.3% (measured, 7 d). Those turns hold 44.4% of model rows.
- D, stall guard: its log text appeared 0 times in 7 d and in every retained log. Nudge rows in the DB: 0 (measured).
- E, transcript size: 60 non-empty gateway sessions, median 23 rows, max 766 rows (measured). Rows, not tokens.

## Windows and sources

- Times are UTC. The box clock and the Mac clock agree (epoch 1791977939 at copy start). 48 h cutoff 2026-10-12 11:38:59. 7 d cutoff 2026-10-07 11:38:59. Restart cutoff 2026-10-12 16:15:00, the last of four s1 restarts (15:32, 15:54, 16:09, 16:15) in `/Users/peterb/Documents/AI/hermes/plan.md`, s1 summary.
- DB copy: `state.db` 48,033,792 bytes and WAL 4,243,632 bytes, the same sizes as the box listing taken just before the copy. `PRAGMA integrity_check` returned ok (measured, `sqlite3 -readonly`).
- Log copies: agent.log, agent.log.1 to .3, errors.log, errors.log.1 and .2, gateway.log. Sizes match the box except agent.log (Limits 7). Coverage, first and last timestamp (measured): agent logs 2026-09-29 01:06 to 2026-10-14 11:38 UTC; errors and gateway logs 2026-09-25 to 2026-10-14 08:37 UTC.
- Scratch: `/private/tmp/claude-501/-Users-peterb-Documents-AI/6f532f03-13e1-40d9-a339-6a0aca3440a4/scratchpad/trace/`. Nothing written to the repo.
- Gateway unit `hermes-gateway`: `systemctl --user is-active` returned active (measured).

## A. Background review runs

Plan item A1 (`/Users/peterb/Documents/AI/hermes/plan.md:23`): proof "0 new task=background_review rows in session_model_usage over 48 h."

Method:
- DB: `session_model_usage` grouped by `task`. New = `first_seen >= cutoff`. Touched = `last_seen >= cutoff`.
- Logs: byte-level Python count of `Background review complete`, the completion line at pin `agent/background_review.py:807`. The box copy of that file contains the text (grep -c = 1).

Raw counts (measured):

| Window | New DB rows | Touched DB rows | Completion lines |
|---|---|---|---|
| 48 h, from 2026-10-12 11:38:59 | 1 | 1 | 1 |
| 7 d, from 2026-10-07 11:38:59 | 12 | 12 | 30 |
| Since restart, from 2026-10-12 16:15:00 | 0 | 0 | 0 |
| All retained | 12 rows, cumulative `api_call_count` 87 | not counted | 83 (first 2026-09-29 16:37:58) |

- Sample since restart: 77 Telegram turns and 1 oneshot turn (measured, `user` rows). Sessions with 10 or more user turns since restart: 3, all sources. Same count for the 7 d before restart: 6.
- The 48 h row has first_seen = last_seen = 2026-10-12 11:52:52 UTC and one call. It matches the last completion line.
- Other `background review` lines in 7 d: 17, all `Interrupt requested (hard)` notices from `run_agent`. Not review runs (measured, Python, template words only).
- Box config: the `background_review` block has `enabled: false` (`config.yaml` line 2466, grep -A3 after the key; flag only).

What it means:
- The literal proof reads 1, not 0. The one row predates the switch. The 48 h window starts before the s1 restarts, so the window as written can't pass. Use "since the switch" instead.
- The switch time is not recorded. `config.yaml` mtime is 2026-10-13 16:05:19 UTC, after s1, so it doesn't date the switch. The last completion was at 11:52:52 UTC on 10-12, so the switch took effect after that (inferred).
- Before the restart: 30 completions over 292 turns in 7 d, all sources, so 0.10 per turn (measured ratio). `docs/core-loop.md` section 6 says the trigger is every 10 user turns (it cites `agent/turn_context.py:752-763`; not re-read here). If still on, the 3 sessions that reached 10 turns after the restart would each have fired at least once (est.). Zero were logged.
- `session_model_usage` keeps one row per session, model, provider, base URL, mode, and task, with a cumulative count. A new review in a session that already has a row updates `api_call_count` and `last_seen` and adds no row. "0 new rows" can hide calls. A stronger proof is zero completion lines and zero rows with `last_seen` after the switch time.

## B. Model calls per user turn

Method:
- Model-call row = one `assistant` row in `messages` (one per model response). `session_model_usage` can't give per-turn counts because it is aggregated.
- Turn = a `user` row up to the next `user` row in the same session. SQL: `LEAD(id) OVER (PARTITION BY session_id ORDER BY id)` on `role = 'user'`, then count `assistant` ids between the two.
- Cross-check: Python count of log lines containing `API call #`. The per-call log line was not found in the pinned source, so this checks only the count.

Raw counts (measured, 7 d):
- Telegram: n=273 turns in 56 sessions. Median 2, p75 6, p90 17, max 243. Mean 8.4 (2292 rows / 273). Turns with zero rows: 5.
- All sources: n=292 turns in 71 sessions. Median 2, max 243. Rows 3517, equal to all `assistant` rows in 7 d, so the split is complete.
- Telegram turns without a bridge tool (see C): n=245, median 1, max 82. With a bridge tool: n=28, median 9.5, max 243.
- Cross-check: 3519 `API call #` lines in 7 d (all sources) against 3517 assistant rows. Off by 2, not traced.

What it means:
- A typical Telegram turn makes 2 model calls. That is the floor in `core-loop.md` section 6, so the median sits on the floor.
- The tail carries the calls: 1 in 10 turns makes 17 or more, and the top turn makes 243. Turns with a bridge tool are 10% of Telegram turns and 44% of their model rows (see C).
- Rows are calls, not tokens or seconds.

## C. Tool search (deferral bridge)

Method:
- Bridge names: `tool_search`, `tool_describe`, `tool_call`, read from `function.name` in each `tool_calls` entry of `messages`.
- Log text: `tool_search activated (tier %d): ...`, pin `tools/tool_search.py:379`, logger `tools.tool_search`, INFO. The box copy contains it (grep -c = 1). It logs only on the path with deferred tools. The no-deferral branch (`tools/tool_search.py:360-362`) returns without logging.
- Counts: Python byte-level count of the text.

Raw counts (measured):
- `tool_search activated` lines: 105 in all retained logs (2026-09-29 11:00:15 to 2026-10-13 16:06:44 UTC), 38 in 7 d, 21 in 48 h. All 105 say tier 1.
- Telegram, 7 d: 15 of 56 sessions used the bridge. 28 of 273 turns (10.3%). Model rows in those turns: 1017 of 2292 (44.4%). Median rows per bridge turn: 9.5 (n=28).
- All sources, 7 d: 17 of 71 sessions, 30 of 292 turns, 1223 of 3517 rows (34.8%).
- By tool, all sources, 7 d: `tool_call` 17 sessions, 164 rows. `tool_describe` 14 sessions, 20 rows. `tool_search` 4 sessions, 5 rows.
- Telegram bridge turns: 10 of 96 turns in 48 h. 6 of 77 since restart.
- Not part of C, for context: the MCP names pinned in s1 were called directly, not through the bridge, 67 times all time (`mcp__gmail__gmail_search` 21, `gmail_read` 8, `mail_query` 16, `mcp__crawl4ai__scrape` 22).

What it means:
- The bridge shows up in about 1 Telegram turn in 10. Those turns are long: median 9.5 model rows, against 1 for the rest. They carry 44% of Telegram model rows.
- The doc's "3 to 4 calls per MCP lookup" can't be checked here. A bridge turn bundles several tools, so per-lookup counts aren't isolated.
- The listing cost (the 4,000-token cap in `core-loop.md` section 4, a config value not re-read) is not measured.

## D. Stall guard

Method:
- Pinned text, `agent/turn_final_response.py:220`, built from two string literals. Grep pattern: `Stall guard: turn ending on trailing continue-intent`. Same block: `Degenerate final:` at `:226` (warning, same counter and scope).
- Box: `agent/turn_final_response.py` contains both patterns (grep -c = 1 each). `stall_guards` is not set in `config.yaml`. The installed default is True (`hermes_cli/config_defaults.py:161`, box copy). The guard is on.
- Logs: byte-level Python counts.
- DB: user rows containing the continuation nudge (pin `agent/conversation_loop.py:958-961`, pattern `[System: Continue now. Execute the required tool calls`), the degenerate nudge (`:965-969`, pattern `[System: Your previous message ended the turn with a fragment`), and any user row starting `[System:`. Counts only.

Raw counts (measured):
- `Stall guard: turn ending on trailing continue-intent`: 0 in 7 d, 0 in 48 h, 0 in all retained logs.
- `Degenerate final:`: 0 in all retained logs.
- DB, all time: continuation nudge rows 0. Degenerate nudge rows 0. User rows starting `[System:` 0.

What it means:
- The guard did not fire in 7 days. Its extra-call cost in this window is 0. The claim ("+1 call per stall, max 2") has no event behind it here, so it can't be measured.
- The DB check assumes nudge messages are written to `messages`. That is not verified in code. The ack continuation uses the same nudge text as the stall guard but logs nothing, so the DB check is its only probe.

## E. Transcript size per gateway session

Method:
- Sessions with `source` in (`telegram`, `api_server`). Per session, count all `messages` rows, `active = 1` rows, `compacted = 1` rows, and `session_meta` rows.

Raw counts (measured, all time unless marked):
- Gateway sessions: 64 (60 telegram, 4 api_server). Sessions with zero rows: 4. Median total rows 20.0 (n=64, zeros included). Max 767.
- Non-empty sessions, excluding `session_meta` rows (n=60): median 23.0, max 766. Max session: `20261009_174406_24248469` (767 total rows, 1 `session_meta`).
- Gateway sessions with a row in 7 d (n=56): median 26.5 total rows, max 767.
- Active rows equal total rows. Compacted rows in gateway sessions: 0.
- All rows in the DB: 7462 = 3598 tool, 3524 assistant, 298 user, 42 session_meta.

What it means:
- Each gateway turn reloads the active transcript. The pinned code has no row cap (`gateway/session_transcript.py:631`, `hermes_state_messages.py:1515-1518`, as cited in `docs/surfaces.md`; not re-read here). The box build was not traced for this.
- A median session is small (23 rows). The largest sends 766 rows into every turn it runs.
- Rows are not tokens. Tool rows are 48% of all rows, and their length was not measured.

## Limits

1. The switch time. The A result shows no completions after 2026-10-12 11:52:52 UTC. It does not date the switch.
2. Post-switch sample. About 30 h of traffic (77 Telegram turns, last recorded activity 2026-10-13 21:55:48 UTC). Claim A's zero rests on that sample.
3. The gap from 21:55:48 UTC on 10-13 to the copy (about 14 h) has no recorded turns. Whether it was idle wasn't verified. No inbound-message log was read. The last gateway log line is 2026-10-14 08:37:08 UTC.
4. `session_model_usage` is aggregate. It has no per-call timestamps. New-row counts miss calls in existing sessions. Window counts use `first_seen` and `last_seen` only.
5. Pin vs box. The pin is 97a9ae8ee5. The box runs v0.21.5 (59004a62 plus local edits). Line numbers are the pin's. Matched strings were confirmed on the box (grep -c). Logic was not diffed.
6. Logs. A shell `grep -c` on `agent.log.1` returned no output (cause not found). Every log count here is from a Python byte-level read of the copy. Rotation keeps three backups, so agent logs before 2026-09-29 01:06 UTC are gone.
7. `agent.log` is live. The box file grew 2,418 bytes after the copy (152,120 to 154,538). The new lines were written after the snapshot time, so the window counts don't change.
8. Turn definition. A user row up to the next user row. System-written user rows would split turns. None exist in the DB (see D), so the counts don't move. Five Telegram turns have no assistant row. They stay in the medians as zeros.
9. Model-call rows are assistant rows: not tokens, latency or cost. The `API call #` count (3519 against 3517) supports the row count within 2. The cause of the 2 was not traced.
10. No CLI sessions in `state.db`. Sources are telegram, subagent, api_server and oneshot. "Typical turn" here means Telegram.
11. `tool_search activated` counts only the deferred-tools path. How often the no-deferral path ran is not measured.
12. Claim D. Zero events shows nothing about what the guard would do. The ack path has no log line. Its only trace is nudge rows, and their persistence is not verified.
13. Claim E. Rows, not bytes. "Reloads the whole transcript" comes from pinned code, not from the box build.
14. Not measured: latency, tokens per request, the review cost in the code comment (`agent/turn_finalizer.py:779`, cited in `docs/core-loop.md`, not re-read), the bridge listing size, and whether the 243-row turn was a loop or real work.

What the data cannot settle:
- A: when the switch happened, whether a review would have run in each session (the trigger is doc-cited, not run), and calls per review (only cumulative counts exist).
- B: time or tokens per turn.
- C: tokens added by the bridge listing, and calls for one deferred lookup.
- D: whether the stall guard helps or hurts. Zero firings only shows it did not fire.
- E: bytes per turn.

## Box state

Read-only. No writes, no restarts, no hermes or model runs. Gateway unit reported active.
