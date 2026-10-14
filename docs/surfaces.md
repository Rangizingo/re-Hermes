# Hermes surfaces: entry points, cron, and state

Read only. Nothing was run: no hermes, no model, no network, no ssh. Pinned to hermes-agent `97a9ae8ee5` (2026-10-11). Paths are relative to that repo root. The source working tree is dirty, so every citation comes from `git show` or `git grep` at the pin.

Labels: **counted** = grep or count of source. **est.** = arithmetic. **not run** = not executed. Code defaults are read from source. Anything about timing or cost is **not run**.

## 1. Short version

- Gateway platform messages enter through an adapter's `handle_message` (`gateway/platforms/base.py:4038`) and end in `agent.run_conversation` (`gateway/run_turn_runner.py:1724`). The loop itself is in [core-loop.md](core-loop.md).
- Cron jobs live in `~/.hermes/cron/jobs.json` (`cron/jobs.py:85-89`). An in-process ticker runs every 60 s by default (`cron/scheduler_provider.py:439`, **not run**). Output goes to the live adapter or to a SQLite queue (`cron/scheduler_delivery.py:1973`, `cron/delivery_queue.py:77-82`).
- Session and message state is one SQLite file, `~/.hermes/state.db` (`hermes_state.py:178`), behind `SessionDB` (`hermes_state.py:461`) and 34 `hermes_state*.py` modules (**counted**).
- Each gateway turn reads the active transcript from state.db before the agent cache is checked (`gateway/run_turn.py:2096`).

## 2. Entry points

CLI and direct runs:
- `cli.py:894` defines `HermesCLI`, which mixes in the chat turn. That turn calls `self.agent.run_conversation` at `hermes_cli/cli_chat_turn_mixin.py:369`.
- `cli.py:1651` `main()` runs under `fire` when the file is run directly (`cli.py:1779-1782`).
- The `hermes` script imports `hermes_cli.main.main` (`hermes:9-11`). How that reaches `HermesCLI` was not traced.
- API server: `gateway/platforms/api_server.py:4186` `_run_agent` defines `_run` (`:4221`), which calls `run_conversation` at `:4290`. It does not construct `TurnRunner`.

Inbound path for a gateway message:
1. An adapter calls `self.handle_message(event)`. Example: `_dispatch_with_text` at `plugins/platforms/telegram/adapter.py:6686`.
2. `BasePlatformAdapter.handle_message` (`gateway/platforms/base.py:4038`) returns quickly by spawning a background task, per its docstring (`:4039-4040`). The task is created at `:3956`, in `_start_session_processing` (`:3949`). The task calls `self._message_handler(event)` at `:4579`, inside `_process_message_background` (`:4562`).
3. The handler is set at `gateway/run_adapters.py:1355` from `_primary_message_handler` (`:1773`). With multiplexing on, that returns `_make_default_profile_message_handler` (`:1776`, defined `:1731-1743`), which calls `self._handle_message(event)`. With multiplexing off, it wraps `_handle_message` directly (`:1777`).
4. `_handle_message` (`gateway/run_inbound.py:1285`) runs admission and busy checks. It calls `_handle_message_with_agent` (`gateway/run_inbound.py:1368`, defined at `gateway/run_turn.py:2144`).
5. `_hmwa_prepare_turn` (`gateway/run_turn.py:2046`, call at `:2160`) reads the transcript at `:2096`. Then `_run_agent` (`:2197`, defined `:2928`) calls `_run_agent_inner` (`:2935`).
6. `_run_agent_inner` calls `_run_agent_build_turn_context` (`gateway/run_turn.py:4303`, defined `:3061`). That creates `TurnRunner` (`:3102`). `_run_agent_inner` starts `turn_runner.run_sync` in a worker (`:4344`).
7. `TurnRunner.run_sync` (`gateway/run_turn_runner.py:1884`) calls `_run_conversation_with_approval` (call at `:1949`, defined `:1684`). That passes the history as `conversation_history` (`:1697`) and calls `agent.run_conversation(api_message, **kwargs)` at `:1724`.

Adapter files (**counted**: 32 adapter classes in 32 files. Query: `git grep -E '^class [A-Za-z_0-9]*Adapter[(:]' <pin> -- gateway plugins/platforms`, minus `BasePlatformAdapter` and the helper `_AiohttpBridgeAdapter`). `Platform` values are in `gateway/config.py:217-242`.

