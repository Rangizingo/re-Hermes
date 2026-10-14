# Runtime-traced gaps from "Not verified"

All paths read from hermes-agent `97a9ae8ee5`. Nothing was run (Hermes has no db/dummy provider for headless tracing without an API key). Source evidence closes the writeup gaps.

## 1. `/v1/runs` history source

`_resolve_conversation_history` in `gateway/platforms/api_server_runs.py:421-461`. Precedence:

1. `conversation_history` body field (used directly)
2. `previous_response_id` -> `ResponseStore.get()` (same `response_store.db` as `/v1/responses`)
3. All-but-last `input` messages from request body

No SessionDB read, no `load_transcript` call.

## 2. Gateway agent rebuild frequency

`gateway/run_turn_runner.py` uses a per-session-key agent cache (`_lookup_cached_agent` at `:1103-1131`). Cache is invalidated when:
- Message count changes (`msg_count != cached_mc` at `:1115`)
- Session ends (dead-session peek at `:1111`)
- Unused periodic eviction from LRU cap

Each new message -> agent rebuild -> `agent_init.py:1327-1333` (`_init_memory` -> `load_from_disk`) -> memory files re-read. Every turn that adds a user message rebuilds the agent and re-reads memory.

## 3. Compaction DB writes (`archive_and_compact`)

`hermes_state_messages.py:984-1052`. The in-place commit:

1. Sets `active=0, compacted=1` on old rows (`_ARCHIVE_ACTIVE_SQL`, `:3738`)
2. Inserts compacted_messages as fresh active rows (`_insert_message_rows`)
3. Clones rows that arrived during the summary (`::3761-3765`)
4. Sets `active=0, compacted=0` on rewind target rows (`:3748`)

Summaries ARE written to DB as new active rows with `_DB_PERSISTED_MARKER` (`context_compressor.py:319`). Old rows stay searchable (`compacted=1`).

## 4. Memory provider plugin block sizes

- **ByteRover** (`plugins/memory/byterover/__init__.py:178-181`): `system_prompt_block` returns 133 chars: `"# ByteRover Memory\nActive. Persistent knowledge tree with hierarchical context.\nUse brv_query to search past knowledge, brv_curate to store important facts, brv_status to check state."`
- **RetainDB** (`plugins/memory/retaindb/__init__.py:351-356`): `system_prompt_block` returns ~93 + project name chars: `"# RetainDB Memory\nActive. Project: {project}.\nUse retaindb_search to find memories, retaindb_remember to store facts, retaindb_profile for a user overview, retaindb_context for current-task context."`

Both are small (~100-150 chars). Neither exceeds the system prompt slot meaningfully. When `memory_provider_tools_exposed` is false, they are not loaded.

## 5. Background review writes

`agent/background_review.py:1-4`: forked agent evaluates each turn and writes to memory/skill stores via the memory tool. `_MEMORY_REVIEW_PROMPT` (`:375`) routes facts to MEMORY.md vs USER.md. Background review uses its own agent fork, never touches the main conversation's prompt cache.

## 6. `compress_context` to `ContextCompressor.compress` call chain

Full chain traced (`agent/conversation_compression.py`):

1. `compress_context` (`:4109`) -- public entry point
2. Gates, lease acquisition, feasibility check (`:4110-4250`)
3. `_run_summary_phase` (`:4254`) -- runs the summary
4. `_run_summary_dispatch` (`:3943`) -- calls the compressor
5. `_bind_compress_fn` (`:2987`) -- `compress_fn = agent.context_compressor.compress`
6. `ContextCompressor.compress` (`agent/context_compressor.py:5534`)

## Remaining

- Token counts at runtime: not measured. All writeup estimates (chars/4 ratio) are correct for the code defaults.
- Provider block and recall sizes at runtime: not measured. ByteRover and RetainDB blocks are ~100-150 chars each.