# PostgreSQL 9.6 → 17 Migration with pglogical

This document explains how we migrated from **PostgreSQL 9.6** to **PostgreSQL 17** with **minimal downtime** using **pglogical** (logical replication). It covers:

1. [Overview](#overview)
2. [Prerequisites & Setup](#prerequisites--setup)
3. [Migration Strategy](#migration-strategy)
4. [Step-by-Step Process](#step-by-step-process)
   - [4.1. Dump & Restore Schema](#41-dump--restore-schema)
   - [4.2. (Optional) Pre-copy Large Tables](#42-optional-pre-copy-large-tables)
   - [4.3. Configure pglogical (Provider Node)](#43-configure-pglogical-provider-node)
   - [4.4. Configure pglogical (Subscriber Node)](#44-configure-pglogical-subscriber-node)
   - [4.5. Create Subscription](#45-create-subscription)
   - [4.6. Cutover](#46-cutover)
5. [Sequences After Cutover](#sequences-after-cutover)
6. [Common Pitfalls](#common-pitfalls)
7. [Conclusion](#conclusion)
8. [References](#references)

---

## Overview

- **Old Server**: PostgreSQL **9.6** (EOL), installed around 2017, IP `10.10.10.50`, Database `acme_db`.
- **New Server**: **Ubuntu 24.04** running PostgreSQL **17**, IP `10.10.10.60`, Database `acme_db`.
- **Goal**: Near-zero downtime upgrade to a fully supported PostgreSQL version on a brand-new OS.

**Why pglogical?**

- Logical replication works **across major versions** (9.6 → 17).
- Allows continuous data synchronization, drastically **reducing downtime**.
- We needed a **new machine**, not an in-place upgrade, so streaming replication or pg_upgrade wasn’t ideal.

---

## Prerequisites & Setup

1. On the **old server**:
   - **Install pglogical** (compile from source if not in repos).
   - Edit `postgresql.conf`:
     ```conf
     wal_level = logical
     max_wal_senders = 10
     max_replication_slots = 10
     ```
   - Update `pg_hba.conf` to allow connections from new server’s IP (`10.10.10.60`):
     ```
     host    replication repuser 10.10.10.60/32 md5
     host    all         repuser 10.10.10.60/32 md5
     ```
   - Create a **replication user** with superuser:
     ```sql
     CREATE ROLE repuser WITH LOGIN SUPERUSER PASSWORD 'secret';
     ```
   - Restart PostgreSQL 9.6 to apply changes.

2. On the **new server** (Ubuntu 24.04):
   - Install **PostgreSQL 17** + `postgresql-17-pglogical`.
   - Create database `acme_db`.
   - Confirm you can connect via `psql`.

---

## Migration Strategy

1. **Dump & restore the schema** (no data) so both databases share the same structure.
2. **(Optional) Pre-copy** large tables manually to avoid re-copying if an error occurs during initial sync.
3. **Use pglogical** to replicate the remaining tables, so changes stream continuously to the new server.
4. **Cut over** after verifying data sync and minimal downtime.

---

## Step-by-Step Process

### 4.1. Dump & Restore Schema

First, **schema-only dump** on the old server, then restore on the new server:

```bash
# On old server (9.6)
pg_dump -h 10.10.10.50 -U repuser -s -d acme_db > acme_schema.sql

# On new server (17)
psql -h 10.10.10.60 -U repuser -d acme_db -f acme_schema.sql
```
This ensures all tables, indexes, and constraints match in both environments.

### 4.2. (Optional) Pre-copy Large Tables

If you have very large tables (tens of GB or more), consider dumping & restoring them manually:
```bash
# On old server
pg_dump -h 10.10.10.50 -U repuser -d acme_db --data-only -t public.large_table > large_table_data.sql

# Copy to new server, then restore:
psql -h 10.10.10.60 -U repuser -d acme_db -f large_table_data.sql
```
Later, you’ll add these tables to the replication set with synchronize_data = false, so pglogical will not do a big initial copy.

### 4.3. Configure pglogical (Provider Node)

On the old server (9.6), in acme_db:

```bash
CREATE EXTENSION pglogical;

SELECT pglogical.create_node(
    node_name := 'acme_provider',
    dsn       := 'host=10.10.10.50 dbname=acme_db user=repuser password=secret'
);
```
Add tables to your replication set:
```bash
-- Add all tables:
SELECT pglogical.replication_set_add_all_tables(
  set_name := 'default',
  schema_names := ARRAY['public'],
  synchronize_data := true
);

-- If you've pre-copied "large_table", remove it from auto-sync:
SELECT pglogical.replication_set_remove_table('default', 'public.large_table');

-- Re-add it with no initial copy:
SELECT pglogical.replication_set_add_table(
  'default',
  'public.large_table',
  synchronize_data := false
);
```
Repeat for any other large tables you’ve manually restored.

### 4.4. Configure pglogical (Subscriber Node)
On the new server (17), in acme_db:
```bash
CREATE EXTENSION pglogical;

SELECT pglogical.create_node(
    node_name := 'acme_subscriber',
    dsn       := 'host=10.10.10.60 dbname=acme_db user=repuser password=secret'
);
```

### 4.5. Create Subscription
Create the subscription on the new server, telling it to pull from the old server:
```bash
SELECT pglogical.create_subscription(
    subscription_name := 'acme_subscription',
    provider_dsn      := 'host=10.10.10.50 dbname=acme_db user=repuser password=secret',
    replication_sets  := ARRAY['default'],
    synchronize_data  := true
);
```
Now, smaller tables do a full initial copy, while any table added with synchronize_data = false only gets incremental changes.

Monitor:
```bash
SELECT * FROM pglogical.show_subscription_status();
```
Look for status = 'replicating'. If it’s down, check logs on both servers for foreign key issues, missing tables, or WAL retention problems.

### 4.6. Cutover
Once fully synchronized:

Stop writes on old server (e.g., put the app in maintenance mode).
Wait a few seconds for the last WAL to replicate.
Point your application to the new server (10.10.10.60).
You’re now running on PostgreSQL 17 with minimal downtime!

---

## Sequences After Cutover

A common issue is that sequence values might be out of sync with the table’s current MAX(id). If your newly loaded table’s highest id is 100,000 but the sequence’s internal counter is only 5,000, you’ll get duplicate key errors.

Solution:
You can find all tables and columns that depend on sequences with below query:
```bash
SELECT
    t.relname AS table_name,
    a.attname AS column_name,
    s.relname AS sequence_name
FROM pg_class s
JOIN pg_depend d  ON d.objid = s.oid
JOIN pg_class t   ON d.refobjid = t.oid
JOIN pg_attribute a
    ON a.attrelid = t.oid
    AND d.refobjsubid = a.attnum
WHERE s.relkind = 'S'
  AND t.relkind IN ('r','p')
  AND d.deptype = 'a'
ORDER BY t.relname;
```
You can set sequences with below query:
```bash
SELECT setval('mytable_id_seq', (SELECT COALESCE(MAX(id), 0) FROM mytable) + 1, false);
```
Run a query like this for each table that uses a serial/bigserial column. Or use a plpgsql block to loop through all sequences automatically.

If tables are too much you can use below script:
```bash
DO $$
DECLARE
    rec record;
    current_max bigint;
BEGIN
    FOR rec IN
        -- This query finds every (table_name, column_name, sequence_name) triple
        -- where the sequence is "owned" by that column (typical of SERIAL/BIGSERIAL).
        SELECT t.relname AS table_name,
               a.attname AS column_name,
               s.relname AS sequence_name
          FROM pg_class s
          JOIN pg_depend d      ON d.objid = s.oid
          JOIN pg_class t       ON d.refobjid = t.oid
          JOIN pg_attribute a   ON a.attrelid = t.oid AND d.refobjsubid = a.attnum
         WHERE s.relkind = 'S'               -- S = sequence
           AND t.relkind IN ('r', 'p')       -- r = ordinary table, p = partitioned table
           AND a.attname = 'id'             -- only fix sequences for "id" columns (adjust if needed)
           AND d.deptype = 'a'              -- 'a' = automatic dependency (SERIAL/IDENTITY)
         ORDER BY t.relname
    LOOP
        -- Get the current max(id) from the table
        EXECUTE format('SELECT max(%I)::bigint FROM %I', rec.column_name, rec.table_name)
          INTO current_max;

        IF current_max IS NULL THEN
            current_max := 0;
        END IF;

        -- Bump the sequence to max(id)+1 so the next insert doesn't conflict
        EXECUTE format('SELECT setval(%L, %s, false)', rec.sequence_name, current_max + 1);

        RAISE NOTICE 'Fixed sequence for table: %, column: %, sequence: %, set to %',
                     rec.table_name, rec.column_name, rec.sequence_name, current_max + 1;
    END LOOP;
END $$;
```


---

## Common Pitfalls

1. Foreign Key Failures: Load parent tables first or disable constraints if you see FK errors during initial copy.
2. WAL Retention: Ensure wal_keep_size or archiving is enough to keep all logs until the subscriber finishes syncing (especially for large data sets).
3. “Relation Already Exists”: If the table is already defined, use --data-only to avoid DDL conflicts.
4. Subscription Down: Usually indicates mismatched schemas, missing indexes, or insufficient privileges. Check server logs carefully.

---

## Conclusion

By following these steps, we achieved a near-zero-downtime migration from a PostgreSQL 9.6 server (installed around 2017) to a PostgreSQL 17 instance on a modern Ubuntu 24.04 environment. We pre-copied large tables, set up logical replication for the rest, handled sequences post-cutover, and avoided extended service disruption.

### Key Takeaways:

- Carefully plan the order of data loading if foreign keys are involved.
- Pre-copy large tables to reduce restart overhead.
- Remember to bump sequence values to avoid insert conflicts.
- Monitor logs on both servers to troubleshoot quickly.
This ensures you can retire an old EOL database version, reduce technical debt, and move to a fully supported, more performant environment.

