# core-loop.md review, lines 121-237 (source pin 97a9ae8ee5)

Verdicts over 110 claims: SUPPORTED 75 | PARTLY 9 | NOT SUPPORTED 2 | NO CITE 15 | OK-LABELED 9.

Doc: /Users/peterb/Documents/AI/re-Hermes/docs/core-loop.md. Not edited. The fixes below are described, not applied.

## Scope and method

- Source: /Users/peterb/Documents/AI/hermes-agent at 97a9ae8ee5, read with `git show` only. Nothing written to the source repo.
- Cites to 59004a623 (same clone) were read at that commit: config_defaults.py:565, 789, 1990; conversation_loop.py:1595; turn_final_response.py:170; tool_search.py:197-208; background_review.py:150; system_prompt.py:41-52. All match the doc, except system_prompt.py:41-52, which is row 25.
- Not checked: the d604141 cells in lines 203-209 and the d604141 sentences at line 211. They cite the installed tree (/Users/peterb/.hermes/hermes-agent), which is outside this pin.
- Cite rule: a cite counts when it sits in the same sentence, table row, or list item and supports the claim. OK-LABELED is used only where the doc itself labels the claim.
- Re-counts:
  - "58 bundled SKILL.md (counted)", line 226: `git ls-tree -r --name-only 97a9ae8ee5 | grep -c '^skills/.*SKILL\.md$'` = 58. Matches. Whole repo: 214 (skills 58, optional-skills 155, plugins 1).
  - "six default-deferred core tools", lines 174 and 205: `git show 97a9ae8ee5:hermes_cli/config_defaults.py | sed -n '2018,2025p' | grep -o '"[a-z_]*"' | wc -l` = 19, which includes the `"defer"` key. That is 18 names. The first six are in `_HERMES_CORE_TOOLS` (toolsets.py:11-39). The other 12 are GUI tools and are not core.
- Estimates: literal character counts were taken with Python `ast` over the pinned file (read from stdin, nothing written). Tokens use chars/4, as the doc says.
- Nothing run: no hermes, model, network, or ssh.

## Verdicts (non-SUPPORTED only)

