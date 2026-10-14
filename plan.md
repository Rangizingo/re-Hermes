# re-Hermes plan

Goal: evidence-backed writeups on the Hermes Agent source at `97a9ae8ee5`. The installed build `d604141` is for comparison only. Status 2026-10-14. Main at `7ea612b`.

## Units

- [x] U1 CLAUDE.md. PR #1 merged. verified: `gh pr view 1` shows MERGED; `git show origin/main:CLAUDE.md | grep -c "Ghidra 12.1.4 is installed"` prints 1.
- [x] U2 Core-loop map, `docs/core-loop.md`. PR #2 merged (`ce4f80b`). verified: `check_cites.sh` on main: TOTAL=198 ok=198 missing=0 range_bad=0. Two Haiku reviewers checked 205 claims; flagged items verified against source and fixed.
- [x] U3 Entry points, cron, and state, `docs/surfaces.md`. PR #3 merged (`7ea612b`). verified: `check_surfaces.sh` on main: TOTAL=107 ok=104 missing=0 range_bad=0; the 3 ambiguous are root-level `cli.py` cites. One Haiku reviewer checked 183 claims; 18 edits.
- [x] U4 REA with Ghidra. Ghidra 12.1.4 via Homebrew. verified: REA doctor with `GHIDRA_INSTALL_DIR` set reports healthy true, ghidra ok (2026-10-14). `~/.zshrc` line read back in a fresh zsh.
- [x] U5 Third writeup, `docs/memory-skills-api.md` (branch `docs/memory-skills-api`). verified: sentence review fixed (1 NOT SUPPORTED, 9 PARTLY, ~22 NO CITE all addressed). PR #4 ready for review at `3772e95`. Do not merge without Peter.
- [-] U6 Dropped on Peter's instruction (2026-10-14): the A9 and A10 proposals were removed. Nothing was applied to `hermes/plan.md`.
- [x] U7 Measurement on ai-core, read-only. Done. Report: `evidence/trace-ai-core-2026-10-14.md`. verified: spot-check on the local copy gives 12 `background_review` rows in total, last at 2026-10-12 11:52:52 UTC, and 0 since the 16:15 UTC restart (matches). Results: A, zero review runs since the restart in 77 Telegram turns; B, median 2 model rows per Telegram turn (n=273), p90 17, max 243; C, bridge used in 10.3% of turns and those turns hold 44.4% of model rows; D, stall-guard text 0 hits in 7 days; E, median 23 rows per gateway session (n=60). Limit: about 34 h of recorded traffic after the switch, and the switch time is not recorded.
- [x] U8 Compliance audit done. 47 patterns in 13 files, all SAFE (no SQL injection). Report: `docs/compliance-audit.md`. Queue closed.

## Notes

- REA's binary tools do not apply to Python source. The writeups read source directly.
- Goal cleared 2026-10-14 on Peter's answer (`.claude/goal.md` removed).
- Ghost-skill gap (`docs/memory-skills-api.md`, discrepancy bullet): read from code, not reproduced. A candidate upstream issue, not confirmed.
- Open PRs: #4 (docs/memory-skills-api, ready for review).

## 2026-10-14 close-out

Shipped
- PRs #1 (CLAUDE.md), #2 (core-loop map), and #3 (surfaces) are merged to main at `7ea612b`.
- Ghidra 12.1.4 is installed. The REA doctor passes with `GHIDRA_INSTALL_DIR` set.
- The ai-core trace is done, counts only: `evidence/trace-ai-core-2026-10-14.md`.
- The third writeup is drafted and committed as WIP at `6f7fc98`. Not pushed.

Decided
- Target is the newest source, `97a9ae8ee5`. The done line is the writeup, with no rebuild.
- The A9 and A10 proposals were removed on Peter's instruction. Nothing was applied to `hermes/plan.md`.
- The autopilot goal file is removed, so the chain has ended.
- The 47 compliance queue entries reviewed: all SAFE. Queue closed.

Corrected (earlier claims that were wrong)
- Core-loop said 35 eager tools (38 with the bridge). Corrected to 49 (52 with the bridge) before `check_fn`: `hermes_cli/config_defaults.py:2018-2025`, `toolsets.py:11-41`.
- Core-loop said the compression trigger is 0.50 everywhere. Under 512K the floor is 0.75: `agent/context_compressor.py:2757-2758`.
- Core-loop said every spent budget gets one grace call. Only a codex reasoning-only fallback does: `agent/turn_truncation.py:635`.
- Core-loop said HEAD does not match 59004a623 at `system_prompt.py:41-52`. Both are identical at that ref.
- Surfaces said `update_job` is the only edit path. `rearm_oneshot` and `rewrite_skill_refs` also call `save_jobs` directly.
- Surfaces said 16 table names. Corrected to 18 with the two Telegram topic tables.
- CLAUDE.md and README said Ghidra is not installed. Fixed in PR #1.
- Third writeup: all review issues fixed (usage tag 21-27 chars, line 120 repair clause corrected, line 96 wording fixed).

Coverage gaps closed with `docs/runtime-traced-gaps.md`.

Remaining
- Peter: the A1 proof in `hermes/plan.md` should say "since the switch". The trace shows the 48-hour window can't pass.
- Peter: merge PR #4 when ready.
