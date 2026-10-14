# core-loop.md lines 1-120 checked against hermes-agent at 97a9ae8ee5

**Verdicts over 95 units: SUPPORTED 73, PARTLY 16, NOT SUPPORTED 1, NO CITE 4, OK-LABELED 1.** The table lists the 22 units that are not SUPPORTED.

## Method

- Doc: `docs/core-loop.md` lines 1-120, read only. Source: `git show 97a9ae8ee5:<path>` in `hermes-agent`, read only. Nothing run, no network.
- Unit = one sentence, one list step, or one table row. A row with two checkable claims counts as two units.
- Character sizes come from a read-only `ast.parse` of the pinned file (`python3 -I`, `len()` of parsed strings). `grep -c` cannot count characters, so this differs from the job's stated method. "Rendered" keeps the f-string text the prompt sends. "Literal" drops it. Token figures are est. (chars / 4).
- Not checked: provenance and process lines (line 3, "per his notes" on line 8, "Grep checks only" on line 9). They make no code claim.
- Fixes are described, not applied. `core-loop.md` was not edited.

## Re-counts (counted)

- `git rev-list --count`: 97a9ae8ee5 = 51,424; 59004a623 = 43,682; difference 7,742. Matches line 8.
- `git describe --tags 97a9ae8ee5`: `rc.11-v0.21.7-55-g97a9ae8ee5`. The tag is rc.11-v0.21.7. "55" is the commit distance, not part of the tag.
- `git ls-tree -r --name-only 97a9ae8ee5 | grep -c 'SKILL\.md$'`: 214 total. skills/ = 58 (matches line 81). optional-skills/ = 155. plugins/ = 1.
- toolsets.py lines 11-41, `grep -o '"[a-z_]*"' | wc -l`: 55 names. `kanban_` names: 14. Both match.
- config_defaults.py lines 2018-2025, `grep -o '"[a-z_]*"' | wc -l`: 19 tokens, which is 18 names plus the `defer` config entry itself.
- `git grep -n _budget_grace_call` (non-test): the only setter of `= True` is agent/turn_truncation.py:635.
- Rendered vs literal sizes: STEER_CHANNEL_NOTE 771 rendered, 572 literal. SKILLS_GUIDANCE 441 rendered, 421 literal. PLATFORM_HINTS telegram 617 rendered, 535 literal. The doc's figures for these three are literal-only.
- Sizes that match as written: rows 2 (663), 3 (600 and 466), 4 (769), 5 (618), 8 (186), 10 (487), 11 (824 and 860), 20 (863). Memory limits 2,200 and 1,375 match at config_defaults.py:1319-1320.
- 59004a623 claims hold: system_prompt.py:41-52 holds `_model_gate`; prompt_builder.py:361 at 59004a623 holds TOOL_USE_ENFORCEMENT_MODELS. d604141 is an ancestor of 59004a623.

## Table

