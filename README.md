# PostgreSQL 9.6 to 17 Migration using pglogical

This repository contains a **step-by-step** guide and **scripts** for migrating a large production database from **PostgreSQL 9.6** to **PostgreSQL 17** with **minimal downtime** using **pglogical** (logical replication). We also demonstrate how to **pre-copy** large tables, handle **foreign key constraints**, and ensure a smooth cutover.

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Migration Steps](#migration-steps)
  - [1. Prepare Old PostgreSQL 9.6 Server](#1-prepare-old-postgresql-96-server)
  - [2. Prepare New PostgreSQL 17 Server](#2-prepare-new-postgresql-17-server)
  - [3. Schema-Only Dump & Restore](#3-schema-only-dump--restore)
  - [4. Pre-Copy Large Tables](#4-pre-copy-large-tables)
  - [5. Configure pglogical (Provider Side)](#5-configure-pglogical-provider-side)
  - [6. Configure pglogical (Subscriber Side)](#6-configure-pglogical-subscriber-side)
  - [7. Create the Subscription](#7-create-the-subscription)
  - [8. Monitor & Final Cutover](#8-monitor--final-cutover)
- [Challenges & Lessons Learned](#challenges--lessons-learned)

---

## Overview

- **Goal**: Migrate from `PostgreSQL 9.6` to `PostgreSQL 17` on a new OS (e.g., Ubuntu 24.04) with near-zero downtime.  
- **Method**: `pglogical` logical replication, which lets us replicate data across major versions.  
- **Why**: We want to keep the old database online while transferring data, then do a fast switchover.

### Key Concepts

- **Logical Replication** (pglogical): Allows different major versions of Postgres to replicate data in real-time.  
- **Pre-Copy**: For very large tables, we manually dump/restore them to avoid re-copying on every failure.  
- **Minimal Downtime**: Most of the data transfer happens while the old DB is still live; we only take a brief downtime window at the end.

---

## Prerequisites

1- PostgreSQL 9.6 (old server).
2- PostgreSQL 17 (new server).
3- pglogical installed on both (build from source if needed for 9.6).
4- Enough wal_keep_size or WAL archiving on the old server to sustain a multi-hour initial copy.

---

## Migration Steps

### 1. Prepare Old PostgreSQL 9.6 Server
  - Install pglogical (via source or PGDG repo if available).
  - In postgresql.conf, set:
  ```bash
wal_level = logical
max_wal_senders = 10
max_replication_slots = 10
```

  - In pg_hba.conf, allow the new server’s IP to connect:
  ```bash
host    replication     repuser 10.10.10.60/32 md5
host    all            repuser 10.10.10.60/32 md5
```

  - Create a replication user:
  ```bash
CREATE ROLE repuser WITH LOGIN SUPERUSER PASSWORD 'secret_password';
```

  - Restart PostgreSQL 9.6 to apply changes.

### 2. Prepare New PostgreSQL 17 Server
  - Install PostgreSQL 17 + pglogical plugin.
  - Create a database named acme_db (or your DB name).
  - Confirm you can connect with psql.

### 3. Schema-Only Dump & Restore
  - Dump the schema (no data) from old server:
    ```bash
    pg_dump -h 10.10.10.50 -U repuser -s -d acme_db > schema_only.sql
    ```
  - Restore to new server:
    ```bash
    psql -h 10.10.10.60 -U repuser -d acme_db -f schema_only.sql
    ```

### 4. Pre-Copy Large Tables
   For any tables in the tens-of-GB range, do a data-only dump and restore, e.g.:
   ```bash
   # Dump table "orders" on old server
    pg_dump -h 10.10.10.50 -U repuser -d acme_db --data-only -t public.orders > orders_data.sql

    # Restore on new server
    psql -h 10.10.10.60 -U repuser -d acme_db -f orders_data.sql
   ```
  If you have foreign key dependencies, load parent tables first. Once these large tables are present on the new server, you can skip the initial bulk copy in pglogical (see below).

### 5. Configure pglogical (Provider Side)
   On old server (Postgres 9.6), in DB acme_db:

   ```bash
   CREATE EXTENSION pglogical;

    SELECT pglogical.create_node(
    node_name := 'provider_node',
    dsn       := 'host=10.10.10.50 dbname=acme_db user=repuser password=secret_password'
    );
   ```
  Now decide which tables belong to the replication set:
  ```bash
  -- Add all tables to 'default' set, then exclude large ones
SELECT pglogical.replication_set_add_all_tables(
  set_name := 'default',
  schema_names := ARRAY['public'],
  synchronize_data := true
);

-- Remove the big pre-copied tables
SELECT pglogical.replication_set_remove_table('default','public.orders');
...

-- Re-add them with synchronize_data=false
SELECT pglogical.replication_set_add_table(
  'default',
  'public.orders',
  synchronize_data := false
);
...
```
### 6. Configure pglogical (Subscriber Side)
  On the new server (Postgres 17):
  ```bash
  CREATE EXTENSION pglogical;

SELECT pglogical.create_node(
    node_name := 'subscriber_node',
    dsn       := 'host=10.10.10.60 dbname=acme_db user=repuser password=secret_password'
);
```

### 7. Create the Subscription
  ```bash
  SELECT pglogical.create_subscription(
  subscription_name := 'acme_subscription',
  provider_dsn      := 'host=10.10.10.50 dbname=acme_db user=repuser password=secret_password',
  replication_sets  := ARRAY['default'],
  synchronize_data  := true
);
```
  - Smaller tables are fully copied.
  - Large, pre-copied tables just replicate changes (no big initial transfer).

### 8. Monitor & Final Cutover
  - Check status:
    ```bash
    SELECT * FROM pglogical.show_subscription_status();
    ```
    It should go from initializing to replicating.

  - Watch logs on both servers for errors (FK constraints, missing tables, WAL issues).
  - Final cutover:
      1. Stop writes on the old server (maintenance mode or shut down the app).
      2. Wait a few seconds for last WAL to apply.
      3. Point your application to the new server (10.10.10.60).
      4. Done! You’re on PostgreSQL 17 now.

---

## Challenges & Lessons Learned

1. Foreign Key Failures: Ensure parent tables are loaded first or disable constraints temporarily.
2. “Relation Already Exists”: Use --data-only if the schema is already present on the new server.
3. Large Table Performance: Consider pg_dump -F d -j 4 (directory + parallel) to speed up huge table loads.
4. WAL Retention: Old server must keep WAL long enough for the entire sync (possibly hours).
5. Subscription Goes Down: Usually means a mismatch or error (FK constraints, missing table, etc.). Fix cause, then re-enable.
