---
name: inspect-table
description: Inspect a Delta Lake table — version, schema, file count, row stats, partitioning, and last commit info. Use when the user wants to understand the current state of a Delta table. Triggers on phrases like "inspect table", "what's in this delta table", "describe delta table", "check my delta table".
---

# inspect-table

Inspect a Delta Lake table by reading its `_delta_log` transaction log directly. No Spark, no `deltalake` library — just filesystem access.

## Input

The user provides a path to a Delta table directory:

```
/delta-skills:inspect-table path/to/table
```

## Steps

### 1. Validate the path is a Delta table

Check that `_delta_log` exists inside the given path:

```bash
TABLE_PATH="$1"

if [ ! -d "$TABLE_PATH/_delta_log" ]; then
  echo "ERROR: No _delta_log directory found at $TABLE_PATH"
  echo "This does not appear to be a Delta Lake table."
  exit 1
fi

echo "✓ Valid Delta table: $TABLE_PATH"
```

### 2. Find the latest version

List all JSON commit files in `_delta_log`, find the highest numbered one:

```bash
ls "$TABLE_PATH/_delta_log/"*.json 2>/dev/null | sort -V | tail -1
```

The filename without `.json` and leading zeros is the current version number.

### 3. Read the latest commit file

Read the latest `_delta_log/XXXXXXXXXX.json` file. Each line is a JSON action. Look for:

- `commitInfo` — operation name, timestamp, user, operation metrics
- `add` — files added, stats (numRecords, minValues, maxValues), partition values
- `remove` — files removed
- `metaData` — schema definition, partition columns, table configuration

### 4. Find and read the schema

The `metaData` action contains `schemaString` — a JSON-encoded schema. Parse it to extract column names and types. `metaData` may not be in the latest commit — scan backwards through log files until you find it, or check the latest checkpoint.

### 5. Count files and estimate rows

Replay `add` and `remove` actions across the log (or from the latest checkpoint) to determine:
- Active file count
- Total size estimate
- Row count estimate (sum of `numRecords` from add stats where available)

For a quick inspection, reading the last 10–20 commit files is sufficient for recent context.

### 6. Check for checkpoints

Look for `_delta_log/_last_checkpoint` — if it exists, read it to find the latest checkpoint version. Checkpoints (`.parquet` files) consolidate the log and can be used for faster state reconstruction.

```bash
if [ -f "$TABLE_PATH/_delta_log/_last_checkpoint" ]; then
  cat "$TABLE_PATH/_delta_log/_last_checkpoint"
fi
```

## Output format

Present a clean summary:

```
✓ Delta table: tables/customers
  Version:        42
  Last commit:    2025-03-18 09:12 UTC
  Operation:      MERGE
  Files:          127 parquet files (~2.4 GB)
  Estimated rows: ~4,200,000
  Partitioned by: country
  Checkpoint:     version 40

Schema:
  id             long       (not null)
  name           string
  email          string
  country        string
  updated_at     timestamp

Last operation metrics:
  numTargetRowsInserted:  1,204
  numTargetRowsUpdated:   11,196
  numOutputRows:          12,400
```

## Error handling

- If `_delta_log` is missing → tell the user it's not a Delta table
- If log files are empty or malformed → report which file failed and continue with what's available
- If schema is not found in recent commits → note that and suggest running `show-history` to find when it was last set
- If the table is very large (1000+ log files) → note that full replay may be slow and offer to read only recent commits

## Notes

- Each line in a Delta log JSON file is an independent JSON object (newline-delimited JSON, not a JSON array)
- `commitInfo` is always the first action in a commit file
- `metaData` only appears when the schema or configuration changes — it is not in every commit
- Partition values appear in `add.partitionValues` as a map of column name to string value