| # | doc line | sentence (first 12 words) | cite | verdict | evidence (file:line plus quote) | fix (one line) |
|---|---|---|---|---|---|---|
| 1 | 7 | HEAD: /Users/peterb/Documents/AI/hermes-agent at 97a9ae8ee5 (2026-10-11, tag rc.11-v0.21.7-55) | git tag, no file | PARTLY | `git describe --tags 97a9ae8ee5` prints `rc.11-v0.21.7-55-g97a9ae8ee5`. Date 2026-10-11 matches. | Write "tag rc.11-v0.21.7, 55 commits past it." |
| 2 | 14 | A text answer ends the turn. | none in sentence (bullet cites :402) | PARTLY | Normal end at agent/turn_final_response.py:376 `return _verdict("break")`. Guards continue instead: :249 `return _verdict("continue")` (stall, degenerate, ack; cap 2) and :318 (dropped tool call; cap 3). | "A text answer ends the turn unless a continuation guard fires (turn_final_response.py:201-318)." |
| 3 | 15 | Default config defers six core tools and every MCP tool | none | NO CITE | Six core names are in config_defaults.py:2019-2020 (see #20). MCP tools defer at tools/tool_search.py:149-161. | Add both cites inline. |
| 4 | 15 | A simple MCP lookup takes 3 to 4 model calls. | none | NO CITE | No source line sets this. It depends on the model's choices. Not labeled est. or not run. | Label "est., not run" and name the call path it assumes (tool_search, tool_describe, tool_call, text). |
| 5 | 15 | The floor is 2. | none | NO CITE | Same as #4. | Label "est., not run" and name the 2-call path (tool_call, then text). |
| 6 | 16 | Three things add calls with no tool result: the stall guard, skill loads | none | NO CITE | Stall guard: agent/turn_final_response.py:172-249 (re-prompt with no tool result). Background review cancel: agent/turn_facade.py:41. Skill-load path: not traced. | Cite each item, or drop the skill-load item until it is traced. |
| 7 | 17 | Fixed prompt text is about 1,275 to 1,375 tokens (est.) per request. | none in sentence (line 96) | PARTLY | Inputs are off (rows 6, 7, 9, 19; see #10). Rendered sum is 5,267 to 5,431 chars counted, so 1,317 to 1,358 tokens (est.). | Use about 1,320 to 1,360 tokens (est.) after fixing the rows. |
| 8 | 57 | Iteration budget spent: turn_iteration_prep.py:412, then one grace call | agent/turn_iteration_prep.py:412; agent/conversation_loop.py:1649 | NOT SUPPORTED | turn_iteration_prep.py:412 `elif not agent.iteration_budget.consume():` then :416 `return _verdict("break")`. No grace is set on this path. The only setter is agent/turn_truncation.py:635 `agent._budget_grace_call = True`, reached after a codex reasoning-only fallback (:634 checks budget). | "Spent budget ends the loop at turn_iteration_prep.py:416. A grace call follows only a codex reasoning-only fallback (turn_truncation.py:635)." |
| 9 | 93 | Only for non-trivial prompts. | agent/turn_context.py:108-133, :793-849, :893-931 | PARTLY | agent/turn_context.py:921 `if not is_trivial_prompt(_query):` gates memory prefetch only. agent/turn_context.py:1172 calls `_collect_pre_llm_call_context` with no triviality check. | "Say memory recall skips trivial prompts (turn_context.py:921). pre_llm_call text does not." |
| 10 | 96 | Total 5,100 to 5,500 chars (counted from literals; row 7 is a range) | table rows 3-19 | PARTLY | Rendered sum (counted): 600, 769, 618, 771, 964 to 1,128, 441, 487, 617, which is 5,267 to 5,431. Literal-only sum (the doc's method): 4,966 to 5,130. The doc's range matches neither. | "Replace with the counted sum and name the method: rendered 5,267 to 5,431, or literal-only 4,966 to 5,130." |
| 11 | 74 | Steer channel note: 572 chars, ~143 tokens (est.) | prompt_builder.py:603; system_prompt.py:566-567 | PARTLY | prompt_builder.py:603 `STEER_CHANNEL_NOTE` renders 771 (counted). The doc's 572 is literal-only: 771 minus STEER_MARKER_OPEN (172) and STEER_MARKER_CLOSE (27), both substituted at prompt_builder.py:615 `f"{STEER_MARKER_OPEN}`. | "Label 572 as literal text only, or change to 771 chars (about 193 tokens, est.)." |
| 12 | 75 | Memory guidance: 1,100 to 1,500 chars, ~275 to 375 tokens (est.) | prompt_builder.py:192-230; system_prompt.py:281-286 | PARTLY | Counted pieces at prompt_builder.py:199-229: memory frame 149, profile frame 241, skill routing on 261, off 189, tail 626. Output range 964 to 1,128. Default 1,036. system_prompt.py:282 calls it once. | "Change to 964 to 1,128 chars (1,036 at defaults), about 241 to 282 tokens (est.)." |
| 13 | 77 | Skills guidance: 421 chars, ~105 tokens (est.) | prompt_builder.py:258; system_prompt.py:295 | PARTLY | Rendered 441 (counted). The gap is `f"{SKILL_SAFETY_HEADING}\n"` at prompt_builder.py:260 (20 chars). | "Label 421 as literal text only, or change to 441 chars (about 110 tokens, est.)." |
| 14 | 79 | 824 and 860 chars when they match (counted). | prompt_builder.py:347, :361, :375, :544 | PARTLY | Counted: TOOL_USE_ENFORCEMENT_GUIDANCE 824 (:347), GOOGLE_MODEL_OPERATIONAL_GUIDANCE 860 (:544). OPENAI_MODEL_EXECUTION_GUIDANCE 3,952 (:441) is not sized. "Only when" holds for auto only: system_prompt.py:572 `_model_gate(agent._tool_use_enforcement,` also takes true, false, or a list. | "Name each block with its count (824, 860, 3,952), and say "only when auto (default) matches"." |
| 15 | 79 | Off for Peter's model name per his note (not re-checked). | none | OK-LABELED | The model name is in Peter's config, not the pinned source. The doc labels it "not re-checked." | None needed. Optional: check the model name against prompt_builder.py:361. |
| 16 | 80 | Only cli, tui, acp, desktop with a code workspace. | agent/coding_context.py:30 | PARTLY | coding_context.py:30 `INTERACTIVE_CODING_PLATFORMS = {"cli", "tui", "acp", "desktop", ""}`. The empty platform string is also in the set. | Add the empty platform string to the list, or say "plus empty platform." |
| 17 | 81 | Header 1,500 to 1,900 chars, ~375 to 475 tokens (est.) | prompt_builder.py:1433-1495 | PARTLY | Fixed text of the main return (prompt_builder.py:1475-1495), counted: 1,175 to 1,189 chars. Hidden notes add 175 (prompt_builder.py:1445) and about 201 plus skill names (:1450). Reaching 1,500 needs both notes. | "Say the base header is 1,175 to 1,189 chars, and give each hidden note its own count." |
| 18 | 85 | One file type loads, first found wins: .hermes.md, AGENTS.md, CLAUDE.md, .cursorrules | prompt_builder.py:1857 | PARTLY | prompt_builder.py:1863 names `.hermes.md/HERMES.md (walk to git root)`. Line 1864 also loads `.cursor/rules/*.mdc`. | Add HERMES.md and .cursor/rules/*.mdc to the list. |
| 19 | 87 | Telegram platform hint: 535 chars, ~134 tokens (est.) | prompt_builder.py:681, :696; system_prompt.py:445, :709 | PARTLY | PLATFORM_HINTS["telegram"] renders 617 (counted). `{_MEDIA_NATIVE}` at prompt_builder.py:699 adds 82. The doc's 535 is literal-only. | Label 535 as literal text only, or change to 617 chars (about 154 tokens, est.). |
| 20 | 108 | Default-deferred: 6 (config_defaults.py:2018-2025): session_search, todo_list, ... | hermes_cli/config_defaults.py:2018-2025 | PARTLY | The defer list holds 18 names (counted). Six are core tools: session_search, todo_list, process_manage, cronjob_manage, image_generate, computer_use. Twelve are desktop GUI names, for example `"drive_preview"` at :2022. | Say 18 default-deferred names, 6 of them core tools. |
| 21 | 109 | Eager candidates: 35. | toolsets.py:11-41; tools/kanban_tools.py:84-100, :1315 | PARTLY | toolsets.py:11-39 `_HERMES_CORE_TOOLS` holds 55 names (counted), including 14 kanban names at :29-35. The default hermes-cli bundle keeps them (toolsets.py:203 via `_bundle`). Only check_fn removes them: kanban_tools.py:90 `return _profile_has_kanban_toolset()`, and registry.py:853. Before check_fn for the default bundle: 55 minus 6 = 49. | Say 49 before check_fn for the default bundle. Give 35 as the count for bundles that drop kanban at selection (coding posture, toolsets.py:67; api-server, :201), or as the chat count after check_fn (not run). |
| 22 | 109 | So up to 38 names per request, before check_fn gates. | same as #21 | PARTLY | Same selection path: 49 eager plus 3 bridge tools = 52 before check_fn for the default bundle. | Say up to 52 before check_fn, and up to 38 after the kanban gate (not run). |

## Notes

- Token rounding: chars / 4 is truncated (663 gives 165.75, shown as ~165; 487 gives 121.75, shown as ~121). Not flagged.
- Line 14 cites only :402 for "plus any retries inside it." The retry loop is at conversation_loop.py:1497-1526. Verdict SUPPORTED, but the cite is weak.
- Line 42 gives no line for the /steer drain (turn_iteration_prep.py:179) or the wrap-up notice (:186). Both are inside prepare_iteration. Verified.
- Verified SUPPORTED with no change needed: the entry points (lines 21-27), the loop call chain and stop conditions (lines 29-59 except the grace claim), the tool-schema pipeline and bridge tools (lines 100-119), and doc table rows 1-5, 8, 10, 14-16, 18, 20, 21, plus the 58-skill count in row 13.
- Not traced, labeled as such in the doc (turn_recovery*.py, which gateway path a message takes): not checked.
