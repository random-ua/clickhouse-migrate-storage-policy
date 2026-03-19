---
name: clickhouse-migrate-storage-policy
description: Migrates all tables in a ClickHouse database to a new storage policy with zero downtime. Use this skill whenever the user wants to move ClickHouse tables to a different storage policy, disk, or S3 bucket — even if they say things like "migrate to s3_2", "change storage policy", "move tables to new storage", "switch disks", or "copy tables to different policy". The skill creates shadow tables under a `_s3_<name>` prefix, copies data, atomically swaps with EXCHANGE TABLES, verifies row counts, and drops the originals.
---

# ClickHouse Storage Policy Migration

Migrates all tables in a database to a new storage policy with zero downtime by using shadow tables and atomic swaps.

## Tool conventions

- **Read-only queries** (system.tables, system.parts, SELECT): use MCP `mcp__clickhouse__run_select_query`
- **Write operations** (CREATE, INSERT, EXCHANGE, DROP): use Bash with:
  ```
  clickhouse client --connection prod-admin --max_execution_time=3600 --receive_timeout=3600 --send_timeout=3600
  ```

## Step-by-step process

### Step 1 — Discover tables and DDLs

Query `system.tables` for the target database:

```sql
SELECT name, create_table_query
FROM system.tables
WHERE database = '<db>'
ORDER BY name
```

This gives the exact DDL for each table, which you'll use as the template for the shadow tables.

### Step 2 — Create shadow tables

For each table, derive `_s3_<original_name>` by modifying the DDL:
- Change the table name from `<db>.<name>` → `<db>._s3_<name>`
- Replace `storage_policy = '<old>'` → `storage_policy = '<new>'`

Run all CREATE statements in a single `clickhouse client` call (ClickHouse accepts multiple `;`-separated statements). This is fast and can be batched.

### Step 3 — Copy data

INSERT sequentially, one table at a time — parallel inserts can strain the server:

```sql
INSERT INTO <db>._s3_<name> SELECT * FROM <db>.<name>
```

Run each as a separate `clickhouse client` call so you get clear per-table success/failure signals.

### Step 4 — Verify row counts

Before swapping, confirm row counts match using `system.parts`:

```sql
SELECT table, sum(rows) AS row_count
FROM system.parts
WHERE database = '<db>' AND active = 1
GROUP BY table
ORDER BY table
```

Every `_s3_<name>` table must match its source. Do not proceed if counts diverge.

### Step 5 — Atomic swap

Use `EXCHANGE TABLES` — this is atomic and requires no downtime:

```sql
EXCHANGE TABLES <db>.<name> AND <db>._s3_<name>
```

All swaps can be batched in one `clickhouse client` call.

### Step 6 — Verify storage policies

After the swap, confirm the original-named tables now report the new policy and the `_s3_` tables report the old policy:

```sql
SELECT name, engine_full
FROM system.tables
WHERE database = '<db>'
ORDER BY name
```

### Step 7 — Final size/row comparison

Pull compressed sizes and row counts for both sets from `system.parts`:

```sql
SELECT table, sum(rows) AS row_count,
       sum(data_compressed_bytes) AS compressed_bytes,
       sum(data_uncompressed_bytes) AS uncompressed_bytes
FROM system.parts
WHERE database = '<db>' AND active = 1
GROUP BY table
ORDER BY table
```

Present a side-by-side table (original vs `_s3_` backup). Minor byte differences due to part merging state are normal — row counts must be identical.

### Step 8 — Drop backup tables

Only drop after the user confirms they're satisfied with the verification:

```sql
DROP TABLE <db>._s3_<name>;
-- repeat for each table
```

## Output at each step

Tell the user what you're about to do and confirm success at each stage. Present verification results as a markdown table so discrepancies are easy to spot.
