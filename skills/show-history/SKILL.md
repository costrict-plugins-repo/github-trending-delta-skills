---
name: show-history
description: Show the commit history of a Delta Lake table — version, timestamp, operation, and row metrics. Use when the user wants to see what changed over time in a Delta table. Triggers on phrases like "show history", "delta table history", "what changed", "commit log", "audit trail".
---

# show-history

Show the full commit history of a Delta Lake table by reading all JSON files in `_delta_log`. No Spark, no library — just filesystem access.

## Input

```
/delta-skills:show-history path/to/table
/delta-skills:show-history path/to/table --limit 10
```

Default limit: 20 most recent versions.

## Steps

### 1. Validate the path is a Delta table

```bash
TABLE_PATH="$1"

if [ ! -d "$TABLE_PATH/_delta_log" ]; then
  echo "ERROR: No _delta_log directory found at $TABLE_PATH"
  echo "This does not appear to be a Delta Lake table."
  exit 1
fi
```

### 2. List all commit files

```bash
ls "$TABLE_PATH/_delta_log/"*.json 2>/dev/null | sort -V
```

This gives you all commit files in version order. Apply `--limit` by taking the last N files.

### 3. Extract commitInfo from each file

For each commit file, read line by line and find the line containing `"commitInfo"`. Parse:

- `timestamp` — milliseconds since epoch, convert to human-readable UTC
- `operation` — e.g. `WRITE`, `MERGE`, `DELETE`, `UPDATE`, `CREATE TABLE`, `OPTIMIZE`, `VACUUM END`
- `operationParameters` — mode, predicate, etc. (summarize briefly)
- `operationMetrics` — key metrics depending on operation type:
  - WRITE: `numFiles`, `numOutputRows`, `numOutputBytes`
  - MERGE: `numTargetRowsInserted`, `numTargetRowsUpdated`, `numTargetRowsDeleted`
  - DELETE: `numDeletedRows`
  - OPTIMIZE: `numFilesAdded`, `numFilesRemoved`, `numRemovedBytes`
- `userName` or `userId` — who made the change (if present)
- `engineInfo` — what engine wrote it, e.g. `Apache-Spark/3.5.0 Delta-Lake/3.1.0`

### 4. Format and display

Present as a table, most recent first:

```
Delta table history: tables/customers
Showing last 10 versions (total: 42 commits)

Version  Timestamp              Operation   Rows        Details
───────  ─────────────────────  ──────────  ──────────  ─────────────────────────────
42       2025-03-18 09:12 UTC   MERGE       +1,204      updated: 11,196  deleted: 0
41       2025-03-17 08:55 UTC   MERGE       +980        updated: 10,220  deleted: 12
40       2025-03-16 09:01 UTC   OPTIMIZE    —           files: -34 +8    saved: 1.2GB
39       2025-03-15 08:48 UTC   MERGE       +1,102      updated: 9,887   deleted: 0
38       2025-03-14 09:03 UTC   MERGE       +1,340      updated: 12,001  deleted: 4
37       2025-03-13 08:59 UTC   MERGE       +890        updated: 9,450   deleted: 0
36       2025-03-12 08:44 UTC   VACUUM END  —           deleted: 23 files
35       2025-03-11 09:15 UTC   MERGE       +1,550      updated: 13,200  deleted: 2
34       2025-03-10 08:52 UTC   MERGE       +1,100      updated: 10,900  deleted: 0
33       2025-03-09 09:05 UTC   SCHEMA CHANGE  —        added column: loyalty_tier (string)
```

## Output variations by operation

Adapt the "Details" column per operation type:

- `WRITE` → `mode: append|overwrite  rows: N`
- `MERGE` → `inserted: N  updated: N  deleted: N`
- `DELETE` → `deleted: N rows`
- `UPDATE` → `updated: N rows`
- `OPTIMIZE` → `files: -N +N  saved: X MB`
- `VACUUM START/END` → `deleted: N files`
- `CREATE TABLE` → `schema set, partitioned by: X`
- `REPLACE TABLE` → note schema or partition change

## Error handling

- If no JSON commit files found → tell the user the table may be empty or corrupted
- If a commit file is missing `commitInfo` → skip it and note the gap
- If timestamps look wrong → display raw value alongside converted

## Notes

- `commitInfo` is always the first newline-delimited JSON object in each commit file
- Timestamps are in milliseconds since Unix epoch
- `operationMetrics` keys vary by engine version — be tolerant of missing keys
- Some older Delta tables written by non-Spark engines may have minimal `commitInfo`
