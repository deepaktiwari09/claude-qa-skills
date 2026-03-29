# PostgreSQL Deep Optimization

## Full Health Check Script

Run this as a complete DB audit:

```sql
-- 1. Database size
SELECT pg_size_pretty(pg_database_size(current_database())) AS db_size;

-- 2. Table sizes (top 20)
SELECT relname AS table_name,
       pg_size_pretty(pg_total_relation_size(relid)) AS total,
       pg_size_pretty(pg_relation_size(relid)) AS data,
       pg_size_pretty(pg_indexes_size(relid)) AS indexes,
       n_live_tup AS live_rows,
       n_dead_tup AS dead_rows
FROM pg_catalog.pg_statio_user_tables
JOIN pg_stat_user_tables USING (relid)
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 20;

-- 3. Index health
SELECT schemaname, relname, indexrelname,
       idx_scan AS times_used,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size,
       CASE WHEN idx_scan = 0 THEN 'UNUSED — candidate for removal'
            WHEN idx_scan < 50 THEN 'RARELY USED'
            ELSE 'ACTIVE'
       END AS status
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC, pg_relation_size(indexrelid) DESC;

-- 4. Tables needing VACUUM
SELECT schemaname, relname,
       n_dead_tup,
       last_vacuum,
       last_autovacuum,
       last_analyze,
       last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

-- 5. Cache hit ratio (should be > 99%)
SELECT
  sum(heap_blks_read) AS heap_read,
  sum(heap_blks_hit) AS heap_hit,
  round(100.0 * sum(heap_blks_hit) / GREATEST(sum(heap_blks_hit) + sum(heap_blks_read), 1), 2) AS cache_hit_ratio
FROM pg_statio_user_tables;

-- 6. Connection usage
SELECT count(*) AS total_connections,
       count(*) FILTER (WHERE state = 'active') AS active,
       count(*) FILTER (WHERE state = 'idle') AS idle,
       count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_txn
FROM pg_stat_activity
WHERE backend_type = 'client backend';

-- 7. Bloat estimate (approximate)
SELECT relname, n_live_tup, n_dead_tup,
       round(100.0 * n_dead_tup / GREATEST(n_live_tup, 1), 1) AS bloat_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 100
ORDER BY bloat_pct DESC;
```

## Slow Query Detection

```sql
-- Enable pg_stat_statements (add to postgresql.conf)
-- shared_preload_libraries = 'pg_stat_statements'
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top 10 slowest queries (by average time)
SELECT
  substring(query, 1, 100) AS short_query,
  calls,
  round(mean_exec_time::numeric, 2) AS avg_ms,
  round(total_exec_time::numeric, 2) AS total_ms,
  rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Top 10 most called queries (optimize these for biggest impact)
SELECT
  substring(query, 1, 100) AS short_query,
  calls,
  round(mean_exec_time::numeric, 2) AS avg_ms,
  round((total_exec_time / 1000)::numeric, 2) AS total_seconds
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;

-- Queries consuming most total time
SELECT
  substring(query, 1, 100) AS short_query,
  calls,
  round(mean_exec_time::numeric, 2) AS avg_ms,
  round((total_exec_time / 1000 / 60)::numeric, 2) AS total_minutes
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

## Index Recommendations by Query Pattern

| Query Pattern | Recommended Index |
|--------------|-------------------|
| `WHERE status = 'active'` | `CREATE INDEX idx_t_status ON t (status)` |
| `WHERE user_id = ? AND created_at > ?` | `CREATE INDEX idx_t_user_created ON t (user_id, created_at)` |
| `WHERE name ILIKE '%search%'` | `CREATE INDEX idx_t_name_gin ON t USING gin (name gin_trgm_ops)` |
| `ORDER BY created_at DESC LIMIT 20` | `CREATE INDEX idx_t_created_desc ON t (created_at DESC)` |
| `WHERE tenant_id = ? AND status = ?` | `CREATE INDEX idx_t_tenant_status ON t (tenant_id, status)` |
| `WHERE deleted_at IS NULL` | `CREATE INDEX idx_t_active ON t (...) WHERE deleted_at IS NULL` |

## Connection Pooling with PgBouncer

For production with many concurrent connections:

```ini
# pgbouncer.ini
[databases]
myapp = host=127.0.0.1 port=5432 dbname=myapp

[pgbouncer]
pool_mode = transaction          ; best for web apps
max_client_conn = 1000           ; max incoming connections
default_pool_size = 25           ; connections per database
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 3
```

## Partitioning for Large Tables

```sql
-- Range partition by date (for tables > 10M rows)
CREATE TABLE audit_logs (
    id BIGSERIAL,
    action TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    data JSONB
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE audit_logs_2026_01 PARTITION OF audit_logs
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE audit_logs_2026_02 PARTITION OF audit_logs
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... etc

-- Auto-create partitions with pg_partman extension
```

## Maintenance Commands

```sql
-- Update statistics (helps query planner)
ANALYZE patients;
ANALYZE; -- all tables

-- Reclaim dead tuples
VACUUM ANALYZE patients;

-- Full vacuum (locks table — use during maintenance window)
VACUUM FULL patients;

-- Rebuild bloated indexes
REINDEX INDEX CONCURRENTLY idx_patients_name;
REINDEX TABLE CONCURRENTLY patients;
```