Under `gateway/` (10, **counted**):
- `gateway/platforms/api_server.py:1172` `APIServerAdapter`. `API_SERVER` (`gateway/config.py:232`).
- `gateway/platforms/bluebubbles.py:105` `BlueBubblesAdapter`. `BLUEBUBBLES` (`:239`).
- `gateway/platforms/msgraph_webhook.py:99` `MSGraphWebhookAdapter`. `MSGRAPH_WEBHOOK` (`:234`).
- `gateway/platforms/qqbot/adapter.py:100` `QQAdapter`. `QQBOT` (`:240`).
- `gateway/platforms/signal.py:176` `SignalAdapter`. `SIGNAL` (`:226`).
- `gateway/platforms/webhook.py:195` `WebhookAdapter`. `WEBHOOK` (`:233`).
- `gateway/platforms/weixin.py:692` `WeixinAdapter`. `WEIXIN` (`:238`).
- `gateway/platforms/whatsapp_cloud.py:164` `WhatsAppCloudAdapter`. `WHATSAPP_CLOUD` (`:224`).
- `gateway/platforms/yuanbao.py:2594` `YuanbaoAdapter`. `YUANBAO` (`:241`).
- `gateway/relay/adapter.py:109` `RelayAdapter`. Experimental. `RELAY` (`:242`).

Under `plugins/platforms/` (22, **counted**, one per file). Telegram and Discord are here, not in `gateway/platforms/`:
- `plugins/platforms/a2a/adapter.py:260` `A2AAdapter`. Dynamic.
- `plugins/platforms/buzz/adapter.py:508` `BuzzAdapter`. Dynamic.
- `plugins/platforms/dingtalk/adapter.py:192` `DingTalkAdapter`. `DINGTALK` (`gateway/config.py:231`).
- `plugins/platforms/discord/adapter.py:1266` `DiscordAdapter`. `DISCORD` (`:222`).
- `plugins/platforms/email/adapter.py:449` `EmailAdapter`. `EMAIL` (`:229`).
- `plugins/platforms/feishu/adapter.py:1285` `FeishuAdapter`. `FEISHU` (`:235`).
- `plugins/platforms/google_chat/adapter.py:375` `GoogleChatAdapter`. Dynamic.
- `plugins/platforms/irc/adapter.py:121` `IRCAdapter`. Dynamic.
- `plugins/platforms/line/adapter.py:378` `LineAdapter`. Dynamic.
- `plugins/platforms/matrix/adapter.py:793` `MatrixAdapter`. `MATRIX` (`:228`).
- `plugins/platforms/mattermost/adapter.py:104` `MattermostAdapter`. `MATTERMOST` (`:227`).
- `plugins/platforms/ntfy/adapter.py:117` `NtfyAdapter`. Dynamic.
- `plugins/platforms/photon/adapter.py:558` `PhotonAdapter`. Dynamic.
- `plugins/platforms/raft/adapter.py:306` `RaftAdapter`. Dynamic.
- `plugins/platforms/simplex/adapter.py:115` `SimplexAdapter`. Dynamic.
- `plugins/platforms/slack/adapter.py:1006` `SlackAdapter`. `SLACK` (`gateway/config.py:225`).
- `plugins/platforms/sms/adapter.py:85` `SmsAdapter`. `SMS` (`:230`).
- `plugins/platforms/teams/adapter.py:350` `TeamsAdapter`. Dynamic.
- `plugins/platforms/telegram/adapter.py:514` `TelegramAdapter`. `TELEGRAM` (`gateway/config.py:221`).
- `plugins/platforms/wecom/adapter.py:102` `WeComAdapter`. `WECOM` (`gateway/config.py:236`).
- `plugins/platforms/wecom/callback_adapter.py:90` `WecomCallbackAdapter`. `WECOM_CALLBACK` (`:237`).
- `plugins/platforms/whatsapp/adapter.py:346` `WhatsAppAdapter`. `WHATSAPP` (`:223`).

Dynamic means no enum line. The enum says plugin platforms are created on demand (`gateway/config.py:218-219`).

## 3. Scheduled work (cron/)

