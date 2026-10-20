---
name: time-travel
description: Reconstruct the state of a Delta Lake table at a specific version. Use when the user wants to understand what the table looked like at a point in time — which files were active, what the schema was, how many rows existed. Triggers on phrases like "time travel", "what did the table look like at version N", "restore version", "table state at version", "go back to version".
---

# time-travel

Reconstruct the logical state of a Delta Lake table at a specific version by replaying the transaction log. No Spark, no library — pure log replay.

## Input

```
/delta-skills:time-travel path/to/table --version 10
```

Version is required.

## How Delta time travel works

The transaction log is append-only. To reconstruct the table at version N:

1. Find the latest checkpoint at or before version N
2. Start from that checkpoint's file list (if available), or from version 0
3. Replay all `add` and `remove` actions from that point up to version N
4. The result is the set of active Parquet files at version N

This is exactly what Spark does — we're just doing it manually by reading JSON.

## Steps

### 1. Validate the path is a Delta table

```bash
if [ ! -d "$TABLE_PATH/_delta_log" ]; then
  echo "ERROR: Not a Delta table."
  exit 1
fi
```

### 2. Find the latest checkpoint at or before the target version

Check `_delta_log/_last_checkpoint`:

```bash
cat "$TABLE_PATH/_delta_log/_last_checkpoint"
# Returns: {"version":40,"size":123}
```

If the checkpoint version is ≤ target version, use it as the starting point. Otherwise start from version 0.

Also check for checkpoint files directly:
```bash
ls "$TABLE_PATH/_delta_log/"*.checkpoint.parquet 2>/dev/null
```

### 3. Replay the log

Starting from the checkpoint (or version 0), read each commit JSON file up to and including the target version.

Maintain a set of active files:
- `add` action → add `path` to active set, store `partitionValues` and `stats`
- `remove` action → remove `path` from active set

Also track:
- The schema at the target version (most recent `metaData` action at or before version N)
- The partition columns

### 4. Report the reconstructed state

```
Table state at version 10: tables/customers
─────────────────────────────────────────────────────

Reconstructed from:  checkpoint v0 + 10 commit files
Active files:        84 parquet files
Estimated size:      1.8 GB
Estimated rows:      ~2,800,000
Partitioned by:      country

Schema at v10:
  id             long
  name           string
  email          string
  updated_at     timestamp
  (note: column 'country' was added later, in version 33)

Active files by partition:
  country=FI     18 files   ~510,000 rows
  country=SE     15 files   ~420,000 rows
  country=DE     22 files   ~640,000 rows
  country=NO     12 files   ~340,000 rows
  ... and 4 more partitions

To read these files, you would need:
  - The 84 Parquet files listed above
  - The schema as defined at v10
  - These files still exist on disk (unless VACUUM has run since v10)
```

### 5. Warn about VACUUM

If the target version is old, the underlying Parquet files may have been deleted by VACUUM. Note:

```
⚠ Warning: If VACUUM has run since version 10, the underlying Parquet files
  for removed versions may no longer exist on disk. Time travel only works
  if the files are still physically present.
```

Check the history for VACUUM operations using the commit log to give a concrete warning.

## Error handling

- If target version doesn't exist → report the max available version
- If checkpoint is a Parquet file (not readable as text) → skip it, replay from version 0 instead, note this limitation
- If log files are missing in the sequence → note the gap and continue with what's available

## Notes

- Delta checkpoints are Parquet files — reading them requires a Parquet-capable tool. If no such tool is available, fall back to replaying from version 0 using JSON files only.
- For tables with many versions and no checkpoints, replay from v0 may involve reading hundreds of JSON files — warn the user if this is the case
- `remove` actions include a `deletionTimestamp` — this is when the logical delete happened, not when VACUUM will clean up
- The default VACUUM retention is 7 days — files older than that may be gone
