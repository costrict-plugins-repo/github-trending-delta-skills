---
name: show-log
description: Read and explain a specific commit from a Delta Lake transaction log. Use when the user wants to understand exactly what happened in a specific version — which files were added or removed, what the operation metadata says. Triggers on phrases like "show log", "explain commit", "what happened in version N", "read delta log", "inspect version".
---

# show-log

Read and explain a specific Delta Lake commit file in detail. Opens the raw `_delta_log/XXXXXXXXXX.json` for the given version and explains every action inside it.

## Input

```
/delta-skills:show-log path/to/table --version 5
```

Version is required.

## Steps

### 1. Validate the path is a Delta table

```bash
TABLE_PATH="$1"

if [ ! -d "$TABLE_PATH/_delta_log" ]; then
  echo "ERROR: No _delta_log directory found at $TABLE_PATH"
  exit 1
fi
```

### 2. Construct the commit file path

Delta log filenames are zero-padded to 20 digits:

```bash
VERSION=$(printf "%020d" $VERSION_NUMBER)
COMMIT_FILE="$TABLE_PATH/_delta_log/${VERSION}.json"

if [ ! -f "$COMMIT_FILE" ]; then
  echo "ERROR: No commit file found for version $VERSION_NUMBER"
  echo "File expected at: $COMMIT_FILE"
  exit 1
fi
```

### 3. Read the commit file

Each line is an independent JSON object (newline-delimited). Read all lines and categorize by `action type`:

- `commitInfo` — operation metadata
- `metaData` — schema or config change
- `protocol` — reader/writer version requirements
- `add` — file added to the table
- `remove` — file removed (logically deleted)
- `txn` — streaming transaction identifier

### 4. Explain each action

**commitInfo** — Always show in full:
- Operation name and parameters
- Timestamp (human-readable)
- Engine and version that wrote this commit
- Isolation level
- All operation metrics

**metaData** — If present, this was a schema or config change. Show:
- Full schema (formatted, not raw JSON)
- Partition columns
- Table configuration (e.g. `delta.enableDeletionVectors`, `delta.autoOptimize`)
- Note this as significant — schema changes affect all downstream readers

**add actions** — Summarize, don't list every file unless < 10:
- Total count of files added
- Total bytes added
- Sample paths (first 3)
- Partition values if partitioned
- Stats summary: total rows, min/max of key columns if available

**remove actions** — Same treatment as add:
- Total count of files removed
- Whether `dataChange` is true (real deletion) or false (OPTIMIZE/compaction)
- Note: removed files are still physically present until VACUUM runs

**protocol** — If present, note the reader/writer version requirements. Highlight if this means older readers will break.

## Output format

```
Commit: version 42 — tables/customers
─────────────────────────────────────────────────────

Operation:   MERGE
Timestamp:   2025-03-18 09:12:33 UTC
Engine:      Apache-Spark/3.5.0 Delta-Lake/3.3.0
Isolation:   Serializable
User:        svc-pipeline@gmail.com

Metrics:
  numSourceRows:              24,000
  numTargetRowsInserted:       1,204
  numTargetRowsUpdated:       11,196
  numTargetRowsDeleted:            0
  numOutputRows:              12,400
  executionTimeMs:            18,432

Files added (14 files, 284 MB total):
  part-00000-abc123.snappy.parquet  (country=FI, 22 MB, 94,200 rows)
  part-00001-def456.snappy.parquet  (country=SE, 19 MB, 81,400 rows)
  part-00002-ghi789.snappy.parquet  (country=DE, 24 MB, 103,100 rows)
  ... and 11 more

Files removed (14 files, 281 MB total):
  These files are logically deleted. They remain on disk until VACUUM runs.
  dataChange: true — this was a real data modification, not a compaction.
```

## Error handling

- If the version doesn't exist → list the available version range
- If a line fails to parse as JSON → skip it, note the line number and raw content
- If `commitInfo` is missing → note this is unusual and show what actions are present

## Notes

- Each line in the file is a self-contained JSON object — do NOT try to parse the whole file as a single JSON document
- `remove` actions with `dataChange: false` are from OPTIMIZE or similar compaction — no actual data was deleted
- `add.stats` contains column-level min/max/nullCount — very useful for understanding data distribution
- For large commits (100+ files), summarize rather than listing everything