Definition:
- `create_job` (`cron/jobs.py:1813`) takes a prompt, a schedule string, a name, a repeat count, and a delivery target. Schedule strings parse into `once`, `interval`, or `cron` kinds (`cron/jobs.py:788-789`). Payload fields are `prompt`, `script`, `skill`, `skills`, and `no_agent` (`cron/jobs.py:469`).
- The callers found use `create_job_with_scheduler_registration` (`cron/scheduler.py:4022`, which calls `create_job` at `:4027`). Callers: the cron tool (`tools/cronjob_tools.py:683`), blueprint CLI (`hermes_cli/blueprint_cmd.py:232`), dashboard (`hermes_cli/web_server_cron.py:165`), `tools/blueprints.py:145`, `cron/suggestions.py:184`, and the API server (`gateway/platforms/api_server.py:3908`).
- Field edits go through `update_job` (`cron/jobs.py:2120`, saves at `:2147`). `rearm_oneshot` (`:2259`, saves at `:2279`) and `rewrite_skill_refs` (`:3480`, saves at `:3501`) call `save_jobs` directly. Shipped definitions merge in through `import_job_definitions` (`cron/job_definition.py:48`).

Stored under `get_hermes_home()/cron` (`cron/jobs.py:85-89`):
- `jobs.json`: definitions and schedule state (`:89`). Saved by `save_jobs` (`:1602`) under a cross-process lock (`:19`), with an atomic replace (`:1586`).
- `output/<job_id>/`: per-job output directory (`:113`, `:426-435`). The code that writes the output file was not traced.
- `executions.db`: run ledger, table `executions` (`cron/executions.py:51`, schema `:60`). Opened with `synchronous_full=True` (`:53`).
- `deliveries.db`: delivery queue (`cron/delivery_queue.py:77-82`).

Triggered:
- The in-process ticker is `InProcessCronScheduler.start` (`cron/scheduler_provider.py:439`, default `interval=60`). Each cycle calls `cron_tick` (`:493`), which is `tick` (`cron/scheduler_tick.py:33`).
- The gateway runs the provider in a supervised thread (`gateway/run.py:5662-5670`). It passes no interval (`:5642-5653`), so the 60 s default applies (**not run**).
- The dashboard starts the same provider only when no live gateway owns cron (`hermes_cli/web_server.py:160-171`, `:268-274`).
- The wait between cycles is `_backoff_wait_seconds(interval, ...)` (`cron/scheduler_provider.py:525`). It doubles only after fd exhaustion (`:28-35`).
- Fire path in `cron/scheduler_provider.py`: `CronScheduler` defines `fire_due` (`:183`), `claim_fire` (`:200`), and `fire_claimed` (`:228`). The in-process code imports `run_one_job` (`:234`).

Output:
- `_deliver_result` (`cron/scheduler_delivery.py:1973`) resolves targets (`:1982`). It tries the live adapter first, then a standalone send (`:1976-1979`). Restart-safe workers queue the send for the gateway (`:1987-1989`) through `enqueue_and_wait` (`cron/delivery_queue.py:390`, called at `cron/scheduler_delivery.py:1999`).
- The gateway drains the queue with `drain_delivery_queue` (`cron/scheduler.py:1366-1383`), called from `gateway/run.py:4726` and `:4740`. It returns early when the queue file is missing (`cron/scheduler.py:1373-1374`).
- A due job waits for the next cycle. With the 60 s default, that is up to 60 s late (**not run**).

## 4. State (hermes_state*.py)

`SessionDB` is the store (`hermes_state.py:461`). Its methods live in mixins and helper modules, listed below. 34 top-level `hermes_state*.py` files (**counted**, `git ls-tree` piped to grep and `wc -l`).

