---
name: clickhouse-migrate-storage-policy
description: Migrates all tables in a ClickHouse database to a new storage policy with zero downtime. Use this skill whenever the user wants to move ClickHouse tables to a different storage policy, disk, or S3 bucket — even if they say things like "migrate to s3_2", "change storage policy", "move tables to new storage", "switch disks", or "copy tables to different policy". The skill creates shadow tables under a `_s3_<name>` prefix, copies data, atomically swaps with EXCHANGE TABLES, verifies row counts, and drops the originals. Handles interrupted/partial migrations safely.
---

# ClickHouse Storage Policy Migration

Migrates all tables in a database to a new storage policy with zero downtime using shadow tables and atomic swaps. Handles interrupted migrations safely by inspecting pre-existing shadow tables before doing anything destructive.

## Tool conventions

- **Read-only queries** (system.tables, system.parts, SELECT): use MCP `mcp__clickhouse__run_select_query`
- **Write operations** (CREATE, INSERT, DROP): MCP works for these — the tool accepts any SQL, not just SELECT.
- **EXCHANGE TABLES**: must use the CLI. The MCP user lacks the DROP TABLE privilege that EXCHANGE requires internally. Multi-statement queries (`;`-separated) are also not supported by MCP — run one statement per call.
  ```
  clickhouse client --connection prod-admin --max_execution_time=3600 --receive_timeout=3600 --send_timeout=3600
  ```

## Step-by-step process

### Step 1 — Discover tables and DDLs

Query `system.tables` for the target database. Only target tables with the old storage policy — skip tables already on the new policy, on `local`, or with no explicit policy:

```sql
SELECT name, create_table_query, engine_full
FROM system.tables
WHERE database = '<db>'
  AND engine LIKE '%MergeTree%'
  AND name NOT LIKE '_s3_%'
ORDER BY name
```

Also fetch any existing `_s3_` shadow tables and their policies — you'll need this for interrupt recovery:

```sql
SELECT name, engine_full
FROM system.tables
WHERE database = '<db>' AND name LIKE '_s3_%'
ORDER BY name
```

### Step 2 — Triage each table (interrupt recovery)

For each target table, determine which migration step to resume from. This prevents data corruption if a previous run was interrupted.

For each table `<name>`, check whether `_s3_<name>` already exists:

**Case A — `_s3_<name>` does not exist:** Normal path. Proceed to CREATE (Step 3).

**Case B — `_s3_<name>` exists with the NEW storage policy (e.g. `s3_2`):**
- Check row counts for both tables via `system.parts` (active=1).
- If `_s3_` has 0 rows: shadow was created but INSERT was interrupted. Resume from INSERT (Step 4).
- If `_s3_` has rows < source: INSERT was partially interrupted. The safe move is to DROP the partial shadow and re-INSERT. Confirm this is safe (no data exists only in the shadow).
- If `_s3_` rows == source rows: INSERT completed. Resume from EXCHANGE (Step 5).
- If `_s3_` rows > source rows: something unexpected — stop and ask the user before touching anything.

**Case C — `_s3_<name>` exists with the OLD storage policy (e.g. `s3`):**
- This means EXCHANGE already happened: `_s3_` is now the backup holding old-policy data, and `<name>` is already on the new policy.
- Verify: check `<name>` engine_full shows the new policy, and row counts are equal.
- If counts match and policies are correct: this table is done. Just DROP the `_s3_` backup (Step 8).
- If counts don't match: stop and ask the user — do not drop anything.

**Never drop a table until you have confirmed the other copy has all the data.**

### Step 3 — Create shadow tables

For tables that need it (Case A), derive `_s3_<name>` DDL from `SHOW CREATE TABLE <db>.<name>`:
- Change the table name from `<db>.<name>` → `<db>._s3_<name>`
- Replace `storage_policy = '<old>'` → `storage_policy = '<new>'`

**`LowCardinality(<non-String>)` columns:** If the source DDL contains `LowCardinality(UInt32)` or other non-String LowCardinality types, add `allow_suspicious_low_cardinality_types = 1` to the shadow's SETTINGS, otherwise ClickHouse rejects the CREATE.

### Step 4 — Copy data

INSERT sequentially, one table at a time (parallel inserts strain the server):

```sql
INSERT INTO <db>._s3_<name> SELECT * FROM <db>.<name>
```

Run each as a separate CLI call. After each INSERT, immediately check row counts match before moving to the next table.

**If INSERT SELECT OOMs**, try two escalating fixes:

1. **Reduce block size** — add `--max_block_size=5000` to the CLI call. This lowers per-block memory, often enough for moderately large tables.

2. **If still OOM (often caused by projections)** — use `ATTACH PARTITION FROM` instead of INSERT SELECT. This copies parts server-side without loading data through query memory:
   - First, add projection definitions to the shadow (without materializing) so both tables have matching projections:
     ```sql
     ALTER TABLE <db>._s3_<name> ADD PROJECTION <proj_name> (<proj_query>)
     ```
   - Drop any partial partitions from the shadow, then attach partition-by-partition:
     ```sql
     ALTER TABLE <db>._s3_<name> ATTACH PARTITION '<YYYYMM>' FROM <db>.<name>
     ```
   - Check which partitions exist via `system.parts` (group by `partition`) to know what to attach.
   - `ATTACH PARTITION FROM` copies existing projection data from the source parts — no recomputation needed.

