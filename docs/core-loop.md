# Hermes core loop map

Read only. Nothing was run: no hermes, no model, no network, no ssh. Nothing under `hermes/` or `hermes-agent/` was written.

## Sources and labels

- **HEAD**: `/Users/peterb/Documents/AI/hermes-agent` at `97a9ae8ee5` (2026-10-11, tag `rc.11-v0.21.7`, 55 commits past it). Code citations are HEAD unless marked. Paths are relative to that root.
- **59004a623** (2026-09-25): Peter's ai-core build per his notes. An ancestor of HEAD. HEAD has 7,742 more commits (counted with `git rev-list`). Peter's line numbers match this commit (checked: `agent/system_prompt.py:41-52` holds `_model_gate`, `agent/prompt_builder.py:359-370` holds `TOOL_USE_ENFORCEMENT_MODELS`).
- **d604141** (2026-07-21): installed build at `/Users/peterb/.hermes/hermes-agent`. An ancestor of 59004a623. Grep checks only, no diff.
- Labels: **counted** = grep or AST count of source. **est.** = chars/4 of string literals, or arithmetic. **not run** = not executed. Config values are marked as config values.

## Short version

- One loop iteration makes one model call, plus any retries inside it (`agent/turn_iteration_prep.py:402`). Tool calls loop back. A text answer ends the turn unless a continuation guard fires (`agent/turn_final_response.py:249`, `:318`).
- Default config defers six core tools (`hermes_cli/config_defaults.py:2018-2025`) and every MCP tool (`tools/tool_search.py:149-161`) behind a three-tool bridge. A simple MCP lookup takes 3 to 4 model calls (est., not run; tool_search, tool_describe, tool_call, then text). The floor is 2 calls (est., not run; tool_call, then text).
- Three things add calls with no tool result: the stall guard (`agent/turn_final_response.py:172-249`), skill loads the prompt asks for (path not traced), and background review after the turn (`agent/turn_facade.py:41`).
- Fixed prompt text is about 1,317 to 1,358 tokens (est.) per request. The skills index, memory, and tool schemas come on top.

## 1. Entry points

- **CLI, interactive**: `hermes` wrapper (`hermes:1-11`) runs `hermes_cli/main.py`. The chat turn is `hermes_cli/cli_chat_turn_mixin.py:49` `chat()`, which calls `self.agent.run_conversation` at `:369`.
- **CLI, single query**: `hermes_cli/cli_single_query.py:84, :223, :246, :275` (`run_conversation`) and `:567` (`chat`).
- **One-shot**: `hermes_cli/oneshot.py:609`.
- **Gateway (Telegram and other platforms)**: `gateway/run_turn_runner.py:1724` and `gateway/run_turn.py:2489` both call `agent.run_conversation`. Which one a message takes was not traced. The inbound side is `gateway/run_inbound.py`.
- **API server**: `gateway/platforms/api_server.py:4290`. **TUI**: `tui_gateway/prompt_turn.py:811`, `tui_gateway/methods_prompt.py:1106`. **ACP**: `acp_adapter/server.py:810`. **Cron**: `cron/scheduler.py:1977` on a worker thread.
- **Direct script**: `run_agent.py:1470` `main()`, calling `run_conversation` at `:1536`.
- Everything lands on `AIAgent` (`run_agent.py:214`), a class built from mixins. The turn entry is `agent/turn_facade.py:22`.

## 2. The agent loop

Call chain:

1. `agent/turn_facade.py:22` `TurnFacadeMixin.run_conversation`. Cancels a running background review (`:41`). Takes the durable turn lease (`:83`).
2. `agent/conversation_loop.py:1709` `run_conversation`. Thin wrapper that stamps the turn boundary.
3. `agent/conversation_loop.py:1547` `_run_conversation_turn`. Builds per-turn context, then runs the loop.

The loop is `agent/conversation_loop.py:1648-1696`.