One line per module (from each module's docstring):
- `hermes_state.py`: `SessionDB` class. Session metadata, message history, model usage.
- `hermes_state_common.py`: shared constants and DDL for the family (tables `:373-582`, FTS `:785-1017`).
- `hermes_state_compression.py`: compression lineage, cooldown and streak counters, locks, turn leases.
- `hermes_state_coverage.py`: which held rows in-place compaction covered.
- `hermes_state_dbfile.py`: file-level health helpers for state.db.
- `hermes_state_errors.py`: exception types and error classification.
- `hermes_state_fts.py`: FTS5 setup and the CJK bigram index DDL (`:46`).
- `hermes_state_gateway.py`: gateway routing index, peers, orphans, heartbeats, handoffs.
- `hermes_state_guard.py`: live-DB guard for tests, and the last init error per process.
- `hermes_state_health.py`: one storage-health latch per state.db path.
- `hermes_state_holders.py`: process and descriptor authority for structural maintenance.
- `hermes_state_identity.py`: durable message identity columns (schema v31).
- `hermes_state_ids.py`: session id minting (`YYYYMMDD_HHMMSS_<hex>`).
- `hermes_state_lockguard.py`: holds a writer's WAL file locks.
- `hermes_state_lockowners.py`: who holds the write lock now (Linux `/proc/locks`).
- `hermes_state_maintenance.py`: retention pruning, stale-session archiving, VACUUM policy.
- `hermes_state_messages.py`: transcript append, replace, rewind, reactions, conversation load (`get_messages_as_conversation` at `:1520`).
- `hermes_state_pidns.py`: PID-namespace identity for lock and lease holders.
- `hermes_state_portability.py`: session list, export, import.
- `hermes_state_profile_repair.py`: finders and fixers for crossed-profile rows.
- `hermes_state_readpool.py`: read-connection budgets for the WAL read path.
- `hermes_state_registry.py`: process-wide shared SessionDB registry.
- `hermes_state_repair.py`: repair, backup, writability preflight.
- `hermes_state_rewind.py`: carrier-aware user-turn rewind (`/undo`, `/retry`).
- `hermes_state_schema.py`: schema creation, column reconciliation, FTS DDL management.
- `hermes_state_search.py`: message search (FTS, trigram, CJK) and FTS maintenance.
- `hermes_state_sessions.py`: session lifecycle, row upsert, inheritance.
- `hermes_state_telegram.py`: Telegram DM topic-mode mixin.
- `hermes_state_timeline.py`: read-only prompt index and bounded transcript jumps.
- `hermes_state_titles.py`: session titles, sanitizing, provenance ranking.
- `hermes_state_tool_retries.py`: outcome records for the start_chat Retry card.
- `hermes_state_usage.py`: token and usage accounting, coalescing background writer.
- `hermes_state_user_copy.py`: plain-language storage-failure notices.
- `hermes_state_wal.py`: journal mode and PRAGMA policy (macOS `synchronous=FULL`, `:117-121`).

Tables (**counted**: 16 static names from `CREATE TABLE` and `CREATE VIRTUAL TABLE` over `hermes_state*.py`, plus 2 Telegram topic tables from a tuple, 18 total):
- Sessions and transcript: `schema_version` (`hermes_state_common.py:373`), `system_prompts` (`:377`), `sessions` (`:382`), `messages` (`:449`).
- Usage and meta: `session_model_usage` (`:482`), `state_meta` (`:504`).
- Gateway: `gateway_routing` (`:509`), `gateway_hygiene_state` (`:517`), `conversation_generations` (`:544`), `gateway_heartbeats` (`:559`).
- Compression and turn locks: `compression_locks` (`:568`), `session_turn_leases` (`:575`).
- Delegation: `async_delegations` (`:582`).
- Full-text (FTS5): `messages_fts` (`:785`), `messages_fts_trigram` (`:905`), `messages_fts_cjk` (`hermes_state_fts.py:46`).
- Telegram DM topic tables (from the tuple): `telegram_dm_topic_mode` (`hermes_state_telegram.py:30`), `telegram_dm_topic_bindings` (`:49`).

Compression state lives in `hermes_state_compression.py` (module docstring) and the two lock and lease tables. The durable successor pointer is read through `get_compression_tip` (`gateway/session_transcript.py:641`).

Default path:
- `~/.hermes/state.db`. `DEFAULT_DB_PATH = get_hermes_home() / "state.db"` (`hermes_state.py:178`). `_default_db_path()` re-resolves it at call time (`:198-201`).
- `get_hermes_home()` returns the context-local override first, then `HERMES_HOME` when set, else the platform default (`hermes_constants.py:111-118`, `:160-169`). The platform default is `~/.hermes` on macOS and Linux, `%LOCALAPPDATA%\hermes` on Windows, with `HERMES_DATA_DIR_SUFFIX` appended when set (`hermes_constants.py:51-58`).
- No config setting sets the path. `hermes_cli/config_defaults.py` mentions state.db only in comments (for example `:2262`, `:2276`). The keys in the `sessions` block that act on it are `auto_prune: True` and `retention_days: 90` (`hermes_cli/config_defaults.py:2264-2278`), plus `auto_archive: False` (`:2281`), `vacuum_after_prune: True` (`:2292`), and `min_interval_hours: 24` (`:2297`).

## 5. Which state reaches each model request

Gateway turn:
- `load_transcript` (`gateway/session_transcript.py:631`) calls `get_messages_as_conversation(..., repair_alternation=True)` (`:644-645`). The query is `ORDER BY id` with no row cap (`hermes_state_messages.py:1515-1518`). The only filter beyond session scope (`:1517`) is `AND active = 1` from `_active_clause` (`hermes_state_messages.py:1388-1390`, call at `:1531`). `_read_all` is a plain `fetchall()` (`hermes_state.py:1136-1138`).
- `_build_gateway_agent_history` (`gateway/run_turn_runner.py:1564`) builds the list. It goes into `run_conversation` as `conversation_history` (`:1697`, `:1724`).
- On a cache hit, `_resolve_turn_agent` (`:1139`) reuses the cached agent (`:1156-1157`). The code then compares the reloaded transcript with the agent's live messages and picks one (`:1572-1573`, helper `_select_cached_agent_history`). So the database read happens on a cache hit too.

CLI resume:
- `hermes_cli/cli_agent_setup_mixin.py:607-610` sets `self.conversation_history` from `get_messages_as_conversation(..., repair_alternation=True)`, minus `session_meta` rows.

Compacted rows:
- `include_compacted` defaults to `False` (`hermes_state_messages.py:1523`). The docstring says the model-fed restore must not regrow what compaction summarized (`:1524-1528`). Dedupe runs only when the flag is set (`:1532-1533`).

Memory files (on disk, not state.db):
- `agent/agent_init.py:1297-1309` loads MEMORY.md and USER.md at agent init.
- `agent/system_prompt.py:7-8` lists the volatile block: skills index, memory, and USER.md. `_memory_parts` (`:517`) builds the memory blocks.

On demand only:
- `session_search` (`tools/session_search_tool.py:619`, schema `:652`, registration `:787`). Text hints tell the model to call it after compaction (`agent/context_compressor.py:936`, `:998`). Apart from those hints, `session_search` appears in 38 lines across 20 files under `agent/` and `gateway/` (`git grep -n session_search <pin> -- agent gateway`). Most are guidance text, display, or tool-name tables. The inline executor is at `agent/inline_tool_executors.py:123`. No automatic history injection was found.

## 6. Top 3 speed-relevant findings

1. **Each gateway turn reads the full active transcript before the cache check.** Label: code read, not timed. Evidence: `gateway/run_turn.py:2096` (read), `:2160` (prepare, which runs before the agent call at `:2197`), `hermes_state_messages.py:1515-1518` (no row cap, `fetchall()` at `hermes_state.py:1136-1138`), `gateway/run_turn_runner.py:1156-1157` (cache lookup, after the read). Read cost grows with transcript length.

2. **The macOS state DB runs with `PRAGMA synchronous=FULL`.** Label: code read, cost not run. Evidence: `hermes_state_wal.py:117-121` (the function, whose docstring cites a checkpoint race that can corrupt btree pages under NORMAL), applied from `_apply_wal_companions` (`:124-128`), which is called at `:290`, `:330`, and `:442`. FULL trades commit speed for durability. The per-commit cost was not measured.

3. **Each message insert fires two FTS5 triggers, each gated by row role or session.** Label: code read, cost not run. Evidence: `messages_fts_insert` at `hermes_state_common.py:793` (rebuild-window condition at `:794-797`) and `messages_fts_trigram_insert` at `:913` (non-tool rows in eligible sessions, `:914-916`). A second `messages_fts_insert` definition appears at `:992`; which branch uses it was not traced. The config comment says unbounded growth "slows FTS5 inserts" (`hermes_cli/config_defaults.py:2262-2263`). Auto-prune defaults on, with 90 days retention (`:2269`, `:2278`). The comment is the authors' claim, not a measurement.

## 7. Not verified

- Nothing was run. No timings, fsync costs, or FTS insert costs were measured. Every performance claim above is a code read.
- SQLite behavior for `synchronous=FULL` in WAL mode was not checked against sqlite.org. Finding 2 relies on general SQLite knowledge.
- Who writes `cron/output/<job_id>/` was not traced.
- The API server's history source (`gateway/platforms/api_server.py:4186-4290`) and the CLI's new-turn history (`hermes_cli/cli_chat_turn_mixin.py:369`) were not traced.
- Receive code in the other adapters was not traced. The Telegram route stops at `_dispatch_with_text` (`plugins/platforms/telegram/adapter.py:6686`).
- How `hermes_cli.main.main` reaches `HermesCLI` was not traced.
- Other SQLite files named in `.gitignore:184` (kanban.db, response_store.db) were not traced.
- The installed build (`/Users/peterb/.hermes/hermes-agent`, `d604141`) was not compared. Every citation is at `97a9ae8ee5`.
- The source working tree is dirty (`AM .claude/autonomy.json`, `M .gitignore`, untracked `.claude/.session-start-*`). Citations come from the pinned commit only.
- `tests/hermes_state/` was used for names only.