**Live tables (ReplacingMergeTree etc.):** After INSERT SELECT completes, the source may have a few more rows from concurrent writes. A small diff (tens to hundreds of rows) is safe — the application will continue writing to the live table after EXCHANGE. Do not re-INSERT just for this.

### Step 5 — Verify row counts

Before swapping, confirm all shadow tables match their sources using `system.parts`:

```sql
SELECT table, sum(rows) AS row_count
FROM system.parts
WHERE database = '<db>' AND active = 1
GROUP BY table
ORDER BY table
```

Every `_s3_<name>` must match its source exactly. Do not proceed with EXCHANGE if any count diverges.

### Step 6 — Atomic swap

Use `EXCHANGE TABLES` — atomic, zero downtime:

```sql
EXCHANGE TABLES <db>.<name> AND <db>._s3_<name>
```

All swaps can be batched in one `clickhouse client` call.

### Step 7 — Verify storage policies and sizes

After the swap, confirm original-named tables report the new policy and `_s3_` tables report the old policy:

```sql
SELECT name, engine_full
FROM system.tables
WHERE database = '<db>'
ORDER BY name
```

Then do a final size/row comparison from `system.parts`:

```sql
SELECT table, sum(rows) AS row_count,
       sum(data_compressed_bytes) AS compressed_bytes,
       sum(data_uncompressed_bytes) AS uncompressed_bytes
FROM system.parts
WHERE database = '<db>' AND active = 1
GROUP BY table
ORDER BY table
```

Present side-by-side. Minor byte differences from part merging are normal — row counts must be identical.

### Step 8 — Drop backup tables

Drop the `_s3_` tables (now holding old-policy data) only after row counts and policies are verified:

```sql
DROP TABLE <db>._s3_<name>;
```

### Step 9 — (Optional) OPTIMIZE FINAL

After migration, tables have fragmented parts from the INSERT SELECT / ATTACH PARTITION process. Running `OPTIMIZE TABLE FINAL` merges them into a single part per partition, which improves query performance and reduces S3 API calls.

**Run sequentially — never in parallel.** On a memory-constrained server (e.g. 6–8 GiB limit), running OPTIMIZE FINAL on multiple large tables simultaneously will OOM. One table at a time:

```sql
OPTIMIZE TABLE <db>.<name> FINAL
```

Use `&&`-chained CLI calls (each starts only after the previous finishes) rather than background jobs in parallel.

**Monitoring cleanup:** After DROP (Step 8), old-policy parts are not removed instantly — the background cleaner runs on a schedule (`old_parts_lifetime`, default 8 min). To track progress:

```sql
-- Pending removal (active=0 = merged away, awaiting GC)
SELECT database, count() AS parts, sum(rows) AS rows,
       formatReadableSize(sum(data_compressed_bytes)) AS compressed
FROM system.parts WHERE active = 0
GROUP BY database ORDER BY sum(data_compressed_bytes) DESC

-- Recent deletions (last minute)
SELECT count() AS parts, formatReadableSize(sum(size_in_bytes)) AS size
FROM system.part_log
WHERE event_type = 'RemovePart' AND event_time >= now() - INTERVAL 1 MINUTE
```

When `active=0` parts reach 0 and `part_log` shows no recent RemovePart events, cleanup is complete.

## Safety rules

- **Never drop a table** unless you have confirmed the other copy has equal row counts.
- **Never re-INSERT** into a shadow that already has data without first checking it's safe (shadow has partial data that should be discarded, not unique data).
- **When in doubt, stop and ask the user** — describe exactly what state you found and what options exist.
- If you discover a state that doesn't fit the cases above (e.g. both tables have different non-zero row counts with no clear explanation), do not proceed — report the anomaly.

## Output at each step

Tell the user what you're about to do and confirm success at each stage. Present verification results as a markdown table so discrepancies are easy to spot.

---

## Running across many databases with subagents

When the user wants to migrate all databases at once, use one agent per database running in parallel. This keeps each database isolated — a failure in one won't affect others — and lets large databases (e.g. `crypto`) run concurrently with small ones.

### How to spawn agents

In a single message, launch one `Agent` tool call per database:

```
Agent(
  description: "Migrate <db> tables to <new_policy>",
  prompt: """
    You are a ClickHouse migration agent. Load and follow the skill at
    /Users/random/.claude/skills/clickhouse-migrate-storage-policy/SKILL.md exactly.

    Migrate all tables in database `<db>` from storage_policy='<old>' to storage_policy='<new>'.

    Rules:
    - Use MCP tool `mcp__clickhouse__run_select_query` for all read-only queries
    - Use `clickhouse client --connection prod-admin --max_execution_time=3600
      --receive_timeout=3600 --send_timeout=3600` for all write operations
    - Only migrate tables currently on '<old>' policy. Skip local, no-policy, already-migrated.
    - Follow all skill steps including interrupt recovery (Step 2 triage).
    - Do NOT drop shadow tables until row counts are confirmed to match.
  """,
  run_in_background: true
)
```

Launch all agents in the **same message** so they start simultaneously.

### Handling results

Agents that lack Bash permission will stall and return asking for it — this means the subagent sandbox didn't auto-approve Bash. In that case, handle those databases directly in the main conversation using the same skill steps, or re-run with explicit Bash permissions in the agent prompt.

When an agent completes successfully it will return a summary table. When it stalls it will describe exactly where it stopped and what it needs — use that to resume.

### Databases to skip

Always skip: `system`, `information_schema`, `INFORMATION_SCHEMA`, and any database already fully migrated. Query `system.tables` first to discover which databases have tables on the old policy before spawning agents.
