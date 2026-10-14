# Memory, skills, and API server history

Read only. Pinned to hermes-agent `97a9ae8ee5`. Paths are relative to that repo root. Every citation was read with `git show`, `git grep`, or a line-range print at the pin. Nothing was run: no hermes, no model, no network, no ssh.

Labels: **counted** = grep count at the pin. **est.** = arithmetic or chars/4. **config value** and **constant** = read from source, not run. **not run** = not executed.

This closes two items in [surfaces.md](surfaces.md) "Not verified": the API server history source and the skill-load path. Turn loop context is in [core-loop.md](core-loop.md).

## Short version

- Memory is two files, read once per agent at init. Their text sits in the cached system prompt, which goes with every request. Writes do not change that text until the next load.
- A loaded skill body is a normal tool message. It stays in history and replays on later turns. Compaction can shrink it to a one-line summary. A second load of the same unchanged file returns a stub.
- The API server reads SessionDB through the same method the gateway uses, but without `repair_alternation`. It never calls `load_transcript`. Responses chaining reads `response_store.db`, not SessionDB.

## 1. Memory

Summary. Two files, `memories/MEMORY.md` and `memories/USER.md`. Both load at agent init into a frozen snapshot. The snapshot goes into the volatile system-prompt tier, so it reaches every request. Writes touch the live list and disk, not the snapshot. Provider blocks and per-turn recall are separate and conditional.

Files and loading
- Folder: `get_hermes_home() / "memories"` (`tools/memory_tool.py:38-40`). File names come from `_path_for` (`tools/memory_tool_store.py:264-266`).
- Other files in that folder: `<name>.lock` (`tools/memory_tool_store.py:227`), `<name>.bak.<ts>` after external drift (`tools/memory_tool_store.py:592`, `:601`), and a `.mem_` temp prefix for atomic writes (`tools/memory_tool_store.py:587`). None of them are read into the prompt.
- Profile clone list names both files (`hermes_cli/profiles.py:40`). `SOUL.md` is identity, not memory (`agent/system_prompt.py:545`).
- Entry delimiter is `\n§\n` (`tools/memory_tool_store.py:25`). Block titles are at `tools/memory_tool_store.py:22-23`.
- Load call: `_init_memory` builds the store and calls `load_from_disk` (`agent/agent_init.py:1296`, `:1327-1333`). It runs when `not skip_memory or _memory_toolset_requested` (`agent/agent_init.py:1315`).
- `MemoryStore.load_from_disk` reads both files and dedups (`tools/memory_tool_store.py:187-209`). A file over its cap warns and keeps its entries (`:211-217`). The snapshot is rendered at `:218`.
- Threat-scanned entries become `[BLOCKED: ...]` in the snapshot only. The live list keeps the raw text (`tools/memory_tool_store.py:189-191`, `:194-201`).
- Frozen by design: `_system_prompt_snapshot` is set at load time (`tools/memory_tool_store.py:141-144`). `format_for_system_prompt` returns that snapshot (`:524-527`).
- Block text: header, usage percent, two rule lines (`tools/memory_tool_store.py:544-550`). An empty file renders nothing (`:546-547`).
- Non-agent readers: `load_on_disk_store()` builds a fresh store for the gateway, Desktop, and `/memory` (`tools/memory_tool.py:47-61`).

What reaches each request
- `_memory_parts` adds the MEMORY and USER blocks when their flag is on (`agent/system_prompt.py:516-525`). The volatile tier includes them (`agent/system_prompt.py:783`).
- Each API call prepends the cached system prompt (`agent/turn_context.py:1336-1343`).
- Both stores default to on (`hermes_cli/config_defaults.py:1312-1313`, config value).
- Caps: 2,200 chars for MEMORY and 1,375 for USER (`hermes_cli/config_defaults.py:1319-1320`, config values). Code defaults match (`tools/memory_tool_store.py:152`).
- Caps are checked on add (`tools/memory_tool_store.py:345-346`). A second check uses the final state (`tools/memory_tool_store.py:504`).
- Memory text per request, max: 2,200 + 1,375 = 3,575 chars of entries (est., arithmetic on config values).
- Block headers add about 260 chars (est.). The count comes from the rule lines and titles at `tools/memory_tool_store.py:548-550`: 129 chars for MEMORY and 131 for USER, with a 6-char usage tag assumed.
- Total about 3,835 chars, or about 960 tokens at chars/4 (est.).
- The config comment says about 800 + 500 tokens at 2.75 chars/token (`hermes_cli/config_defaults.py:1319-1320`). That is about 1,300 tokens (est.) for the same caps.
- Real file sizes on this machine: not run.
- Mid-session writes leave the snapshot unchanged (`tools/memory_tool_store.py:524-526`). A new agent reads the files again. How often the gateway builds one: not traced.
- Guidance text (how to use the memory tool) is added only when the memory tool is visible (`agent/system_prompt.py:281-286`, `:293`).

