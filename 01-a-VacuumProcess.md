### Optimizing Autovacuum: PostgreSQL's Vacuum Process

In PostgreSQL, inserting data into a table should not normally create dead tuples immediately. However, under certain conditions, an INSERT can indirectly contribute to dead tuples. Let’s break it down:

#### Understanding Dead Tuples

Dead tuples in PostgreSQL are row versions that are no longer visible to any active transaction. They occur because PostgreSQL uses MVCC (Multi-Version Concurrency Control), meaning that instead of modifying a row in place, PostgreSQL creates a new version of the row while marking the old version as “dead” when an update or delete happens.

#### What is Vacuum and Autovacuum?
In PostgreSQL, when you **update** or **delete** a row, the old version of that row isn't immediately removed. Instead, a new version is created, leaving the old one as a **"dead tuple"** or **obsolete tuple**. This approach allows other database connections to continue accessing the older version without interruption. However, over time, these dead tuples accumulate and consume disk space.

The **vacuum** process cleans up these dead tuples, reclaiming disk space and improving performance. You can run vacuum manually, but PostgreSQL also provides **autovacuum**, which is a background process that automatically triggers vacuuming on tables when necessary. Autovacuum runs periodically (default is every minute), checks which tables need vacuuming based on the number of dead tuples and inserts, and initiates vacuum processes for those tables.

#### Why Tune Autovacuum?
**Purpose**: The job of autovacuum is to trigger the vacuum process automatically to vaccume different tables.

While autovacuum is essential, it isn't a one-size-fits-all solution. It requires proper tuning to work effectively. The optimal settings for autovacuum depend on the workload and size of the database. 

For example:
- **Small Database, Low Transaction Rate**: Overly aggressive vacuuming can consume excessive resources, negatively impacting query performance.
- **Large Database, High Transaction Rate**: If autovacuum isn't aggressive enough, it can lead to excessive bloat, consuming storage and slowing down queries.

Thus, tuning autovacuum correctly is critical for maintaining optimal database performance and avoiding problems like performance degradation or excessive storage usage.
- **`autovacuum_naptime`**
The `autovacuum_naptime` parameter in PostgreSQL controls how often the autovacuum process wakes up and checks if it needs to run on any tables. 
    - **Default value**: 1 minute (60 seconds).
        - **Purpose**: It sets the time between two consecutive autovacuum checks. Every time the process wakes up, it checks if any tables have enough dead tuples to require vacuuming. If not, it goes back to sleep until the next interval.
        - If you set it too short, it will check more often, using more CPU resources. If set too long, it may delay cleaning up bloat, leading to slower performance.
        - The default value is 15 seconds for Amazon RDS for PostgreSQL and 5 seconds for Aurora PostgreSQL-Compatible.

- **Note::** - By default its On, Do not turn off until and unless you know what you are doing.

### Common Autovacuum Problems and Solutions with Queries

---

#### 1. **Vacuum Isn't Triggered Often Enough**
Autovacuum triggers vacuum based on scale factors and thresholds. The key parameters are:

![image](https://github.com/user-attachments/assets/019f7f52-a50d-47d0-a852-4eb484cc34df)

**Key Parameters**:
#### 1. **`autovacuum_vacuum_scale_factor`**
   - **Purpose**: This setting controls the threshold at which autovacuum will run a vacuum operation for **tables** that have been updated or deleted.
   - **How it works**: The value represents a fraction of the total number of rows in the table. Once the number of dead tuples (i.e., rows that have been updated or deleted but not yet cleaned up) exceeds the specified fraction of the total rows, autovacuum will trigger a vacuum.
   - **Default value**: `0.2` (which means 20% of the table's rows)
   - **Example**: 
     - If you have a table with 1 million rows, and `autovacuum_vacuum_scale_factor` is set to 0.2, the autovacuum will trigger when there are 200,000 dead tuples in the table.

#### 2. **`autovacuum_vacuum_insert_scale_factor`**
   - **Purpose**: This setting controls when autovacuum will trigger a vacuum operation for **inserts**. It is similar to `autovacuum_vacuum_scale_factor`, but specifically for tables that primarily experience insert operations.
   - **How it works**: The value represents a fraction of the number of inserted rows. When the number of dead tuples from insert operations exceeds the specified fraction, autovacuum will be triggered.
   - **Default value**: `0.2` (same as the vacuum scale factor)
   - **Example**: 
     - If you have a table with 1 million rows and 200,000 rows have been inserted, and the `autovacuum_vacuum_insert_scale_factor` is 0.2, autovacuum will be triggered once there are 40,000 dead tuples from those insert operations.
   - How Can INSERT Cause Dead Tuples?  Even though INSERT does not modify existing tuples, it can lead to dead tuples in the following ways:

		1. Rollback of a Transaction
			•	When you insert data inside a transaction but later ROLLBACK, all inserted rows become dead tuples since they are no longer visible to any transaction.
			•	Example:
		```sql
		BEGIN;
		INSERT INTO users (id, name) VALUES (1, 'John');
		ROLLBACK;  -- The inserted row becomes a dead tuple
		```
			•	The space is not immediately reclaimed until autovacuum cleans it up.
		
		2. Insert Conflict in ON CONFLICT DO UPDATE
			•	If you use INSERT ... ON CONFLICT DO UPDATE, PostgreSQL first attempts an insert but may update an existing row instead. The old version of the updated row then becomes a dead tuple.
			•	Example:
		```sql
		INSERT INTO users (id, name) 
		VALUES (1, 'Alice') 
		ON CONFLICT (id) DO UPDATE 
		SET name = EXCLUDED.name;
		```
		
			•	If a row with id = 1 already exists, PostgreSQL will update it, creating a dead tuple of the previous version of that row.
		
		3. Insert Followed by DELETE in the Same Transaction
			•	If you insert a row and then delete it within the same transaction, the inserted row becomes a dead tuple because it is no longer needed.
			•	Example:
		```sql
		BEGIN;
		INSERT INTO users (id, name) VALUES (2, 'Bob');
		DELETE FROM users WHERE id = 2;
		COMMIT;  -- The inserted row becomes dead
		```
		
		4. High Insert/Delete Workloads Without Vacuuming
			•	In workloads where a table gets frequent INSERTS followed by DELETEs, dead tuples accumulate quickly unless autovacuum runs efficiently.
			•	Example:
		
		```sql
		INSERT INTO logs (event, created_at) VALUES ('login_failed', now());
		DELETE FROM logs WHERE created_at < now() - interval '1 day';
		```
			•	If many inserts and deletes happen rapidly, and autovacuum lags, dead tuples increase.
		
		How to Handle Dead Tuples from Inserts?
			•	Ensure autovacuum is running (SELECT * FROM pg_stat_all_tables WHERE relname = 'your_table' to check).
			•	Manually run VACUUM ANALYZE your_table; if needed.
			•	Use ON CONFLICT DO NOTHING instead of ON CONFLICT DO UPDATE where possible.
			•	Partition large insert/delete tables to reduce dead tuples.
			•	Use fillfactor settings to allow space for updates without immediate tuple invalidation. 
	
**Signs of the problem**: If dead tuples accumulate faster than expected, leading to bloat (more disk space than necessary - because it has too many dead tuples (old, unused versions of rows) that haven't been cleaned up) and slower queries.

**Solution**: Adjust the scale factors based on table size and growth rate. For large tables or very write-intensive workloads, use lower scale factors (e.g., instead of 0.02, which triggers vacuum when 20% of 1 billion rows is dead data, set it to something much lower like 0.002). Monitor the last autovacuum run time using the `pg_stat_user_tables` view.

**Query to check the last autovacuum run**:
```sql
SELECT
    relname,  -- Name of the table (relation).
    last_vacuum,  -- Timestamp of the last manual vacuum performed on the table.
    last_autovacuum,  -- Timestamp of the last autovacuum run on the table.
    autovacuum_count,  -- Number of times autovacuum has run on this table.
    vacuum_count,  -- Number of times manual vacuum has run on this table.
    n_dead_tup,  -- Number of dead tuples (obsolete rows) in the table.
    n_live_tup  -- Number of live (active) tuples (rows) in the table.
FROM
    pg_stat_user_tables;  -- View showing statistics for user tables.
```
This query shows the last time vacuum and autovacuum were run on each table, as well as the number of dead tuples. If `last_autovacuum` is too old, it may indicate that autovacuum isn’t running often enough.

---

#### 2. **Vacuum Running Too Slowly**
If autovacuum is running too slowly, it might not be completing fast enough to keep up with the workload, leading to bloat.

**Signs of the problem**: Constantly running autovacuum processes, rising bloat, and slow queries.

**Solution**:
##### Optimizing PostgreSQL Vacuum: Disabling Cost Limiting and Increasing Workers

- **Disable Cost Limiting**

    PostgreSQL’s `VACUUM` has a built-in **cost limit** to prevent it from consuming too many resources and overwhelming the system. For databases running on **SSDs**, this cost limiting is often unnecessary due to faster I/O speeds. In such cases, you can **disable** the cost limit by setting:

    - **`autovacuum_cost_delay` to 0** (No nap time for `VACUUM`).
  
    Alternatively, you can **increase** the `autovacuum_cost_limit` to a higher value (e.g., **10000**) to allow `VACUUM` to use more resources before taking breaks.

-  **Increase Workers**

    To make the autovacuum process more efficient, especially on large databases, you can **increase** the number of worker processes available. This allows autovacuum to process multiple tables concurrently, reducing the overall time for cleanup.

    - **`autovacuum_max_workers`**: Increase this value to allow more concurrent vacuum processes.
  
    **Note:** Ensure that your system has sufficient **CPU cores** to handle the increased number of workers without degrading overall system performance.

- **Diagnose Slowness**  
  Use the `pg_stat_progress_vacuum` view to identify where `VACUUM` is slow. It will show whether the process is stalled at **heap scanning** (data rows) or **index vacuuming**, helping you pinpoint the bottleneck.

    - **Optimize Heap Scanning**  
      To improve heap scanning performance, increase **shared_buffers** to allocate more memory for caching, or use **pg_prewarm** to load tables and indexes into memory for faster access. This reduces disk I/O, speeding up vacuuming.
    
    - **Optimize Index Vacuuming**  
      For better index vacuuming, increase the **max_parallel_maintenance_workers** parameter to enable parallel vacuuming of indexes. You can also increase **maintenance_work_mem** or **autovacuum_work_mem** to provide more memory, reducing the number of cycles needed for index cleanup.


**Queries to help diagnose slowness**:

- **Check autovacuum worker status**:
```sql
SELECT * FROM pg_stat_activity
WHERE backend_type = 'autovacuum worker';
```
This query will show you all autovacuum workers that are currently running. If you see too many, it might indicate that vacuuming is slow and needs tuning.

- **Monitor progress of running vacuum**:
```sql
SELECT
    relid::regclass AS table_name,  -- The name of the table being vacuumed
    phase,                        -- The current phase of the vacuum process (e.g., 'scan heap', 'vacuum heap', 'index vacuum')
    heap_blks_total,              -- The total number of heap blocks in the table. (PostgreSQL stores table data on disk in files. These files are divided into fixed-size pages, and these pages are what we call heap blocks._
    heap_blks_scanned,            -- The number of heap blocks scanned so far in the current phase
    heap_blks_vacuumed             -- The number of heap blocks vacuumed so far in the current phase
FROM
    pg_stat_progress_vacuum
--WHERE
  --  phase = 'scan heap' OR phase = 'vacuum heap'; -- Filter to show progress for the heap scanning and vacuuming phases
```
This query shows the current status of vacuuming processes, including how much of the table has been processed and the current phase (heap scan or vacuum).

- **Check if vacuum is slow due to cost delay**:
```sql
SHOW autovacuum_cost_delay;
SHOW autovacuum_cost_limit;
```
If `autovacuum_cost_delay` is set to a non-zero value, you may want to consider reducing it or setting it to 0 for faster vacuuming, especially on SSDs.

---

#### 3. **Dead Tuples Not Being Removed After Vacuum**
Sometimes, even after vacuum runs, dead tuples remain. This happens when other processes still need those rows, such as long-running transactions or replication conflicts.

- **Reasons for Autovacuum Issues:**
    - **Long-running Backends**: Old, open transactions can prevent vacuum from cleaning up rows accessed by those transactions.
    - **Standby Queries**: Queries running on a hot standby replica can prevent vacuum from cleaning up rows on the primary.
    - **Unused Replication Slots**: Stalled or unused replication slots (both physical and logical) can hold back cleanup, especially of catalog tables.
    - **Uncommitted Prepared Transactions**: Uncommitted two-phase commit transactions can also block cleanup.

- **Solutions for Autovacuum Issues:**
    - **Long-running Backends**:  
          - Identify and terminate long-running backend processes using queries against `pg_stat_activity` and `pg_terminate_backend()`.
          - Use `statement_timeout` or `log_min_duration_statement` to prevent this.

    - **Standby Queries**:  
          - Balance `hot_standby_feedback` (which prevents cleanup on the primary) with `vacuum_defer_cleanup_age` (which delays cleanup) to minimize replication conflicts.

    - **Unused Replication Slots**:  
          - Drop unused replication slots using `pg_drop_replication_slot()`.

    - **Uncommitted Prepared Transactions**:  
          - Identify and commit or rollback prepared transactions using `pg_prepared_xact`.

**Queries to help troubleshoot dead tuples**:

- **Check for long-running backend**:
```sqlSELECT 
    pid,                                -- Process ID of the backend process
    datname,                             -- Name of the database the backend is connected to
    usename,                             -- Username of the session running the query
    application_name,
	state,                               -- Current state of the backend process (e.g., active, idle)
    backend_xmin,                        -- Transaction ID associated with the backend (used to track old transaction data)
    xact_start,							  --The time when the transaction started (if applicable).
    query_start,                         -- Timestamp of when the query started
    age(backend_xmin) AS backend_age,    -- Age of the backend's transaction ID, indicating how long the backend has been running
    client_addr,                         -- IP address of the client connected to the PostgreSQL instance
    client_hostname,                     -- Hostname of the client (if available)
    client_port,                         -- Port number of the client
	--query,                               -- The actual SQL query being executed by the backend
	state								  --The time when the transaction started (if applicable).
FROM
    pg_stat_activity
WHERE
    backend_xmin IS NOT NULL            -- Filters to only include rows where a transaction ID exists for the backend
    --AND state = 'active'                -- Only includes active sessions (those currently executing queries)
    --AND query_start < now() - interval '5 minutes'  -- Filters for queries that have been running for more
order by age(backend_xmin)
```
This will show you long-running queries and their durations. You can identify and terminate long-running queries that may be holding onto old tuples.

- **Check for replication slot issues**:
```sql
SELECT * FROM pg_replication_slots;
```
If you see any replication slots that are "stuck" or unused, they might be preventing vacuum from cleaning up dead tuples. Drop unused slots using:
```sql
SELECT pg_drop_replication_slot('slot_name');
```

- **Identify uncommitted prepared transactions**:
```sql
SELECT * FROM pg_prepared_xacts;
```
If you have uncommitted transactions here, you may need to commit or roll them back to allow vacuum to proceed.

---

#### 4. **Vacuum Terminating Itself**
Vacuum might terminate early if it encounters lock contention, especially with DDL operations.

**Solution**: Schedule vacuum during off-peak hours to avoid conflicts with DDL operations.

**Query to monitor lock contention**:
```sql
SELECT
    pid,
    blocked_pid,
    blocked_user,
    query,
    age(clock_timestamp(), query_start) AS duration
FROM pg_locks l
JOIN pg_stat_activity a ON a.pid = l.pid
WHERE l.granted = false;
```
This query helps identify blocked processes, especially when vacuuming encounters DDL locks.

---

#### 5. **Transaction ID (XID) Wraparound**
PostgreSQL uses **32-bit transaction IDs (XIDs)**, meaning there’s a limit to how many transactions can occur. When nearing the limit, PostgreSQL will trigger a wraparound vacuum to free up transaction IDs. This is critical for avoiding downtime.

**Solution**: Ensure regular vacuuming to prevent XID wraparound issues. Adjust the `autovacuum_freeze_max_age` parameter to trigger more frequent vacuuming on tables that don’t receive frequent updates.

**Queries to monitor XID wraparound**:

- **Check tables approaching wraparound**:
```sql
SELECT 
    relname,
    age(relfrozenxid) AS xid_age
FROM 
    pg_class
WHERE 
    relkind = 'r'  -- Only user tables
    AND age(relfrozenxid) > 100000000;  -- Adjust threshold based on your system
```
This query identifies tables with XID ages approaching the wraparound limit, helping you target those that need more frequent vacuuming.

- **Check current wraparound vacuum settings**:
```sql
SHOW autovacuum_freeze_max_age;
SHOW vacuum_freeze_min_age;
```
You can adjust these settings to trigger vacuuming earlier if necessary, helping prevent wraparound.

---

### Conclusion

By understanding the common autovacuum problems and their solutions, and using the provided PostgreSQL queries, you can effectively tune autovacuum to maintain a PostgreSQL database that performs efficiently over time. Regular monitoring and tuning are essential to avoid bloat, improve query performance, and maintain database health. With proper adjustments to autovacuum settings, you can prevent issues such as excessive resource consumption, slow queries, and transaction ID wraparound, ensuring that your PostgreSQL database remains optimized for both small and large workloads.

---

References:
- https://www.youtube.com/watch?v=D832gi8Qrv4