- **Condition** (`:1648-1650`): `api_call_count < max_iterations` and `iteration_budget.remaining > 0`, or the grace flag is set.
- **One pass per model call**, in this order:
  1. `begin_iteration` (`agent/turn_iteration_prep.py:352`). Interrupt check at `:382` breaks the loop. Iteration budget consumed at `:412`. `api_call_count += 1` at `:402`.
  2. `prepare_iteration` (`agent/turn_iteration_prep.py:144`). Skill-nudge counter at `:166-167`. Drains `/steer`. Adds wrap-up notices.
  3. `assemble_api_request` (`agent/turn_request_assembly.py:106`). Builds `api_messages`, the prompt-cache plan (`:205-230`), and a size estimate (`:241-259`).
  4. `run_preflight_gate` (`agent/turn_preflight_gate.py:20`). Compresses first if the request is over threshold.
  5. `_run_api_retry_loop` (`agent/conversation_loop.py:1497-1526`). Calls `build_api_request` (`agent/turn_api_request.py:93`), then `perform_api_call` (`agent/turn_api_call.py:67`, streams by default via `_should_stream` at `:46`), then `check_api_response` (`agent/turn_response_check.py:92`). Errors go to `handle_api_error` (`agent/turn_api_error.py:52`).
  6. `apply_retry_restarts` (`agent/turn_iteration_prep.py:437`).
  7. `normalize_model_response` (`agent/turn_response_intake.py:120`).
  8. Branch on the response:
     - **Tool calls present**: `run_tool_round` (`agent/turn_tool_round.py:46`). Writes the assistant tool-call row to SQLite before any tool runs (`:127-130`). Runs tools through `agent._execute_tool_calls` (`:163`, defined at `run_agent.py:1294`). Runs `compress_after_tool_results` (`:202`). Continues (`:230`).
     - **No tool calls**: `finish_text_response` (`agent/turn_final_response.py:56`). Nudge checks at `:180-205` and `:291-318`. Breaks the loop at `:376`.
- After the loop: `finalize_turn` (`agent/turn_finalizer.py:525`), called at `agent/conversation_loop.py:1698`.

**Stop conditions**

- Text answer: `agent/turn_final_response.py:376`.
- Interrupt: `agent/turn_iteration_prep.py:382`.
- Iteration budget spent: the loop breaks at `agent/turn_iteration_prep.py:416`. A grace call follows only a codex reasoning-only fallback (`agent/turn_truncation.py:635`).
- Error and recovery exits: `turn_api_error.py`, `turn_recovery*.py` (not traced in full).
- No default cap on calls. `run_agent.py:241` defaults `max_iterations` to `sys.maxsize`. `agent.max_turns` defaults to `None` (`hermes_cli/config_defaults.py:78`), which resolves to `sys.maxsize` (`hermes_cli/config.py:1833`). The area guide says the default is 500 (`agent/AGENTS.md`). That is stale.

**Parallel tools**: parallel-safe runs go to up to 8 workers (`agent/tool_executor.py:130`, `:240-247`). Other calls run one at a time. The planner is described at `run_agent.py:1294-1300`.

## 3. Prompt assembly

The system prompt is built once per session into three cached tiers: stable, context, volatile (`agent/system_prompt.py:734-797`). It stays cached until compression invalidates it (`agent/system_prompt.py:800-831`). The whole system prompt goes into every request (`agent/turn_context.py:1331-1344`).