Provider blocks and recall
- External provider `system_prompt_block()` text joins the memory parts (`agent/memory_manager.py:501-505`, `agent/system_prompt.py:529-540`). It appears only when `memory_provider_tools_exposed` is true (`agent/memory_manager.py:105-114`).
- Default provider is empty, so none is loaded (`hermes_cli/config_defaults.py:1325`, config value).
- Per-turn recall: `prefetch_all` (`agent/memory_manager.py:511-520`) runs in `_memory_turn_start_and_prefetch` (`agent/turn_context.py:902-922`). Trivial prompts skip it (`agent/turn_context.py:921`).
- Recalled text goes only into the API copy of the user message (`agent/turn_context.py:933-937`, `:1337-1339`). It never enters the system prompt.
- Provider block and recall sizes: not measured.

Always loaded or conditional
- Always, with defaults: the MEMORY block and the USER block, each only when its file has entries (`agent/system_prompt.py:521-525`, `tools/memory_tool_store.py:546-547`).
- Conditional: the guidance text (tool visible), the provider block (provider set and tool exposed), per-turn recall (provider set and non-trivial prompt), and the whole store when `skip_memory` is set and the memory toolset is not requested (`agent/agent_init.py:1315`).

## 2. Skills

Summary. The index is a `## Skills` block in the volatile tier of the system prompt. Its text tells the model to load matching skills with `skill_view`. `skill_view` returns the body as a JSON tool result. That result is stored and replayed. Compaction can shrink it to a one-line summary. A repeat load of an unchanged file returns a stub.

Index
- `skills_prompt = _skills_prompt(agent)` (`agent/system_prompt.py:753`). It goes first in the volatile tier (`agent/system_prompt.py:783`).
- `_skills_prompt` returns empty unless `skills_list`, `skill_view`, or `skill_manage` is a valid tool (`agent/system_prompt.py:301-314`, gate at `:304`).
- Builder: `build_skills_system_prompt` (`agent/prompt_builder.py:1361-1364`), then `_build_skills_system_prompt_inner` (`agent/prompt_builder.py:1503`).
- The cache key includes a signature for every skill file, so edits from other processes show up in the next build (`agent/prompt_builder.py:1513-1525`, comment `:1513-1515`).
- Lookup order: in-process LRU (`agent/prompt_builder.py:1530-1534`), then a disk snapshot as the fast path (`:1526`, `:1544-1549`). Without a snapshot it scans (`:1550-1554`).
- Disabled names and unmet conditions hide rows (`agent/prompt_builder.py:1536-1539`). `skill_view` still resolves hidden skills (`:1555-1557`).
- Line format: `    - {name}: {desc}` (`agent/prompt_builder.py:1467`). No skills, no block (`:1439-1440`).
- The index in the prompt stays fixed for the session. Later changes arrive as a one-shot note on the user message (`agent/skills_index_delta.py:1-14`, `:100-118`, call at `agent/turn_context.py:1179`).

Instruction text ("MUST load")
- "you MUST load it with skill_view(name) and follow its instructions" (`agent/prompt_builder.py:1477-1478`). Also: update a loaded skill that had missing steps (`:1487-1488`), and proceed without one only when none apply (`:1493`).
- Single-query sessions get a softer line: load a skill "only when it carries domain knowledge you lack" (`agent/prompt_builder.py:1468-1471`; text at `agent/oneshot_footprint.py:42-44`).
- `SKILLS_GUIDANCE` is added when `skill_manage` is visible (`agent/system_prompt.py:295`). It tells the model to reload any `[SKILL_PRUNED]` placeholder before relying on it (`agent/prompt_builder.py:258-264`).
- Pinned skills (`skills.auto_load`) go in the stable prefix, not history (`agent/system_prompt.py:317-341`, `:760`). Agents that skip context files get none (`agent/system_prompt.py:325-326`).