| # | doc line | sentence (first 12 words) | cite | verdict | evidence | fix |
|---|---|---|---|---|---|---|
| 1 | 121 | The schemas are built in code, not literal dicts. | none | NO CITE | No cite in paragraph. Counter-evidence: 48 files under tools/ match `"parameters": {` (git grep -l, 97a9ae8ee5). | Cite the builder function, or drop "not literal dicts". |
| 2 | 121 | A static scan found none of the 35 (not run as a runtime | none | NO CITE | No cite. No scan command or file list for the count 35. | Show the scan command and output, or drop the count. |
| 3 | 121 | Peter's note says 41,218 chars for the 19-tool array on ai-core. | note, not in repo | OK-LABELED | Paragraph label "not measured (not run)" and "Not checked here." | None. |
| 4 | 125 | A flush happens before each tool round ... and at turn end. | turn_tool_round.py:127-130 | NO CITE | Turn-end flush exists at agent/turn_final_response.py:364: `agent._flush_messages_to_session_db(messages, conversation_history)`. | Add cite turn_final_response.py:364. |
| 5 | 126 | Every request sends the full transcript, canonicalized | turn_context.py:1236-1262 | PARTLY | turn_context.py:1258-1259: "Only the rows BEFORE this turn's user message are the replayed prefix"; :1263: "so nothing is canonicalized". | Say canonicalization covers the replayed prefix only. Rows from the current turn go out unchanged. |
| 6 | 126 | Only compression shortens it. | agent/AGENTS.md (section only) | PARTLY | AGENTS.md:59: "the ONLY context mutation is compression" holds on the main loop. Other truncation: agent/codex_runtime_history_seed.py:63-64 `body[-MAX_HISTORY_SEED_CHARS:]`; gateway/platforms/api_server.py:406 `return conversation_history[-limit:]`. | Scope to the default path, name the two exceptions, cite AGENTS.md:59. |
| 7 | 127 | Trigger: compression.threshold is 0.50 | hermes_cli/config_defaults.py:589 | NOT SUPPORTED | Config value is 0.50 (config_defaults.py:589). The trigger is not 0.50 below 512K: agent/context_compressor.py:2757-2758 `return max(threshold_percent, _SMALL_CTX_THRESHOLD_PERCENT)`; constants at :1150-1151 (512_000 and 0.75). | Say the trigger is 0.50 at 512K and up, and 0.75 below 512K. |
| 8 | 130 | That summary is one extra model call on the same engine slot. | none | NO CITE | "engine slot" has no code cite. The one-summary-call part is at agent/AGENTS.md:85-86, in the same bullet. | Cite the slot code, or label it as Peter's engine. |
| 9 | 133 | Caps before history: terminal output 50,000 chars and 2,000 lines | hermes_cli/config_defaults.py:555 | PARTLY | Line 553 comment: "max_lines: max `limit` one read_file call may request". Line 555 `"max_lines": 2000` is the read_file cap, not a terminal cap. | Drop "2,000 lines" from the terminal clause. State that read_file max_lines is 2,000. |
| 10 | 139 | This needs the tool to be eager. Default config does not make | none | NO CITE | Default `"enabled": "auto"` at config_defaults.py:1999. MCP tools are deferrable at tools/tool_search.py:149-161. Neither is cited here. | Cite config_defaults.py:1999 and tool_search.py:149-161. |
| 11 | 140 | Default, MCP deferred: 3 to 4 calls. | none | NO CITE | The sequence matches the bridge in tools/tool_search.py:273 (bridge_tool_schemas). The count is derived and not cited. | Cite tool_search.py:273 and conversation_loop.py:1648. |
| 12 | 141 | Each load adds one call. The model decides. | none | NO CITE | Mechanism at agent/turn_tool_round.py:230: `return _verdict("continue")`. Not cited here. | Cite turn_tool_round.py:230, or drop. |
| 13 | 144 | Empty response: +1 call per retry, budget 3, jittered wait | empty_response_guard.py:54; turn_empty_response.py:46-70 | PARTLY | empty_response_guard.py:230: "(3, or 1 when a single attempt is". Line 236 returns REDUCED_EMPTY_RETRY_BUDGET (1) when estimated cost reaches the threshold (default $0.25, empty_response_guard.py:56). | Say the budget is 3 by default and 1 when one attempt's estimated cost passes the threshold. |
| 14 | 155 | Title generation: first 1 to 3 turns of an untitled session | turn_context.py:196-232; title_generator.py:736; config_defaults.py:774-776 | PARTLY | title_generator.py:756: "Untitled sessions always get another shot". Line 757: "a placeholder gets turns 2-3". `_session_is_untitled` (title_generator.py:703-710) is true only when no title exists. | Say the 1-3 turn window applies to placeholder titles. Untitled sessions are retried each turn. |
| 15 | 166 | Total for this request: 2 at the floor. 3 to 4 with the default | none | NO CITE | Derived from lines 139-141. No cite in this paragraph. | Cite the floor and deferral lines here. |
| 16 | 166 | Peter's note says 5 calls (not run here). | note, not in repo | OK-LABELED | Label "(not run here)". | None. |
| 17 | 173 | Cost: 1 to 2 extra calls per deferred tool use. Listing up to | none in item | NO CITE | Value 4000 is at config_defaults.py:2014 (cited only at line 119). Bridge sequence at tool_search.py:273. | Cite config_defaults.py:2014 and tool_search.py:273 in this item. |
| 18 | 175 | Risk: a bigger tool array on every request. Cached after the first | none | NO CITE | No cite for caching. agent/prompt_caching.py was not read (doc line 234). | Cite the cache code, or drop "cached after the first call". |
| 19 | 175 | A small model may pick tools less well with more visible tools | none | OK-LABELED | Label "(not measured)". | None. |
| 20 | 183 | Cost: +1 call per skill loaded. | none | NO CITE | Mechanism at agent/turn_tool_round.py:230. Not cited here. | Cite turn_tool_round.py:230, or drop. |
| 21 | 183 | Header ~375 to 475 tokens (est.), plus one line per visible skill. | prompt_builder.py:1433-1495 | PARTLY | Literal text of the return at prompt_builder.py:1475-1495 is 1,167 chars (ast). Adding `_basic_tools` (8-24 chars) gives about 1,190 chars, or about 298 tokens by chars/4. The doc says 375-475. Line 1477: "Before replying, scan the skills below." | Change to about 300 tokens (est.), or name what else the figure counts. |
| 22 | 188 | It runs after delivery, but on the same engine slot as the next turn. | none | OK-LABELED | The same bullet labels the premise: "Peter's engine is one slot per his notes (not run here)." | None. |
| 23 | 193 | Cheap while the prefix caches. A cache miss re-prefills it. | none | NO CITE | Re-prefill is at agent/system_prompt.py:781-782: "a changed one re-prefills from here". Not cited here. Cache code not read (line 234). | Cite system_prompt.py:781-782 and the cache code, or drop "cheap". |
| 24 | 197 | Title generation is a sixth side call and cheap to cut. | none | PARTLY | Section 6 lists two side calls (lines 151 and 155). Title generation is the sixth item in the cost list (line 197), not the sixth side call. | Write "second side call" or "sixth cost item". |
| 25 | 217 | Line refs such as agent/system_prompt.py:41-52: 59004a623 matches them. HEAD does not. | agent/system_prompt.py:41-52 | NOT SUPPORTED | HEAD :41 `def _model_gate(setting: Any, model: Optional[str], default_models) -> bool:`. The 59004a623 lines 41-52 are the same code. The note itself is not in the repo. | Drop "HEAD does not" for this ref, or name the refs that differ. |
| 26 | 218 | Tool Search defers MCP tools; bridge always on | none in row | NO CITE | Behavior holds at HEAD: tools/tool_search.py:149-161 (MCP deferrable) and :196-205 (bridge on when deferrable tokens > 0). | Add the two HEAD cites to the row. |
| 27 | 219 | No per-tool pin in 59004a623 or HEAD. Only enabled and defer exist. | none in row | NO CITE | `git grep -i pin` on tools/tool_search.py matches only "typing" (line 17). Config keys at config_defaults.py:1991-2026 also include threshold_pct, listing, listing_max_tokens. | Cite config_defaults.py:1991-2026. Say "no per-tool pin key". |
| 28 | 222 | Background review about 30K tokens per event | agent/turn_finalizer.py:779 | OK-LABELED | Row label "Comment only. Not measured." The comment says "~30K tokens / event with no human-in-the-loop benefit". | None. |
| 29 | 223 | First call 15.7k tokens (20k chars system, 41k chars tools) | none | OK-LABELED | Row label "Not run. Needs a live run." | None. |
| 30 | 223 | Fixed source blocks total about 5,100 to 5,500 chars (est.). | prompt_builder.py:382, 419, 603, 192, 258, 394 | PARTLY | Literal chars (ast) for the six cited blocks: 769 + 618 + 572 + 1,543 + 421 + 487 = 4,410. The range needs the Telegram hint (prompt_builder.py:852-861, 863 chars): 5,273. | Name the Telegram hint in the row, or change the range to about 4,400 chars (est.). |
| 31 | 224 | Compression fails, 6 to 9 min per try | none | OK-LABELED | Row label "Not run." | None. |
| 32 | 224 | His plan A2 sets 300 s, which allows about 15 min. | none | NO CITE | Plan A2 is not in the repo. A grep of docs/ finds only core-loop.md itself. The arithmetic is right: 3 x 300 s = 900 s = 15 min. | Cite the plan file, or drop the plan claim. |
| 33 | 225 | Hermes overhead under 1% of turn time | none | OK-LABELED | Row label "Not run". | None. |
| 34 | 226 | 86 skills, 8.6k chars in the index | none (58 count is cited) | OK-LABELED | Row label "Not run. Count depends on install." The 58 count was re-run and matches (see Scope). | None. |
| 35 | 231 | Token figures are chars/4 of source literals (est.). | none | PARTLY | The line 183 header figure does not follow this method: 1,167 literal chars / 4 = 292 tokens. Line 223's 1,275-1,375 does follow it (5,100/4 and 5,500/4). | Recompute the line 183 figure with the same method. |

## Notes (not verdicts)

- Defer list: the doc's "six" is the core subset. The full list has 18 names (config_defaults.py:2018-2025). The Fix at line 174 is accurate as written.
- Outside the doc's sentences: config_defaults.py:177-180 says `stall_guards: false` also disables the identical-call loop breaker. No code in agent/, tools/, or hermes_cli/ reads the flag for that breaker. tool_guardrails.py:323 is a comment only. Check this before flipping the flag.
- Title toggle: `model_upgrade_enabled` is read at title_generator.py:174 and gated at :780. The line 197 claim holds.
- Cite format: AGENTS.md cites at lines 126 and 130 name sections, not file:line (CLAUDE.md rule).
- Lines 233-234: every file named as "not read in full" exists at the pin.
