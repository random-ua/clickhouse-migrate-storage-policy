# clickhouse-migrate-storage-policy

A Claude Code skill that migrates all tables in a ClickHouse database to a new storage policy with zero downtime.

## What it does

1. Reads all table DDLs from the target database
2. Creates shadow tables (`_s3_<name>`) with the new storage policy
3. Copies data via `INSERT INTO … SELECT *` (sequentially, one table at a time)
4. Verifies row counts match before touching the originals
5. Atomically swaps original and shadow tables with `EXCHANGE TABLES`
6. Confirms policies and sizes, then drops the shadow tables

## Usage

Trigger the skill by asking Claude something like:
- *"Migrate all tables in `my_db` to storage policy `s3_2`"*
- *"Move tables in `analytics` to new S3 storage"*
- *"Change storage policy for all tables in `logs` to `cold`"*
