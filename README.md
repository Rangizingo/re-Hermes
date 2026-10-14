# re-Hermes

Reverse engineering of the Hermes Agent (Nous Research, MIT licensed).

## Target

- Source: `hermes-agent` clone, commit `97a9ae8ee5` (2026-10-11). This is the newest source and the one analyzed.
- Installed reference: Hermes v0.19.0, build `d604141` (2026-07-21), at `/Users/peterb/.hermes/hermes-agent`. Used only to compare against the source.

## Scope

- Output is an evidence-backed writeup. No rebuild.
- Question: where each turn's model rounds and prompt tokens come from. Feeds the local-model speed work in `/Users/peterb/Documents/AI/hermes`.
- Both Hermes trees are read-only here. Nothing in them changes.

## Layout

- `docs/`: writeups.
- `evidence/`: captured output the writeups cite.

## Known gaps

- REA's binary tools need Ghidra. Ghidra is not installed yet.
- The compliance hook flags 19 files in the upstream clone for SQL string concatenation (47 queue entries). Not edited here. Upstream code, needs human review of which values are user-supplied.
