# surfaces.md review at hermes-agent 97a9ae8ee5

Verdicts over 183 units: SUPPORTED 164, PARTLY 11, NOT SUPPORTED 2, NO CITE 3, OK-LABELED 3.

Units by section (counted): §1 7, §2 62, §3 30, §4 52, §5 20, §6 11, §7 1. Sum 183.

## Findings (non-SUPPORTED rows only)

| # | doc line | sentence (first 12 words) | cite | verdict | evidence (file:line, quote) | fix |
|---|---|---|---|---|---|---|
| 1 | 10 | Cron jobs live in `~/.hermes/cron/jobs.json`. | none (the row's only cite is :439, for the ticker) | NO CITE | `cron/jobs.py:89` `JOBS_FILE = CRON_DIR / "jobs.json"`. Code agrees. | Add `cron/jobs.py:85-89` to the sentence. |
| 2 | 10 | Output goes to the live adapter or to a SQLite queue. | none (the row's only cite is :439) | NO CITE | `cron/scheduler_delivery.py:2107-2114` (live adapter, then `_deliver_standalone`); `cron/delivery_queue.py:82` `"cron" / "deliveries.db"`. Code agrees. | Add `cron/scheduler_delivery.py:2107-2114` and `cron/delivery_queue.py:77-82`. |
| 3 | 11 | Session and message state is one SQLite file, `~/.hermes/state.db`, behind `SessionDB` | none | NO CITE | `hermes_state.py:178` `DEFAULT_DB_PATH = _IMPORT_DEFAULT_DB_PATH = get_hermes_home() / "state.db"`; `hermes_state.py:461` `class SessionDB(`. Code agrees. | Add `hermes_state.py:178` and `:461`. |
| 4 | 25 | With multiplexing on, that returns `_make_default_profile_message_handler` (`:1776`, defined `:1731-1741`) | `gateway/run_adapters.py:1731-1741` | PARTLY | `gateway/run_adapters.py:1743` `return _handler` closes the def that starts at `:1731`. | Change `:1731-1741` to `:1731-1743`. |
| 5 | 31 | Adapter files (counted: 32 adapter classes in 32 files. | `git grep -E '^class [A-Za-z_0-9]*Adapter[(:]'` with no path | PARTLY | Query as written returns 232 lines in 173 files. With `-- gateway plugins/platforms` and the two helper classes removed: 32 lines in 32 files. | Add `-- gateway plugins/platforms` to the query. |
| 6 | 75 | Callers: the cron tool, blueprint CLI, dashboard, tools/blueprints.py, cron/suggestions.py. | `tools/cronjob_tools.py:683`, `hermes_cli/blueprint_cmd.py:232`, `hermes_cli/web_server_cron.py:165`, `tools/blueprints.py:145`, `cron/suggestions.py:184` | PARTLY | `gateway/platforms/api_server.py:3908` `_cron_create(**kwargs)` is a live caller (alias imported at `:1068`). | Add `gateway/platforms/api_server.py:3908` to the list. |
| 7 | 76 | Edits go through `update_job` (`cron/jobs.py:2111`). | `cron/jobs.py:2111` | NOT SUPPORTED | `cron/jobs.py:2279` `save_jobs(jobs)` inside `rearm_oneshot` (rewrites schedule); `cron/jobs.py:3501` `save_jobs(jobs)` inside `rewrite_skill_refs`. `remove_job` saves directly at `:2299`. | Say field edits from pause, resume, and trigger go through update_job (`:2158`, `:2197`, `:2219`), and that rearm_oneshot (`:2279`) and rewrite_skill_refs (`:3501`) call save_jobs directly. |
| 8 | 136 | Tables (counted, CREATE TABLE and CREATE VIRTUAL TABLE grep over hermes_state*.py): | grep method, no file:line | PARTLY | 16 static table names (re-count). Two more come from a tuple: `hermes_state_telegram.py:30` `"telegram_dm_topic_mode",` and `:49` `"telegram_dm_topic_bindings",`, created at `:131` `CREATE TABLE IF NOT EXISTS {table} ({ddl})`. Total 18. | Add both Telegram topic tables, noted as created on /topic opt-in (`hermes_state_telegram.py:119-122`). |
| 9 | 148 | `get_hermes_home()` returns `HERMES_HOME` when set, else the platform default | `hermes_constants.py:111-118`, `:160-169` | PARTLY | `hermes_constants.py:113` `override = get_hermes_home_override()` runs before the HERMES_HOME check (`:116`). | Add the context-local override as the first step. |
| 10 | 148 | The platform default is `~/.hermes`, plus `HERMES_DATA_DIR_SUFFIX` when set | `hermes_constants.py:51-58` | PARTLY | `hermes_constants.py:54` `if sys.platform == "win32":` takes the `LOCALAPPDATA` base (`:55-57`). | Say `~/.hermes` on macOS and Linux, `%LOCALAPPDATA%\hermes` on Windows. |
| 11 | 149 | The keys that act on it are `sessions.auto_prune` and `sessions.retention_days` | `hermes_cli/config_defaults.py:2264-2278` | PARTLY | Same `sessions` block also has `"vacuum_after_prune": True,` (`:2292`), `"auto_archive": False,` (`:2281`), and `"min_interval_hours": 24,` (`:2297`). | List these, or narrow the sentence to the retention keys. |
| 12 | 154 | The only filter is `AND active = 1` from `_active_clause` | `hermes_state_messages.py:1388-1390`, `:1531` | PARTLY | `hermes_state_messages.py:1517` `FROM messages WHERE session_id IN` is also a row filter. | Say the only filter beyond session scope is `AND active = 1`. |
| 13 | 156 | On a cache hit, `_resolve_turn_agent` reuses the cached agent (`:1152-1153`). | `gateway/run_turn_runner.py:1139`, `:1152-1153` | PARTLY | `gateway/run_turn_runner.py:1152-1153` only fetch `_agent_cache_lock` and `_agent_cache`. Reuse is at `:1156-1157`: `found = self._lookup_cached_agent(sig, cache_lock, cache`. | Change `:1152-1153` to `:1156-1157`. |
| 14 | 173 | Each gateway turn reads the full active transcript before the cache check. | `gateway/run_turn.py:2096`; `gateway/run_turn_runner.py:1152-1153` | PARTLY | Read at `gateway/run_turn.py:2096` checks out. The cache cite `:1152-1153` only fetches cache handles. The lookup is at `gateway/run_turn_runner.py:1156`. | Change `run_turn_runner.py:1152-1153` to `:1156-1157`. |
| 15 | 177 | Each message insert fires FTS5 index triggers. | `hermes_state_common.py:793`, `:913` | PARTLY | `hermes_state_common.py:914` `WHEN new.role <> 'tool'` and the session predicate (`:915-916`) gate the trigram trigger. `messages_fts_insert` has a rebuild-window WHEN (`:794-797`). | Say inserts fire messages_fts_insert (outside an FTS rebuild window) and messages_fts_trigram_insert (non-tool rows in eligible sessions). |
| 16 | 169 | Apart from those hints, the only match in agent/ and gateway/ is the inline executor | `agent/inline_tool_executors.py:123` | NOT SUPPORTED | 38 matching lines in 20 files (re-count). Non-hint hits include `agent/system_prompt.py:294` `SESSION_SEARCH_GUIDANCE if "session_search" in names` and `gateway/session.py:640` (prompt text, "use the session_search tool to recall that prior session"). | Drop "the only match". State the 38 lines in 20 files, cite the executor at `inline_tool_executors.py:123`, and say the rest are guidance, display, and tool-name tables. |
| 17 | 94 | With the 60 s default, that is up to 60 s late (not run). | none | OK-LABELED | `cron/scheduler_provider.py:439` `interval=60` is the default. Lateness is runtime timing, labeled not run. | None. |
| 18 | 173 | Read cost grows with transcript length. | `hermes_state_messages.py:1515-1518` | OK-LABELED | Code has no row cap (`:1518` `ORDER BY id`). Growth is runtime timing. The label reads "not timed", not "not run". | Keep. Optional: change the label to "not run" to match the legend. |
| 19 | 175 | FULL trades commit speed for durability. | none | OK-LABELED | No code evidence. General SQLite behavior. §7 (line 182) says it was not checked against sqlite.org. | None. |

## Re-counts

Repo `G=/Users/peterb/Documents/AI/hermes-agent`, pin `REF=97a9ae8ee5`. Each line is the command, then the result.

- `git -C "$G" ls-tree --name-only "$REF" | grep -E '^hermes_state.*\.py$' | wc -l`: 34
- `grep -c '^- \`hermes_state' docs/surfaces.md` (doc §4 rows): 34
- `git -C "$G" grep -n -E '^class [A-Za-z_0-9]*Adapter[(:]' "$REF" | wc -l`: 232 (173 files via `grep -l`)
- `git -C "$G" grep -n -E '^class [A-Za-z_0-9]*Adapter[(:]' "$REF" -- gateway plugins/platforms | grep -v -E 'class (BasePlatformAdapter|_AiohttpBridgeAdapter)[(:]' | wc -l`: 32 lines, 32 distinct files (`cut -d: -f2 | sort -u | wc -l`)
- Same pipeline with `-- gateway`: 10. With `-- plugins/platforms`: 22.
- `grep -c '^- \`gateway/' docs/surfaces.md`: 10. `grep -c '^- \`plugins/platforms/' docs/surfaces.md`: 22.
- `git -C "$G" grep -h -o -E 'CREATE (VIRTUAL )?TABLE IF NOT EXISTS [a-z_0-9]+' "$REF" -- 'hermes_state*.py' | awk '{print $NF}' | sort -u | wc -l`: 16 distinct static names
- `git -C "$G" grep -n -E '^        "telegram_dm_' "$REF" -- hermes_state_telegram.py | wc -l`: 2 (the `_TOPIC_TABLES` tuple). Static 16 plus 2 is 18.
- `git -C "$G" grep -n session_search "$REF" -- agent gateway | wc -l`: 38. Same with `-l | wc -l`: 20 files.
- `grep -n 'est\.' docs/surfaces.md`: line 5 (the legend) only. The doc has no "est." number, so no arithmetic to re-check.

## Method and judgment calls

- Read-only. `git show` and `git grep` at `97a9ae8ee5` only. No checkout, no writes in hermes-agent, no hermes run, no model, no network, no ssh. docs/surfaces.md not edited.
- Each behavior cite was read with about 10 lines of context. Locator cites (class, enum, CREATE TABLE, caller lines) were checked with `git grep -n` and the exact line.
- Units are sentences or list items that make a claim. Headings, labels, and §7 process notes are not counted. The §7 `.gitignore:184` names were checked and are SUPPORTED.
- OK-LABELED only where the doc says "not run", "cost not run", "not timed", or §7 flags the claim.
- Kept as SUPPORTED, with the cite judged close enough: `_build_gateway_agent_history` at run_turn_runner.py:1564 (the call site; the def is gateway/run.py:1245). `agent/agent_init.py:1297-1309` (a comment says memory is loaded; the load call is at :1333). `_memory_parts` at :517 (its docstring line).
- Not checked: §7 working-tree status and the installed build. These are state claims, not behavior claims about the pinned source.
