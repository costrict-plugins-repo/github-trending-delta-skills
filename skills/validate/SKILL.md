---
name: validate
description: Validate the integrity of a Delta Lake table — check for missing files, broken log entries, schema inconsistencies, and common issues. Use when the user suspects something is wrong with a Delta table or wants a health check. Triggers on phrases like "validate table", "check delta table", "is this table healthy", "something wrong with my table", "debug delta table", "pipeline failed".
---

# validate

Validate the integrity of a Delta Lake table by checking its transaction log and physical files. Surfaces common issues without requiring any Delta library.

## Input

```
/delta-skills:validate path/to/table
```

## What this checks

1. Basic structure
2. Transaction log continuity
3. Schema consistency
4. Physical file presence
5. Common operational issues

## Steps

### 1. Basic structure check

```bash
TABLE_PATH="$1"

# Check _delta_log exists
[ -d "$TABLE_PATH/_delta_log" ] || { echo "FAIL: No _delta_log directory"; exit 1; }

# Check there are commit files
COMMIT_COUNT=$(ls "$TABLE_PATH/_delta_log/"*.json 2>/dev/null | wc -l)
[ "$COMMIT_COUNT" -gt 0 ] || { echo "FAIL: No commit files in _delta_log"; exit 1; }

echo "✓ _delta_log exists with $COMMIT_COUNT commit files"
```

### 2. Transaction log continuity

List all JSON commit files and check that version numbers are sequential with no gaps:

```bash
ls "$TABLE_PATH/_delta_log/"*.json | sort -V
```

Expected: `00000000000000000000.json`, `00000000000000000001.json`, etc.

- If there are gaps (e.g. v3 missing between v2 and v4) → this is a serious integrity issue
- If the sequence starts from a non-zero checkpoint → that's normal, note it

### 3. Checkpoint consistency

If `_last_checkpoint` exists, verify:
- The checkpoint version it references has a corresponding `.checkpoint.parquet` file
- The checkpoint version is ≤ the latest commit version

```bash
if [ -f "$TABLE_PATH/_delta_log/_last_checkpoint" ]; then
  cat "$TABLE_PATH/_delta_log/_last_checkpoint"
  # Verify the referenced checkpoint file exists
fi
```

### 4. Schema consistency

Scan all `metaData` actions across the log:
- Extract schema from each `metaData` occurrence
- Check for unexpected schema changes (column drops, type changes)
- Note any schema evolution (column additions are normal and safe)

Flag:
- Column type changes (e.g. `int` → `string`) — breaking change
- Column drops — data loss risk
- Multiple schema changes in a short time window — unusual

### 5. Recent commit health

Read the last 5 commit files and check:
- Each file has a valid `commitInfo` as the first line
- No commit file is empty or zero bytes
- Timestamps are increasing (no time going backwards)
- No duplicate version numbers

### 6. Physical file spot check

For the 5 most recently added Parquet files (from recent `add` actions), check they exist on disk:

```bash
# For each recent add action path
ls "$TABLE_PATH/path/to/part-file.parquet" 2>/dev/null || echo "MISSING: ..."
```

If files are missing → the table has lost physical data. This is serious.

### 7. Deletion vector check

If any `add` actions reference `deletionVector`, note that deletion vectors are in use. Check that the referenced `.bin` files exist alongside their Parquet files.

### 8. OPTIMIZE and VACUUM history

Check the last 20 commits for:
- Last OPTIMIZE run — if > 7 days ago, table may have too many small files
- Last VACUUM run — note the retention threshold if visible in operationParameters
- If VACUUM has never run → warn about accumulating deleted files

## Output format

```
Delta table validation: tables/customers
─────────────────────────────────────────────────────

Structure
  ✓ _delta_log exists
  ✓ 42 commit files found (v0 → v41)
  ✓ No gaps in version sequence
  ✓ Checkpoint at v40, file present

Schema
  ✓ Schema consistent across all versions
  ℹ Schema changes detected:
      v0:  original schema (4 columns)
      v33: added column loyalty_tier (string) — safe, additive

Recent commits (last 5)
  ✓ All commit files valid and non-empty
  ✓ Timestamps increasing
  ✓ No duplicate versions

Physical files (spot check, last 5 added files)
  ✓ All 5 sampled files present on disk

Operational health
  ℹ Last OPTIMIZE: v40 (2025-03-16) — 2 days ago, looks healthy
  ⚠ Last VACUUM: v36 (2025-03-12) — 6 days ago, approaching 7-day default retention
  ℹ Deletion vectors: not in use

─────────────────────────────────────────────────────
Summary: 1 warning, 0 errors

Recommendation: Run VACUUM soon to clean up deleted files before they accumulate.
```

## Severity levels

- `✓` — healthy
- `ℹ` — informational, no action needed
- `⚠` — warning, action recommended
- `✗` — error, table may be broken or data may be at risk

## Common issues and what they mean

**Missing commit files (gaps in sequence)**
The transaction log is corrupted. Some versions are unreadable. Do not write to this table until investigated.

**Missing physical Parquet files**
Data has been lost. Files referenced in the log no longer exist. This usually means VACUUM ran with too short a retention period, or files were deleted manually.

**Timestamps going backwards**
Multiple writers with clock skew, or a commit was manually edited. Usually harmless but worth noting.

**Very large number of small files**
Table has never been OPTIMIZE'd or OPTIMIZE runs infrequently. Query performance will degrade.

**Checkpoint file missing**
`_last_checkpoint` points to a checkpoint that doesn't exist. Delta readers will fall back to replaying from v0, which will be slow for large tables.