What a load does
- Schema at `tools/skills_tool.py:673`. Registered as `skill_view` with handler `_skill_view_with_bump` (`tools/skills_tool.py:724-726`, `:697`).
- Handler order: dedup check (`tools/skills_tool.py:708-709`), then `skill_view` (`:710`), then record (`:714`), then usage counters (`:716-720`). Background-review forks skip dedup (`:703-707`).
- Return value is JSON (`tools/skills_tool.py:652`). The body is the `content` field (`:637`). `_source_path` is kept for the dedup fingerprint (`:642-643`).
- The tool message has role `tool` (`agent/tool_dispatch_helpers.py:448`). Tool progress is flushed to the session DB before it reaches the UI (`agent/tool_executor.py:196-198`; flush at `:207`).
- Replay: later turns read only active rows (`hermes_state_messages.py:1390`, `:1529-1531`). The gateway loads the stored transcript before each agent call (`gateway/run_turn.py:2096`).
- No compaction: the body stays in history and goes out on each later request.
- Repeat load of the same path, same mtime and size, same task: a stub with `"content_returned": false` and the text "Skill content unchanged since it was loaded earlier" (`tools/skills_tool_dedup.py:52-81`, text `:17-21`). Cache is per task (`tools/skills_tool_dedup.py:13`), capped at 200 entries (constant, `:15`, `:48-49`).
- Compaction clears the dedup cache. `_reset_read_dedup_caches` does that (`agent/conversation_compression.py:3414-3429`). It runs from `_finish_compaction_boundary` (`:3432`, call at `:3522`) and four other call sites (counted by grep). After that, a repeat load returns the full body again (`tools/skills_tool_dedup.py:19-21`).

Compaction path
- After each tool round, `compress_after_tool_results` runs (`agent/turn_tool_round.py:202`, `agent/turn_preflight.py:244`). It compresses when `should_compress` says so (`agent/turn_preflight.py:305`), through `_compress_context` (`agent/turn_preflight.py:315`; forwarder at `agent/compression_facade.py:221-227`).
- Before a request, the same threshold test runs in `run_preflight_compression` (`agent/turn_preflight.py:102`, `:147`).
- `compress_context` (`agent/conversation_compression.py:4109`) reaches the engine's `compress()` through a helper that binds its kwargs (docstring at `agent/conversation_compression.py:2986`). The call that links them was not traced.
- Engine: `ContextCompressor` (`agent/context_compressor.py:2149`), method `compress` at `agent/context_compressor.py:5534`. Phase 1 prunes old tool results with no model call (`agent/context_compressor.py:5573-5576`).
- Head and tail are built from the pruned copy. The unpruned copy serves no-op and abort returns, and finalize (`agent/context_compressor.py:5570-5572`).
- Phase 1 calls `_prune_old_tool_results` (`agent/context_compressor.py:3277-3280`). Default minimum is 200 chars (constant `:839`, used at `:3279`).
- In `_demote_tool_result_at` (`agent/context_compressor.py:3176-3204`): skip if at or under the minimum (`:3195`). Skip a protected skill body (`:3199-3202`). Otherwise replace it with a summary (`:3203`).
- skill_view summary (`agent/context_compressor.py:1721-1725`): `[skill_view] name=X (N chars)`. The reload marker `[SKILL_PRUNED: ... reload with skill_view(name='X')]` is added only when N > 5000 (`:1724`, constant `:851`, marker at `:856-861`).
- Dispatch: `_TOOL_RESULT_SUMMARIZERS["skill_view"]` (`agent/context_compressor.py:1945`, `:1959`), reached through `_summarize_tool_result_unguarded` (`:2010-2022`). Only a refusal check (`:2014`, not read) runs before the dispatch.
- Discrepancy: the comment at `agent/context_compressor.py:850` says "Small skill_view results stay verbatim". The code does not do that for bodies from 201 to 5,000 chars. They get the one-line summary with no marker. By the code's own rationale (`:843-844`), that is the ghost-skill case. Read from code, not run.
- Protection (`agent/context_compressor.py:1103-1119`): a skill body stays whole when its load is in the last 10 messages (`:1083`, `:1110`, constant), in the protected tail (`:1111`, `:1117`), or its name appears in a tail user message (`:1118`). The set is built at `:3299` and used at `:3304`. Pass-4 pressure demotion ignores it (`:1106`, `:3181`).
- The blocked and proactive prune (`agent/context_compressor.py:3387-3392`, call at `:3425-3426`) calls `_prune_old_tool_results`, which builds the same protection internally (`:3299`). It is off by default: `proactive_prune_tokens` is 0 (`hermes_cli/config_defaults.py:610-615`, `agent/agent_init.py:1556-1557`).
- Archive: compaction soft-archives old active rows. The archive SQL sets `active = 0, compacted = 1` (`hermes_state_messages.py:70`). The in-place case is at `agent/conversation_compression.py:3737`. Replay excludes archived rows (`hermes_state_messages.py:1390`, `:1523-1528`), so an archived skill body is not replayed.

## 3. API server history

Summary. The API server has three history sources. Session routes and the `X-Hermes-Session-Id` path read SessionDB. Stateless calls use the request body. Responses chaining reads `response_store.db`. The SessionDB calls use default flags. The server never calls `load_transcript`.

