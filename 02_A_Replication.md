# Mastering PostgreSQL Logical Replication: The Ultimate Deep-Dive Guide

---

## 1. Introduction to Replication in PostgreSQL

Replication means copying data from one PostgreSQL server (publisher) to another (subscriber). This can be for high availability, disaster recovery, analytics, or geo-distribution.

* **Physical replication** copies the exact bytes of data files, streaming WAL files.
* **Logical replication** streams changes at the SQL level, allowing more flexibility (e.g., replicating specific tables, replicating between different PostgreSQL versions).

Logical replication was introduced in PostgreSQL 10 and has matured since.

---

## 2. The Logical Replication Architecture: Components & Flow

![Logical Replication Architecture](https://www.postgresql.org/media/img/docs/replication-arch.svg)

**Key Components:**

* **Publisher:** Source database exposing changes.
* **Subscriber:** Target database applying changes.
* **WAL (Write Ahead Log):** Log of all changes.
* **Logical Decoding Plugin:** Converts WAL to logical change sets.
* **Replication Slot:** Ensures WAL isn’t removed before subscriber reads it.
* **Publication:** Defines tables published.
* **Subscription:** Connects subscriber to publication.

**Replication Flow:**

1. Changes logged in WAL.
2. Logical decoding converts WAL changes into logical changes.
3. Changes pushed to subscribers via replication protocol.
4. Subscriber applies changes to its tables.

---

## 3. Core Concepts with Detailed Explanation & Book Analogy

## 2. Replication in PostgreSQL: Physical vs Logical

| Feature                 | Physical Replication                    | Logical Replication                                        |
| ----------------------- | --------------------------------------- | ---------------------------------------------------------- |
| Level                   | Block-level (binary copy of data files) | Row-level (logical changes: inserts, updates, deletes)     |
| Use case                | Standby servers for failover            | Selective replication of tables, cross-version replication |
| Data copied             | Entire database cluster                 | Specific tables                                            |
| Replication direction   | One-way (primary -> standby)            | One-way, but can be configured bidirectionally             |
| Supports schema changes | No (must be identical cluster versions) | Partial support (schema changes require manual sync)       |
| Streaming               | Streaming WAL files                     | Streaming logical changes from WAL                         |

---

## 3. Key Concepts Explained (With Book Analogy)


| **Concept**                   | **What It Means**                                                                                                                  | **Book Analogy**                                                                            | **Example**                                                                                               |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| **WAL** (Write-Ahead Log)     | A sequential log of all changes (inserts, updates, deletes) made to the database, written *before* applying them to the main data. | The entire **book** containing every change, written line by line.                          | The book has 500 pages of changes recorded so far.                                                        |
| **LSN** (Log Sequence Number) | A unique identifier (like a position marker) that points to a specific place in the WAL. Example: “Checkpoint at LSN 12345.”       | A **line number and page number** in the book. E.g., Page 3, Line 45.                       | Your last read position is **Page 3, Line 45** (LSN 3:45).                                                |
| **Replication Slot**          | A persistent pointer to an LSN that marks where a replica should resume reading from the WAL.                                      | A **bookmark** placed at a specific line in the book, so you know where to resume reading.  | The bookmark is at Page 3, Line 45, so next time you start reading from **Line 46 on Page 3** (LSN 3:46). |
| **Publication**               | A defined set of tables on the **publisher** (source) database that should be replicated.                                          | Choosing which **chapters** (tables) from the book should be shared.                        | You decide to share only **Chapters 1 and 2** (Tables A and B) from the book.                             |
| **Subscription**              | A configuration on the **subscriber** (target) database that connects to a publication and fetches changes.                        | The **reader** who opens the book at the bookmark and starts reading the selected chapters. | You (the subscriber) connect to the publication and start reading from the bookmark at Page 3, Line 46.   |

---

### How the example ties in:

* You read the book (WAL) up to Page 3, Line 45.
* You place a bookmark there (Replication Slot) so you don’t lose your place.
* When you return, you start reading from Line 46 on Page 3 (the LSN after the bookmark).
* The publisher only shares selected chapters (Publication).
* You subscribe to those chapters (Subscription) and pull changes starting from your bookmark.

---

## 4. WAL Internals and Logical Decoding

* The WAL records every change before writing to data files — **write-ahead logging**.
* Physical replication streams raw WAL files.
* Logical replication uses **logical decoding**, which extracts changes at the logical row level from WAL.
* Logical decoding plugins (e.g., `pgoutput`, `test_decoding`) interpret WAL changes into readable formats.
* Logical replication streams these decoded changes to subscribers.

**WAL Segments:** PostgreSQL divides WAL into fixed-size files (default 16MB). The LSN tracks positions inside these segments.

---

## 5. Replication Slots Deep Dive

* Replication slots ensure that the WAL needed by subscribers isn’t deleted prematurely.
* Two main types: Physical and Logical replication slots.
* Logical replication slots hold the LSN where a subscriber last received changes.
* Slots persist until dropped.
* Excessive retention of WAL due to lagging slots causes disk bloat.

**Managing Slots:**

```sql
-- View all replication slots:
SELECT * FROM pg_replication_slots;

-- Drop a replication slot if no longer needed:
SELECT pg_drop_replication_slot('slot_name');

-- Monitor replication lag and slot status:
SELECT * FROM pg_stat_replication;
```

---

## 6. Publication & Subscription Mechanics

### Publication

* Created on publisher.
* Can include all tables or a subset.
* Syntax example:

```sql
CREATE PUBLICATION mypub FOR TABLE table1, table2;
-- Or publish all tables:
CREATE PUBLICATION mypub FOR ALL TABLES;
```

### Subscription

* Created on subscriber.
* Connects to publication and starts replication.
* Can specify whether to copy initial data or not.
* Syntax example:

```sql
CREATE SUBSCRIPTION mysub
CONNECTION 'host=publisher_host port=5432 dbname=mydb user=replicator password=secret'
PUBLICATION mypub
WITH (copy_data = true);  -- defaults to true
```

---

## 7. Setting up Logical Replication: Full Configuration Guide

### 7.1 Configure Publisher (`postgresql.conf`)

```conf
wal_level = logical
max_wal_senders = 10
max_replication_slots = 10
max_worker_processes = 10
```

Add replication permissions in `pg_hba.conf`:

```
host replication replicator 192.168.1.0/24 md5
```

Reload config:

```sql
SELECT pg_reload_conf();
```

---

### 7.2 Create Publication (Publisher Side)

```sql
CREATE PUBLICATION my_publication FOR TABLE employees, departments;
```

---

### 7.3 Create Subscriber Tables

Ensure tables exist on subscriber with matching schemas:

```sql
CREATE TABLE employees (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  department_id INT NOT NULL
);

CREATE TABLE departments (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL
);
```

---

### 7.4 Create Subscription (Subscriber Side)

```sql
CREATE SUBSCRIPTION my_subscription
CONNECTION 'host=publisher_host port=5432 dbname=mydb user=replicator password=secret'
PUBLICATION my_publication;
```

---

## 8. Initial Data Synchronization & Ongoing Replication

* When a subscription is created, an initial snapshot copy of the published tables is performed automatically.
* After that, ongoing data changes are streamed continuously.
* You can disable initial snapshot copy if you want to do it manually:

```sql
CREATE SUBSCRIPTION my_subscription
CONNECTION 'conninfo'
PUBLICATION my_publication
WITH (copy_data = false);
```

---

## 9. Schema Changes and Logical Replication

* Logical replication **does not replicate DDL changes** (e.g., `ALTER TABLE`).
* Schema changes must be applied manually on the subscriber.
* Avoid incompatible schema changes during replication.

---

## 10. Monitoring & Performance Tuning

### Useful Queries

```sql
-- View replication slots:
SELECT * FROM pg_replication_slots;

-- View replication lag and status:
SELECT pid, application_name, state, sent_lsn, write_lsn, flush_lsn, replay_lsn FROM pg_stat_replication;

-- Check subscription status:
SELECT * FROM pg_stat_subscription;
```

### Tune These Parameters as Needed

* `wal_sender_timeout`
* `wal_receiver_status_interval`
* `max_replication_slots`
* `max_wal_senders`

---

## 11. Troubleshooting Common Issues

| Issue                       | Description                              | Troubleshooting Steps                              |
| --------------------------- | ---------------------------------------- | -------------------------------------------------- |
| Replication lag             | Subscriber behind publisher changes      | Check network, load, and subscriber queries        |
| Replication slot bloat      | WAL files piling up due to inactive slot | Drop or fix lagging replication slots              |
| Schema mismatch errors      | Subscription errors on apply             | Synchronize schemas manually                       |
| Subscription not connecting | Authentication or network issues         | Verify `pg_hba.conf`, connection strings, firewall |
| Conflicts in bi-directional | Conflicting changes in both nodes        | Use conflict resolution or avoid conflicts         |

---

## 12. Advanced Use Cases & Bidirectional Replication

* Two-way replication between nodes (bidirectional) requires manual conflict resolution.
* Useful for multi-master or geo-distributed setups.
* Logical replication can replicate between different PostgreSQL versions.
* Future work includes row filtering and column filtering.

---

## 13. Security & Permissions

* Use dedicated replication user with `REPLICATION` privilege:

```sql
CREATE USER replicator WITH REPLICATION PASSWORD 'secret';
```

* Use SSL for replication connections.
* Restrict connections by IP in `pg_hba.conf`.

---

## 14. Limitations & Gotchas

* No automatic schema (DDL) replication.
* Large objects (BLOBs) aren’t replicated.
* Slow subscribers cause WAL retention (disk bloat).
* Sequence values are not replicated automatically.
* Schema must match on subscriber.

---

## 15. Real-world Best Practices

* Monitor replication lag, slots, and disk usage regularly.
* Automate schema synchronization scripts.
* Use SSL connections.
* Clean up unused replication slots.
* Avoid writes on subscriber in most cases.
* Start small with selective publications.

---

## 16. Summary & Further Learning

Logical replication allows selective, row-level replication with lots of flexibility. It’s powerful but requires careful setup and management.

---

# **Bonus: Complete Example Setup Script**

You can use this step-by-step example on both publisher and subscriber to test logical replication:

---

### On Publisher

```sql
-- Enable replication-friendly settings in postgresql.conf:
-- wal_level = logical
-- max_wal_senders = 10
-- max_replication_slots = 10

-- Reload config:
SELECT pg_reload_conf();

-- Create replication user:
CREATE ROLE replicator WITH LOGIN REPLICATION PASSWORD 'replicator_pass';

-- Create tables:
CREATE TABLE employees (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  department_id INT NOT NULL
);

CREATE TABLE departments (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL
);

-- Create publication:
CREATE PUBLICATION my_publication FOR TABLE employees, departments;
```

---

### On Subscriber

```sql
-- Create tables with matching schemas:
CREATE TABLE employees (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  department_id INT NOT NULL
);

CREATE TABLE departments (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL
);

-- Create subscription:
CREATE SUBSCRIPTION my_subscription
CONNECTION 'host=publisher_host port=5432 dbname=publisher_db user=replicator password=replicator_pass'
PUBLICATION my_publication;
```

---

**Now:**

* Insert some data into `employees` or `departments` on the publisher.
* Changes will replicate automatically to the subscriber.