| # | Piece | Source | When it loads | Size |
|---|---|---|---|---|
| 1 | SOUL.md identity | `agent/prompt_builder.py:1653`; `agent/system_prompt.py:544-550` | File exists in HERMES_HOME. Stable tier. | File size. Capped (`agent/prompt_builder.py:1161`, `:1171-1191`). |
| 2 | Default identity | `agent/prompt_builder.py:159` | No SOUL.md | 663 chars, ~165 tokens (est.) |
| 3 | Help guidance | `agent/prompt_builder.py:170`, `:182`; `agent/system_prompt.py:746-757` | Always. The no-skills variant (`:182`) is used when skill tools or the hermes-agent skill are missing. | 600 chars, ~150 tokens (est.). No-skills variant: 466 chars. |
| 4 | Task completion guidance | `agent/prompt_builder.py:382`; `agent/system_prompt.py:556-562`; flag `hermes_cli/config_defaults.py:184` | Tools visible and flag on (default on) | 769 chars, ~192 tokens (est.) |
| 5 | Parallel tool-call guidance | `agent/prompt_builder.py:419`; `agent/system_prompt.py:556-562`; flag `hermes_cli/config_defaults.py:188` | Same as row 4 | 618 chars, ~154 tokens (est.) |
| 6 | Steer channel note | `agent/prompt_builder.py:603`; `agent/system_prompt.py:566-567` | Tools visible | 771 chars rendered (counted; the literal template is 572), about 193 tokens (est.) |
| 7 | Memory guidance | `agent/prompt_builder.py:192-230`; `agent/system_prompt.py:281-286` | Memory tool visible (eager by default) | 964 to 1,128 chars (1,036 at defaults), about 241 to 282 tokens (est.) |
| 8 | Session search guidance | `agent/prompt_builder.py:237`; `agent/system_prompt.py:294` | Only if `session_search` is visible. Deferred by default, so usually absent. | 186 chars, ~46 tokens (est.) |
| 9 | Skills guidance | `agent/prompt_builder.py:258`; `agent/system_prompt.py:295` | `skill_manage` visible (eager by default) | 441 chars rendered (counted; literal is 421), about 110 tokens (est.) |
| 10 | Async handoff guidance | `agent/prompt_builder.py:394`; `agent/system_prompt.py:581-582` | `delegate_task` visible. Gone when delegation is disabled. | 487 chars, ~121 tokens (est.) |
| 11 | Tool-use enforcement, execution, Google guidance | `agent/prompt_builder.py:347`, `:361`, `:375`, `:544`; `agent/system_prompt.py:572-578` | Only when the model name contains a listed substring | 824 chars (`agent/prompt_builder.py:347`) and 860 chars (`:544`), counted. Applies only when the setting is auto (default) and the model name matches. Off for Peter's model name per his note (not re-checked). |
| 12 | Coding brief and git snapshot | `agent/coding_context.py:30`, `:364`, `:399`; `agent/system_prompt.py:664-693` | Only cli, tui, acp, desktop, and the empty platform string, with a code workspace. Not Telegram (`agent/coding_context.py:30`). | Not counted |
| 13 | Skills index | `agent/prompt_builder.py:1361-1600`; header `:1433-1495`; `agent/system_prompt.py:301-314`, `:783` | A skill tool is visible. Volatile tier. | Header 1,175 to 1,189 chars (counted, `agent/prompt_builder.py:1475-1495`). Hidden notes add 175 chars (`:1445`) and about 201 chars plus skill names (`:1450`, est.), so the full header is about 1,551 to 1,565 chars (est.). Plus one line per visible skill (not counted). This repo ships 58 bundled `SKILL.md` files (counted). |
| 14 | Pinned skills (`skills.auto_load`) | `agent/system_prompt.py:317-341` | Only if configured | Not counted |
| 15 | MEMORY.md and USER.md | `agent/system_prompt.py:516-541`; limits `hermes_cli/config_defaults.py:1319-1320` | Memory enabled. Volatile tier. | Limits 2,200 and 1,375 chars (config values). The config comment says ~800 and ~500 tokens. |
| 16 | External memory provider block | `agent/system_prompt.py:529-540`; `agent/memory_manager.py:501` | Only with a provider | Not counted |
| 17 | Project context file | `agent/prompt_builder.py:1857`; `agent/system_prompt.py:713-726` | First found wins among `.hermes.md` or `HERMES.md` (walks to the git root), `AGENTS.md`, `CLAUDE.md`, `.cursorrules`. `.cursor/rules/*.mdc` also loads (`agent/prompt_builder.py:1863-1864`) | Floor 20,000 chars. Scales with the window, ceiling 500,000 (`agent/prompt_builder.py:1161`, `:1171-1191`). |
| 18 | Timestamp, profile line, env hints, env probe | `agent/system_prompt.py:489`, `:373`, `:701-706`; `agent/prompt_builder.py:1142` | Always. Env probe on by default (`hermes_cli/config_defaults.py:192`). | Small. Not counted. |
| 19 | Telegram platform hint | `agent/prompt_builder.py:681` (dict), `:696` (telegram value); `agent/system_prompt.py:445`, `:709` | Always on Telegram | 617 chars rendered (counted; the literal template is 535), about 154 tokens (est.) |
| 20 | Telegram rich-messages add-on | `agent/prompt_builder.py:852`; `agent/system_prompt.py:459-472` | Only when `_telegram_rich_messages_enabled()` is true. Default not traced. | 863 chars, ~216 tokens (est.) |
| 21 | Caller system message, ephemeral system prompt | `agent/system_prompt.py:771-772`; `agent/turn_context.py:1339-1341` | Only when set | Not counted |

