# PostgreSQL 9.6 to 17 Migration using pglogical

This repository contains a **step-by-step** guide and **scripts** for migrating a large production database from **PostgreSQL 9.6** to **PostgreSQL 17** with **minimal downtime** using **pglogical** (logical replication). We also demonstrate how to **pre-copy** large tables, handle **foreign key constraints**, and ensure a smooth cutover.

---

## Table of Contents

- [Overview](#overview)
- [Repository Structure](#repository-structure)
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
- [References](#references)

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

## Repository Structure

```bash
my-postgres-migration/
├── README.md                  # You are here!
├── docs/
│   └── migration_guide.md     # More detailed or step-by-step docs
├── scripts/
│   ├── 01_create_schema_dump.sh      # Example shell script to dump schema
│   ├── 02_restore_schema.sh          # Example shell script to restore schema
│   ├── 03_pre_copy_large_tables.sh   # Example shell script for large table dumps
│   ├── 04_pglogical_setup_provider.sql
│   ├── 05_pglogical_setup_subscriber.sql
│   ├── 06_create_subscription.sql
│   └── ...
└── LICENSE
