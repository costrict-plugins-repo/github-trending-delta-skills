# delta-skills

A Claude Code plugin for inspecting and debugging Delta Lake tables — no Spark, no libraries, no setup. Just your tables and Claude Code.

## What is this?

`delta-skills` gives you slash commands that let you explore Delta Lake tables directly from Claude Code. It reads the `_delta_log` transaction log natively — no `deltalake`, no PySpark, no Databricks. Just filesystem access and Claude's intelligence to interpret what it finds.

## Installation

```
/plugin marketplace add chanukyapekala/delta-skills
/plugin install delta-skills@delta-skills
```

Skills will be available as `/delta-skills:<skill-name>` in all future sessions.

### Local development

```bash
git clone https://github.com/chanukyapekala/delta-skills.git
cd delta-skills
claude --plugin-dir .
```

## Skills

### `inspect-table`

Inspect a Delta table — version, schema, file count, row stats, partitioning, last commit.

```
/delta-skills:inspect-table path/to/table
```

### `show-history`

Show the commit history of a Delta table — version, timestamp, operation, rows written.

```
/delta-skills:show-history path/to/table
/delta-skills:show-history path/to/table --limit 10
```

### `show-log`

Read and explain a specific commit from the transaction log.

```
/delta-skills:show-log path/to/table --version 5
```

### `time-travel`

Reconstruct the state of a Delta table at a specific version.

```
/delta-skills:time-travel path/to/table --version 3
```

### `validate`

Validate the integrity of a Delta table — check for missing files, broken log entries, schema inconsistencies.

```
/delta-skills:validate path/to/table
```

## How it works

A Delta table is a directory with a `_delta_log` subdirectory containing JSON commit files (and occasional Parquet checkpoints). Every operation — write, delete, merge, schema change — is recorded there.

`delta-skills` reads those files directly. No library required. Claude interprets the raw log and surfaces what matters.

## User journey

You have a local lakehouse or synced Delta tables:

```
tables/
├── customers/
├── orders/
└── products/
```

Open Claude Code and type:

```
/delta-skills:inspect-table tables/customers
```

Get back:

```
✓ Delta table: tables/customers
  Version:     42
  Last commit: 2025-03-18 09:12
  Files:       127 parquet files
  Schema:      id (long), name (string), email (string), updated_at (timestamp)
  Partitioned: country
```

No Spark cluster. No SDK. No setup.

## Prerequisites

- Claude Code
- A Delta Lake table (local or mounted path)
- That's it

## Platform support

Tested on macOS and Linux. Windows support is not yet verified.

## Contributing

This is v0.0.1 — early and intentionally minimal. Contributions, ideas, and new skills are welcome.

Open an issue or submit a PR at: https://github.com/chanukyapekala/delta-skills

## About

Built by [@chanukyapekala](https://github.com/chanukyapekala) and the [DataTribe Collective](https://datatribecollective.com) community.

Inspired by [duckdb-skills](https://github.com/duckdb/duckdb-skills).

## License

MIT