**Outside the system prompt, per request**

- Memory recall and `pre_llm_call` plugin text go into the current user message at API time (`agent/turn_context.py:108-133`, `:793-849`, `:893-931`). Memory recall skips trivial prompts (`agent/turn_context.py:921`); `pre_llm_call` text does not. The system prompt stays byte-stable.
- MoA aggregate (`agent/turn_request_assembly.py:129-130`) and prefill rows (`:133-138`), only when configured.

**Fixed blocks, Telegram with tools, memory, skill tools, and delegation on.** Rows 3 to 7, 9, 10, and 19. Rendered total 5,267 to 5,431 chars (counted; literal-only would be 4,966 to 5,130), about 1,317 to 1,358 tokens (est., chars/4). The skills header, memory, context file, and tool schemas come on top.

## 4. Tool schemas

- **Build path**: `model_tools.py:214` `get_tool_definitions` (memoized, `:228-277`) calls `_compute_tool_definitions` (`:502`). That selects names (`:315`), filters by `check_fn` (`tools/registry.py:843`), sanitizes, then runs `tools/tool_search.py:350` `assemble_tool_defs`.
- **Stored once** as `agent.tools` (`agent/agent_init.py:1116-1122`). Each request sends that list (`agent/turn_request_assembly.py:205`). `valid_tool_names` is built from it (`agent/agent_init.py:1124`), so deferred tools are not in it.
- **MCP tools** are appended between turns, append-only (`agent/turn_context.py:550-570`).

**Counts** (counted from source, before `check_fn` gates)

- Core list: **55** names (`toolsets.py:11-41`). Platform bundles built by `_bundle` start from it (`toolsets.py:55-57`). `hermes-telegram` adds no extras (`toolsets.py:208`).
- `kanban_*` names: **14**. Only for kanban workers and profiles with the kanban toolset (`tools/kanban_tools.py:84-100`, gate at `:1315`). Not in chat.
- Default-deferred: **18** names (`hermes_cli/config_defaults.py:2018-2025`, counted). Six are core tools: `session_search`, `todo_list`, `process_manage`, `cronjob_manage`, `image_generate`, `computer_use`. Twelve are desktop GUI names, such as `drive_preview`.
- Eager candidates for the default bundle: **49** (55 core names minus 6 deferred; `toolsets.py:11-41`, `hermes_cli/config_defaults.py:2018-2025`). Plus 3 bridge tools when the bridge is on, so up to **52** names per request before `check_fn` gates. The 14 kanban names drop out only at `check_fn` (`tools/kanban_tools.py:90`), so the count after that is lower (not run). Browser, voice, and connector tools may drop out too.

**Deferral rule**: `tools/tool_search.py:149-168`. A tool defers if it is in the defer list, or it is an MCP tool (toolset starts with `mcp-`), or it is a plugin tool. MCP tools defer no matter what the list says. The 59004a623 build has the same rule (`tools/tool_search.py:150-162`).

