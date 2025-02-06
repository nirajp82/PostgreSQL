# PostgreSQL Autovacuum Tuning Guide

## Table of Contents
| Section | Description |
| --- | --- |
| [What is Vacuum and Autovacuum?](#what-is-vacuum-and-autovacuum) | Explanation of vacuuming and autovacuum in PostgreSQL. |
| [Why Tune Autovacuum?](#why-tune-autovacuum) | Reasons for tuning autovacuum based on workload and table size. |
| [Common Autovacuum Problems and Solutions](#common-autovacuum-problems-and-solutions) | Overview of common autovacuum problems and their solutions. |

---

## What is Vacuum and Autovacuum?
In PostgreSQL, when a row is updated or deleted, the old row version isn't immediately removed. Instead, a new version is created, leaving the old one as a "dead tuple" or "obsolete tuple." This design allows other connections to access the older version without interruption. However, these dead tuples accumulate and take up space.

**Vacuum** is the process that cleans up these dead tuples, reclaiming disk space and improving performance.  
**Autovacuum** is a background process in PostgreSQL that automatically runs vacuum on tables. It wakes up periodically (by default, every minute), checks for tables with excessive dead tuples, and triggers vacuum for those tables.

---

## Why Tune Autovacuum?
Although autovacuum is essential for database maintenance, it’s not a one-size-fits-all solution. Its efficiency depends on the workload and table size. Here are some considerations:

- **Small Database, Low Transaction Rate**: Over-aggressive vacuuming can use up resources unnecessarily, slowing down query performance.
- **Large Database, High Transaction Rate**: Insufficient vacuuming can lead to excessive bloat, slowing down queries and consuming storage.

Properly tuning autovacuum is key to maintaining optimal database performance.

---

## Common Autovacuum Problems and Solutions

### 1. Vacuum Isn't Triggered Often Enough
Autovacuum triggers based on scale factors and thresholds. Key parameters include:
- `autovacuum_vacuum_scale_factor`: The fraction of dead tuples relative to table size that triggers a vacuum.
- `autovacuum_vacuum_insert_scale_factor`: The number of inserts that triggers a vacuum.

#### Signs:
- Bloat or dead tuples growing faster than expected.
- Slow queries.

#### Solution:
Adjust scale factors based on table size and growth rate. For large tables, use lower scale factors (e.g., 0.02 or even 0.002 for very write-intensive workloads).  
Monitor the last autovacuum run using `pg_stat_user_tables`.

---

### 2. Vacuum Running Too Slowly
Vacuum may be triggered, but it might not complete fast enough, leading to accumulating bloat.

#### Signs:
- Constantly running autovacuum processes.
- Rising bloat and slow queries.

#### Solutions:
- **Disable Cost Limiting**: Vacuum has a cost limit to prevent system overload. For SSD-based databases, you can safely disable this by setting `autovacuum_cost_delay` to 0. Alternatively, increase `autovacuum_cost_limit` (e.g., 10000).
- **Increase Workers**: Increase `autovacuum_max_workers` to allow concurrent processing of multiple tables (ensure adequate CPU cores).
- **Diagnose Slowness**: Use `pg_stat_progress_vacuum` to identify bottlenecks (e.g., heap scanning or index vacuuming).
- **Optimize Heap Scanning**: Use `pg_prewarm` or increase `shared_buffers` to improve cache hit ratios.
- **Optimize Index Vacuuming**: Increase `max_parallel_maintenance_workers` for parallel index vacuuming and adjust `maintenance_work_mem` or `autovacuum_work_mem` to reduce index vacuum cycles.

---

### 3. Dead Tuples Not Being Removed After Vacuum
Sometimes, dead tuples persist even after vacuum runs. This can happen because processes still need the rows.

#### Reasons:
- **Long-running Backends**: Open transactions prevent vacuum from cleaning rows accessed by those transactions.
- **Standby Queries**: Hot standby replicas can block vacuum on the primary.
- **Unused Replication Slots**: Stalled replication slots can prevent cleanup.
- **Uncommitted Prepared Transactions**: Two-phase commit transactions may block cleanup.

#### Solutions:
- **Long-running Backends**: Identify and terminate long-running backend processes using queries on `pg_stat_activity` and `pg_terminate_backend`. Use `statement_timeout` or `log_min_duration_statement` to avoid this.
- **Standby Queries**: Balance `hot_standby_feedback` (prevents cleanup on the primary) with `vacuum_defer_cleanup_age` (delays cleanup) to reduce conflicts.
- **Unused Replication Slots**: Drop unused replication slots using `pg_drop_replication_slot()`.
- **Uncommitted Prepared Transactions**: Identify and commit or rollback prepared transactions using `pg_prepared_xact`.

---

### 4. Vacuum Terminating Itself
Vacuum can terminate if it encounters lock contention, especially with DDL commands.

#### Solution:
Minimize DDL operations during peak hours, or schedule vacuum during off-peak times.

---

### 5. Transaction ID Wraparound Autovacuum
PostgreSQL uses 32-bit transaction IDs (xids), which are limited. When the database approaches this limit, it can’t accept new transactions. To prevent this, autovacuum performs a wraparound vacuum, aggressively freezing old transaction IDs to make them visible to all future transactions, freeing up xid space.

The key parameter here is `autovacuum_freeze_max_age`. If a table isn’t vacuumed regularly, a wraparound vacuum is triggered once the transaction count exceeds `autovacuum_freeze_max_age - vacuum_freeze_min_age`.

**Note:** Wraparound vacuums are more aggressive and access more pages than regular vacuums. Proper maintenance is essential to prevent database downtime.

---

By following these guidelines and regularly tuning autovacuum based on your PostgreSQL database's workload, you can maintain a high-performance and well-maintained environment.
