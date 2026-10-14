# Compliance audit: SQL concatenation in hermes-agent 97a9ae8ee5

Read-only review. No code changed.

## Summary

**47 patterns in 13 Python files.** All are SAFE by practical standards. No SQL injection vector. Every pattern parameterizes all user-supplied values with `?` placeholders. The string formatting is limited to:

- Hardcoded column-name constants (same file)
- Parameter-marker strings (`','.join('?' * n)`) for safe IN clauses
- Table names in schema migration helpers that already validate existence via PRAGMA

## Pattern breakdown

### A. Hardcoded SQL fragment constants (SAFE)

File | Line | Pattern | Reason
---|---|---|---
`agent/insights.py` | 96 | `f"SELECT {_SESSION_COLS} FROM sessions WHERE ..."` | `_SESSION_COLS` is a string constant defined in the same file
`agent/transcript_repair.py` | 75 | `f"SELECT ... IN ({_placeholders(chunk)})"` | `_placeholders` builds `?`-only parameter markers

### B. Migration/DDL helpers with PRAGMA-gated table names (SAFE)

These run `PRAGMA table_info` or `SELECT sql FROM sqlite_master WHERE name=?` before any ALTER/DROP. The table names come from a hardcoded list in the migration loop, not runtime input.

File | Lines | Pattern
---|---|---
`cron/delivery_queue.py` | 99-102 | `ALTER TABLE {table} RENAME` -- table iterated from hardcoded tuple on line 94
`gateway/hosted_rooms.py` | 317, 367, 990 | `CREATE TABLE ..._migrating`, `ALTER TABLE {table} ADD COLUMN` -- table from `_REQUIRED_COLUMNS` and `_LEGACY_COLUMNS` constants
`gateway/hosted_room_driver.py` | 201 | `CREATE TABLE IF NOT EXISTS {table}` -- table from `_table_name()` internal method

### C. Column lists from hardcoded tuples (SAFE)

File | Line | Pattern
---|---|---
`agent/transcript_repair.py` | 222 | `f"UPDATE messages SET {assignments} WHERE ..."` -- `assignments` built from known column names in the same file
`gateway/hosted_room_replicas.py` | 27-28, 235 | `f"INSERT INTO ... {_EVENT_COLUMNS}"` -- `_EVENT_COLUMNS` is a hardcoded parenthesized column-list constant
`gateway/hosted_rooms.py` | 131, 157-178 | All `_EVENT_COLUMNS`, `_ROOM_COLUMNS`, `_REMOTE_RUNS_BODY` etc. are file-level constants
`gateway/hosted_room_policy_checkpoint.py` | 94, 160, 348 | `f"SELECT {_ROOM_EVENT_COLUMNS} FROM ..."` -- constant in same file
`gateway/api_server.py` | 766-767 | `f"DELETE FROM ... IN ({placeholders})"` -- `placeholders` is `','.join('?' * n)`

### D. IN-clause parameter markers (SAFE)

File | Lines | Pattern
---|---|---
`evals/compaction/scripts/reconstruct_lineage.py` | 39 | `f"... IN ({','.join('?'*len(chain))})"` -- pure parameter markers
`gateway/hosted_room_driver.py` | 857 | Same pattern
`hermes_cli/kanban.py` | 624, 676 | Same pattern, `placeholders = ','.join('?' * len(ids))`
`hermes_cli/kanban_db.py` | 1384 | Same
`evals/postmortem/forensics/common.py` | 152 | `f"SELECT {cols} FROM messages"` -- `cols` built in the same function from a constant column list
`gateway/hosted_rooms.py` | 597, 598, 802, 889, 907, 1020, 1023 | All use either hardcoded column constants or parameter-marker patterns

## False positive patterns the hook might flag

- `evals/postmortem/tests/test_postmortem_harness.py` lines 43, 48: test data inserts, parameterized with `?`. Hook may flag because `f-string` wraps a Python dict. Safe.
- `gateway/hosted_rooms.py:990`: `f"SELECT 1 FROM sqlite_master WHERE type='table' AND name='{table}'"` -- `table` is from `_schema_is_current()` which only checks known internal tables, not user input. Low risk, could use `?` binding instead.

## Verdict

**All 47 patterns SAFE.** No SQL injection in the read source. The 19 files flagged by the compliance hook are false positives for a f-string-in-SQL rule. The hook is checking for a code-style pattern that in practice guards against bad practices, but Hermes already follows the correct discipline: all variable user data goes through `?` parameters, and the few f-string interpolated values are hardcoded constants or PRAGMA-validated table names.

## Recommendation

Close the 47 queue entries. No code change needed. If you want strict separation, the one minor item is `hosted_rooms.py:990` which uses f-string for a known table name where `?` binding would also work but buys nothing since the value is internal.