Call sites
- Session routes: `_handle_session_chat` (`gateway/platforms/api_server.py:3570`) and `_handle_session_chat_stream` (`gateway/platforms/api_server.py:3615`). Both load history through `_conversation_history_for_session` (`gateway/platforms/api_server.py:3047-3054`, call at `:3052`). Calls at `:3588-3589`, `:3599-3601`, `:3674-3676`.
- That helper uses the per-profile SessionDB from `_ensure_session_db_async` (`gateway/platforms/api_server.py:3048`).
- Chat completions, no session header: `history = conversation_messages[:-1]`, built from the request body (`gateway/platforms/api_server_openai_routes.py:668-675`).
- Chat completions with `X-Hermes-Session-Id` (needs an API key, `gateway/platforms/api_server_openai_routes.py:688-694`): the live tip is resolved (`:710-711`), then history comes from SessionDB (`:712`). On error it becomes `[]` (`:713-715`). This replaces the body history.
- Responses: an explicit `conversation_history` in the body wins (`gateway/platforms/api_server_openai_routes.py:1085`, `:1100-1102`).
- Responses chaining: `previous_response_id` (`gateway/platforms/api_server_openai_routes.py:1058`), or a `conversation` name resolved to its last response (`:1063-1065`). Both together is a 400 (`:1061-1062`).
- Chain lookup: `ResponseStore.get` (`gateway/platforms/api_server_openai_routes.py:1104-1105`), then `stored["conversation_history"]` (`:1108`). The per-profile store comes from `_current_response_store` (`gateway/platforms/api_server.py:1848`).
- Input items except the last become history (`gateway/platforms/api_server_openai_routes.py:1113`). `truncation: "auto"` trims with `_auto_truncate_response_history` (`gateway/platforms/api_server_openai_routes.py:1117-1118`; def at `gateway/platforms/api_server.py:396-418`).
- `ResponseStore`: class at `gateway/platforms/api_server.py:695`, file `response_store.db` (`:705`), methods `get` (`:735`), `put` (`:753`), `get_conversation` (`:777`).
- Stored snapshot: `_build_response_conversation_history` (`gateway/platforms/api_server_openai_routes.py:1240-1262`, call at `:1187`). Tool-output cap defaults to 0, meaning stored verbatim (`gateway/platforms/api_server.py:1185-1186`, `gateway/platforms/api_server_openai_routes.py:1249-1250`).

Comparison with the gateway
- Gateway: `load_transcript` (`gateway/session_transcript.py:631`) calls `get_messages_as_conversation` with `repair_alternation=True` (`gateway/session_transcript.py:644-645`).
- API server: the same SessionDB method, no flag (`gateway/platforms/api_server.py:3052`, `gateway/platforms/api_server_openai_routes.py:712`). The default is `repair_alternation=False` (`hermes_state_messages.py:1521`).
- Docstring: repair is for live replay callers, so a stored `user;user` pair does not re-trigger repair on every request (`hermes_state_messages.py:1524-1528`).
- CLI resume passes `repair_alternation=True` too (`hermes_cli/cli_agent_setup_mixin.py:607`).
- Counted (`git grep` at the pin, `*.py`): 0 `load_transcript` calls in the two API server files. 2 `get_messages_as_conversation` calls there, at `gateway/platforms/api_server.py:3052` and `gateway/platforms/api_server_openai_routes.py:712`.
- Effect of the missing flag on replay: not run. The docstring implies the API server's history is not repaired.

## Not verified

- `/v1/runs` history source (`gateway/platforms/api_server.py:4398`). Not read.
- `_resolve_live_session_id` body (imported at `gateway/platforms/api_server_openai_routes.py:710`). Not read.
- Effect of `repair_alternation=False` on API replay. Not run.
- Call that links `compress_context` (`agent/conversation_compression.py:4109`) to `ContextCompressor.compress` (`agent/context_compressor.py:5534`). Only the binder docstring at `agent/conversation_compression.py:2986` was read.
- Whether a Phase 1 summary or marker is written to the DB. The code comment says history stays lossless (`agent/context_compressor.py:5570-5571`). Compaction writes its compressed set as the new active rows (`agent/conversation_compression.py:2440` comment). Not traced further.
- How often the gateway rebuilds an agent, so how often MEMORY.md is re-read (`gateway/run_turn_runner.py:1139-1157`, not read).
- Memory provider plugins (`plugins/memory/`) and their block sizes. Not measured.
- Background review writes to MEMORY and USER (`agent/background_review.py`), and the `write_approval` gate (`hermes_cli/config_defaults.py:1318`). Not traced.
- `/skill` slash path (`agent/memory_manager.py:507-509` mentions it), `skill_manage`, and `build_auto_load_prompt` (`agent/skill_commands.py`). Not traced.
- The 200-to-5,000-char summary gap (`agent/context_compressor.py:850` against `:839` and `:3195`). Read from code. Not reproduced.
- Token counts are est. only (chars/4 or arithmetic). Nothing was measured.