**Bridge**: `tools/tool_search.py:196-205` turns the bridge on whenever any deferrable tool exists. `enabled` is `"auto"` or `"on"`. `"off"` never activates. Three tools replace the deferred ones:

- `tool_search`: keyword queries. Default 5 hits, max 25 (config values at `hermes_cli/config_defaults.py:2004-2006`).
- `tool_describe`: loads full schemas. The bridge text says to call it first when parameters are unknown (`tools/tool_search.py:273` onward).
- `tool_call`: an array of `{name, arguments}`.

The listing embedded in the search description is capped at 4,000 tokens (config value, `hermes_cli/config_defaults.py:2014`; budget logic at `tools/tool_search.py:208-213`). It goes out on every request.

**Schema bytes**: not measured (not run). Literal `"parameters": {` blocks appear in 48 files under `tools/` (counted with `git grep -l`), so some schemas are literal dicts. Peter's note says 41,218 chars for the 19-tool array on ai-core. Not checked here.

## 5. History and compression

- **Storage**: SQLite `SessionDB` (`hermes_state.py:461`). A flush happens before each tool round (`agent/turn_tool_round.py:127-130`) and at turn end (`agent/turn_final_response.py:364`).
- **Replay**: a resumed session loads its persisted history (`agent/turn_context.py:705-751`). Every request sends the full transcript. Only the replayed prefix (rows before this turn's user message) is canonicalized (`agent/turn_context.py:1236-1263`). No sliding window. On the default path only compression shortens it (`agent/AGENTS.md:59`). Two other paths truncate: `agent/codex_runtime_history_seed.py:63-64` and `gateway/platforms/api_server.py:406`.
- **Trigger**: `compression.threshold` is 0.50 (`hermes_cli/config_defaults.py:589`; same value in 59004a623 at `:565`). Windows under 512K use at least 0.75 (`agent/context_compressor.py:2757-2758`; constants at `:1150-1151`). `ContextCompressor` derives the trigger from the window (`agent/context_compressor.py:2651`, used at `:2691`). `should_compress_info` (`:2998`) compares the last prompt token count with `threshold_tokens`.
  - Before each request: `agent/turn_preflight_gate.py:20`.
  - After each tool round: `agent/turn_preflight.py:244`, called at `agent/turn_tool_round.py:202`.
- **Method**: prune old tool results first, with no model call. Then a structured summary from the compression aux model (`agent/AGENTS.md`, compression section). That summary is one extra model call (`agent/AGENTS.md:85-86`). On Peter's box it shares the one engine slot (his notes, not run). Aux timeout 120 s (`hermes_cli/config_defaults.py:765`). Retry rounds capped at 3 (`hermes_cli/config_defaults.py:609`).
- **Gateway hygiene**: 0.85 threshold and a 5,000-message hard limit (`gateway/run_turn.py:679-685`; `hermes_cli/config_defaults.py:633`).
- **Micro-compaction**: off by default (`hermes_cli/config_defaults.py:626`).
- **Caps before history**: terminal output 50,000 chars (`hermes_cli/config_defaults.py:552-555`, `max_bytes`). A `read_file` call may request at most 2,000 lines (`max_lines`, `:553`, same block). File reads 100,000 chars (`hermes_cli/config_defaults.py:528`).

## 6. Rounds per turn: "check my inbox"

Assumes Telegram, default config, a gmail MCP tool, and an email skill that matches.

- **Floor: 2 model calls.** Call 1 returns a tool call. Call 2 returns text (`agent/turn_tool_round.py:230`; `agent/turn_final_response.py:376`). This needs the tool to be eager. Default config does not make MCP tools eager (`hermes_cli/config_defaults.py:1999` sets `enabled: auto`; MCP tools are deferrable at `tools/tool_search.py:149-161`).
- **Default, MCP deferred: 3 to 4 calls** (bridge at `tools/tool_search.py:273`, loop at `agent/conversation_loop.py:1648`). `tool_search`, then `tool_describe` (skippable when the model already knows the parameters), then `tool_call`, then the answer.
- **Skill mandate**: `agent/prompt_builder.py:1475-1494` tells the model to load any matching skill with `skill_view` before acting. Each load adds one call (`agent/turn_tool_round.py:230`). The model decides.
- **Stall guard**: if the answer ends on a phrase like "let me now", +1 call (`agent/agent_runtime_helpers.py:3428-3431`, text up to 400 chars). Max 2 per turn (`agent/turn_final_response.py:183`, `:196`, `:208`). Logged at `:220`. On by default (`hermes_cli/config_defaults.py:181`).
- **Dropped tool call** (`finish_reason` says tool calls, none present): +1 call, max 3 in a row (`agent/turn_final_response.py:291-294`).
- **Empty response**: +1 call per retry, budget 3 by default, or 1 when one attempt's estimated cost passes the threshold (default $0.25; `agent/empty_response_guard.py:56`, `:230-236`), jittered wait 5 to 60 s (`agent/empty_response_guard.py:54`; `agent/turn_empty_response.py:46-70`). Post-tool empty nudge: +1 once per tool round (`agent/turn_empty_response.py:198-220`).
- **Open reasoning tag**: up to 2 retries (`agent/turn_response_intake.py:163-170`).
- **Over-threshold request**: +1 summary call (`agent/turn_preflight_gate.py:20`).
- **Approval guardian**: an LLM call only for commands the danger check flags (`approvals.mode` is `"smart"`, `hermes_cli/config_defaults.py:1679`; `tools/approval.py:753`). Not traced for MCP calls.

**Side calls, not loop rounds**

- **Background review**, after delivery (`agent/turn_finalizer.py:763-790`). Forks a full `AIAgent` with a 16-iteration cap (`agent/background_review.py:173`, fork at `:977`). Triggers:
  - Every 10 user turns (`agent/turn_context.py:752-763`; config value `hermes_cli/config_defaults.py:1322`).
  - After 10 tool iterations without `skill_manage` (`agent/agent_init.py:1380-1382`; `agent/turn_iteration_prep.py:166-167`; `agent/turn_finalizer.py:763-768`).
  - On by default (`hermes_cli/config_defaults.py:811`). The code comment at `agent/turn_finalizer.py:779` says the fork costs about 30K tokens per event (not measured).
- **Title generation**: first 1 to 3 turns of a session with a placeholder title (`agent/turn_context.py:196-232`; `agent/title_generator.py:736`; `hermes_cli/config_defaults.py:774-776`). Untitled sessions are retried each turn (`agent/title_generator.py:756`). The code says a placeholder gets turns 2-3, at most three calls (`:757`).

**Not on the default path**

- Scripted prelude: no-op without a caller (`agent/turn_scripted_prelude.py:53-58`). No production caller; `agent/conversation_loop.py:1758` and `agent/turn_facade.py:157` are plumbing only.
- Micro-compaction: off (`hermes_cli/config_defaults.py:626`).
- Verify-on-stop: off (`hermes_cli/config_defaults.py:221`).
- Kanban stop guard: kanban workers only (`agent/turn_stop_gates.py:79`).
- Memory query rewrite: only with its memory plugin (`plugins/memory/query_rewrite.py:5`).
- Curator: inactivity-triggered skill review, not per turn (`agent/curator.py:3`).

**Total for this request**: 2 at the floor (`agent/turn_tool_round.py:230`). 3 to 4 with the default MCP deferral (`tools/tool_search.py:273`). Add 1 per skill load and 1 per stall (max 2). Peter's note says 5 calls (not run here). That fits the range.

## 7. Top 5 cost cuts

Ranked by model rounds removed per turn. Costs are estimates from code paths, not runs.

1. **Tool deferral.** `tools/tool_search.py:149-168`, `:196-205`. Defer list `hermes_cli/config_defaults.py:2018-2025`. Bridge skip at `model_tools.py:532-533`.
   - Cost: 1 to 2 extra calls per deferred tool use (`tools/tool_search.py:273`). Listing up to 4,000 tokens on every request (config value, `hermes_cli/config_defaults.py:2014`).
   - Fix: `tools.tool_search.enabled: "off"` (`hermes_cli/config_defaults.py:1991-1999`). Skips assembly, so every tool is eager again, including the six default-deferred core tools.
   - Risk: a bigger tool array on every request. Size not measured. A small model may pick tools less well with more visible tools (not measured).

2. **Stall guard.** `agent/turn_final_response.py:180-205`. Trailing check `agent/agent_runtime_helpers.py:3428-3431`. Config `agent.stall_guards` (`hermes_cli/config_defaults.py:181`, read at `agent/agent_init.py:1418-1421`).
   - Cost: +1 call per stall, max 2 per turn (est.).
   - Fix: `agent.stall_guards: false`. Config only.
   - Risk: a model that says "let me now" and stops will end the turn with no action. The config comment says the guard exists for that failure (`hermes_cli/config_defaults.py:178-181`). Count the "Stall guard" log lines in `agent.log` before turning it off.

3. **Skill mandate and index.** `agent/prompt_builder.py:1475-1494`. Header `:1433-1495`. Index build `agent/system_prompt.py:301-314`.
   - Cost: +1 call per skill loaded (`agent/turn_tool_round.py:230`). Header about 300 tokens (est.; about 1,190 chars at `agent/prompt_builder.py:1475-1495`), plus one line per visible skill. It sits in the volatile tier, so any change re-prefills from there (`agent/system_prompt.py:781-783`).
   - Fix: shorten the mandate text, hide unused skills, shorten descriptions. Peter's D1 and H6 plans aim here.
   - Risk: the model misses a skill it needed. Skills hold procedures, so a miss costs quality.

4. **Background review.** `agent/turn_finalizer.py:763-790`. On by default `hermes_cli/config_defaults.py:811`. Memory interval `hermes_cli/config_defaults.py:1322`. Skill interval `agent/agent_init.py:1380-1382`. Cap `agent/background_review.py:173`.
   - Cost: up to 16 iterations per trigger. Replays the conversation when the review model is `auto` (`hermes_cli/config_defaults.py:800-811`). It runs after delivery, but on the same engine slot as the next turn. Peter's engine is one slot per his notes (not run here).
   - Fix: `auxiliary.background_review.enabled: false` (Peter's A1). Or raise `memory.nudge_interval` and `skills.creation_nudge_interval`.
   - Risk: no automatic memory or skill saves. The config comment says `/refine` still works (`hermes_cli/config_defaults.py:800-811`).

5. **Fixed guidance blocks.** `agent/prompt_builder.py:382`, `:419`, `:603`, `:192`, `:258`, `:394`. Gates in `agent/system_prompt.py:553-583`.
   - Cost: about 1,317 to 1,358 tokens (est.) per request with the Telegram hint. Cheap while the prefix caches. A changed prefix re-prefills from there (`agent/system_prompt.py:781-782`).
   - Fix: `agent.task_completion_guidance: false` (`hermes_cli/config_defaults.py:184`) and `agent.parallel_tool_call_guidance: false` (`:188`). Delegation off already drops the async block (Peter did this).
   - Risk: the task-completion block pushes persistence. The parallel block drives parallel calls, and cutting it can cost time. Test one at a time.

Title generation is the sixth cost item and cheap to cut. `auxiliary.title_generation.model_upgrade_enabled: false` keeps the derived title (`hermes_cli/config_defaults.py:776` comment). It is not in the top five.

## Versus the other builds

| Behavior | d604141 (installed) | 59004a623 (ai-core per notes) | HEAD 97a9ae8ee5 |
|---|---|---|---|
| Loop condition | `agent/conversation_loop.py:721` | `agent/conversation_loop.py:1595` | `agent/conversation_loop.py:1648` |
| Bridge turns on | deferrable tokens >= 10% of window (`tools/tool_search.py:234-258`) | any deferrable tool (`tools/tool_search.py:197-208`) | any deferrable tool (`tools/tool_search.py:196-205`) |
| Core tools deferred by default | none found | six (`hermes_cli/config_defaults.py:1990`) | six (`hermes_cli/config_defaults.py:2018-2025`) |
| Stall continuation | not found | `agent/turn_final_response.py:170` | `agent/turn_final_response.py:180-205` |
| Background review on by default | setting present (`hermes_cli/config.py:1809`) | `hermes_cli/config_defaults.py:789` | `hermes_cli/config_defaults.py:811` |
| Review iteration cap | not run | `agent/background_review.py:150` (16) | `agent/background_review.py:173` (16) |
| Compression threshold | 0.50 (`hermes_cli/config.py:1466`) | 0.50 (`hermes_cli/config_defaults.py:565`) | 0.50 (`hermes_cli/config_defaults.py:589`) |

d604141 keeps its defaults in `hermes_cli/config.py`. It has no `config_defaults.py`. The ai-core commit has the same deferral, stall guard, and review behavior as HEAD. Only line numbers differ. The installed d604141 is older and gates the bridge by size.

## Peter's notes, checked against source

| Note says | Source says | Status |
|---|---|---|
| Line refs such as `agent/system_prompt.py:41-52` | 59004a623 matches them. HEAD matches for this ref too. | Notes match 59004a623. This doc uses HEAD lines. |
| Tool Search defers MCP tools; bridge always on | Same rule at 59004a623 and HEAD (`tools/tool_search.py:149-161`, bridge `:196-205`) | Confirmed for the ai-core commit |
| "Patch 04" pin list keeps tools visible | No per-tool pin key in 59004a623 or HEAD. Config keys at `hermes_cli/config_defaults.py:1991-2026`: `enabled`, `defer`, `threshold_pct`, `listing`, `listing_max_tokens`. | Not upstream. Lives in his local patch. |
| `agent.max_turns` 150 is a knob | Default `None` (`hermes_cli/config_defaults.py:78`), resolved at `hermes_cli/config.py:1833` | Knob exists. Default unlimited. |
| Default `max_iterations` 500 (area guide) | `run_agent.py:241` sets `sys.maxsize` | Area guide is stale |
| Background review about 30K tokens per event | Code comment at `agent/turn_finalizer.py:779` | Comment only. Not measured. |
| First call 15.7k tokens (20k chars system, 41k chars tools) | Fixed source blocks total about 5,273 chars (est., literal text, Telegram hint included; the six cited blocks alone are 4,410). Tool bytes not measured here. | Not run. Needs a live run. |
| Compression fails, 6 to 9 min per try | Code bounds: 120 s per call, max 3 attempts (`hermes_cli/config_defaults.py:765`, `:609`). His plan A2 (not in this repo) sets 300 s, which allows about 15 min (3 x 300 s). | Not run. The bound is in code. |
| Hermes overhead under 1% of turn time | Not measurable without a run | Not run |
| 86 skills, 8.6k chars in the index | This repo ships 58 bundled `SKILL.md` (counted) | Not run. Count depends on install. |

## Not verified

- Nothing was run. No hermes, no model, no network, no ssh.
- Token figures are chars/4 of source literals (est.). A real tokenizer will differ. Schema bytes are not measured.
- Round counts come from code paths. The model decides how many calls it makes.
- Gateway call site for a given message: not traced. `gateway/run_inbound.py` was not read.
- Not read in full: `turn_truncation.py` (length continuation), `micro_compaction.py`, the body of `conversation_compression.py`, the body of `turn_response_check.py`, `turn_recovery*.py`, `tools/approval_smart.py`, `agent/prompt_caching.py`.
- d604141: grep checks only, no full diff.
- Skill disable keys (`hermes_cli/config_defaults.py:1448` section) and the Telegram rich-messages default: not traced.
- ai-core is not reachable from here. Its build is inferred from Peter's line numbers and commit 59004a623 in this clone.
