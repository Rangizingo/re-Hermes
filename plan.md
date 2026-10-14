# re-Hermes plan

Goal: evidence-backed writeups on the Hermes Agent source at `97a9ae8ee5`. The installed build `d604141` is for comparison only. Status 2026-10-14. Main at `7ea612b`.

## Units

- [x] U1 CLAUDE.md. PR #1 merged. verified: `gh pr view 1` shows MERGED; `git show origin/main:CLAUDE.md | grep -c "Ghidra 12.1.4 is installed"` prints 1.
- [x] U2 Core-loop map, `docs/core-loop.md`. PR #2 merged (`ce4f80b`). verified: `check_cites.sh` on main: TOTAL=198 ok=198 missing=0 range_bad=0. Two Haiku reviewers checked 205 claims; flagged items verified against source and fixed.
- [x] U3 Entry points, cron, and state, `docs/surfaces.md`. PR #3 merged (`7ea612b`). verified: `check_surfaces.sh` on main: TOTAL=107 ok=104 missing=0 range_bad=0; the 3 ambiguous are root-level `cli.py` cites. One Haiku reviewer checked 183 claims; 18 edits.
- [x] U4 REA with Ghidra. Ghidra 12.1.4 via Homebrew. verified: REA doctor with `GHIDRA_INSTALL_DIR` set reports healthy true, ghidra ok (2026-10-14). `~/.zshrc` line read back in a fresh zsh.
- [~] U5 Third writeup, `docs/memory-skills-api.md` (branch `docs/memory-skills-api`, committed as WIP, not pushed as a PR). verified: citation check TOTAL=120 ok=120 missing=0 range_bad=0. Spot-checks: skill summary has no marker below 5,000 chars; API server has 0 `load_transcript` calls and 2 `get_messages_as_conversation` calls; memory caps 2,200 and 1,375. Sentence review done (`evidence/review-memory-skills-api.md`): 107 scored, 1 NOT SUPPORTED (line 120, repair clause inverted), 9 PARTLY (lines 38-39 usage tag is 21-27 chars, not 6), 22 NO CITE. Next: fix those, re-run the citation check, open PR. Do not merge without Peter.
- [-] U6 Dropped on Peter's instruction (2026-10-14): the A9 and A10 proposals were removed. Nothing was applied to `hermes/plan.md`.
- [x] U7 Measurement on ai-core, read-only. Done. Report: `evidence/trace-ai-core-2026-10-14.md`. verified: spot-check on the local copy gives 12 `background_review` rows in total, last at 2026-10-12 11:52:52 UTC, and 0 since the 16:15 UTC restart (matches). Results: A, zero review runs since the restart in 77 Telegram turns; B, median 2 model rows per Telegram turn (n=273), p90 17, max 243; C, bridge used in 10.3% of turns and those turns hold 44.4% of model rows; D, stall-guard text 0 hits in 7 days; E, median 23 rows per gateway session (n=60). Limit: about 34 h of recorded traffic after the switch, and the switch time is not recorded.
- [ ] U8 Parked: 47 compliance queue entries in the upstream clone. Review only. Peter's decision.

## Notes

- REA's binary tools do not apply to Python source. The writeups read source directly.
- Goal cleared 2026-10-14 on Peter's answer (`.claude/goal.md` removed).
- Ghost-skill gap (`docs/memory-skills-api.md`, discrepancy bullet): read from code, not reproduced. A candidate upstream issue, not confirmed.
- Open PRs: none. The third writeup's PR is not open yet.
