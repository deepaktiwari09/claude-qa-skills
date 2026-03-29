---
name: db-qa-testing
description: Database QA testing, query optimization, and index analysis for PostgreSQL, MySQL, and MongoDB. Use when asked to "check the database", "optimize queries", "analyze indexes", "verify DB changes", "test database", "check DB performance", "audit schema", "optimize for production", "scale database", or any database-related QA work.
compatibility: Requires access to PostgreSQL (psql), MySQL (mysql), or MongoDB (mongosh)
metadata:
  author: deepaktiwari09
  version: "1.0.0"
---

# Database QA Testing & Optimization

You are a senior DBA and QA engineer. Your job is to verify data integrity after changes, find performance bottlenecks, optimize queries and indexes, and ensure the database is production-ready whether it serves 10 users or 10 million.

## Step 0: Detect Database Type & Connect

Before anything, determine what database you're working with:

```bash
# Check for connection strings in env files
grep -i "database\|db_url\|postgres\|mysql\|mongo\|redis" .env .env.local .env.* 2>/dev/null

# Check docker-compose for DB services
grep -i "postgres\|mysql\|mongo\|mariadb\|redis" docker-compose*.yml 2>/dev/null

# Check ORM config (Prisma, TypeORM, Drizzle, GORM, etc.)
find . -name "schema.prisma" -o -name "ormconfig*" -o -name "drizzle.config*" -o -name "*.go" 2>/dev/null | head -5
```

### Connection Commands

```bash
# PostgreSQL
psql "postgresql://user:pass@localhost:5432/dbname"
# or
PGPASSWORD=pass psql -h localhost -p 5432 -U user -d dbname

# MySQL
mysql -h localhost -P 3306 -u user -ppassword dbname

# MongoDB
mongosh "mongodb://user:pass@localhost:27017/dbname"
# or
mongosh --host localhost --port 27017 -u user -p pass --authenticationDatabase admin dbname
```

**IMPORTANT:** Never hardcode credentials. Always read from `.env` or ask the user.

---

## Part 1: Data Verification (QA After Code Changes)

### Verify CRUD Operations

After a feature is implemented, verify the data layer:

#### PostgreSQL / MySQL
```sql
-- After CREATE: verify record exists with correct fields
SELECT * FROM patients WHERE id = (SELECT MAX(id) FROM patients);

-- After UPDATE: verify field changed and updated_at set
SELECT id, phone, updated_at FROM patients WHERE id = 123;

-- After DELETE: verify soft delete (if applicable)
SELECT id, deleted_at FROM patients WHERE id = 123;
-- or hard delete
SELECT COUNT(*) FROM patients WHERE id = 123; -- should be 0

-- Verify foreign key integrity
SELECT p.id, p.first_name, a.id as appointment_id
FROM patients p
LEFT JOIN appointments a ON a.patient_id = p.id
WHERE p.id = 123;

-- Check for orphaned records
SELECT a.id FROM appointments a
LEFT JOIN patients p ON p.id = a.patient_id
WHERE p.id IS NULL;
```

#### MongoDB
```javascript
// After CREATE
db.patients.findOne({ _id: ObjectId("...") })

// After UPDATE
db.patients.findOne({ _id: ObjectId("...") }, { phone: 1, updatedAt: 1 })

// Check for orphaned references
db.appointments.aggregate([
  { $lookup: { from: "patients", localField: "patientId", foreignField: "_id", as: "patient" } },
  { $match: { patient: { $size: 0 } } },
  { $count: "orphaned" }
])
```

### Verify Migrations

After running a migration:
```sql
-- PostgreSQL: check column was added
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_name = 'patients' AND column_name = 'new_column';

-- Check constraint was added
SELECT conname, contype, pg_get_constraintdef(oid)
FROM pg_constraint
WHERE conrelid = 'patients'::regclass;

-- Check index was created
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'patients';
```

---

## Part 2: Query Performance Analysis

### PostgreSQL — EXPLAIN ANALYZE

```sql
-- Basic query plan
EXPLAIN ANALYZE
SELECT * FROM patients
WHERE first_name ILIKE '%ahmed%'
ORDER BY created_at DESC
LIMIT 20;
```

**Reading the output:**
```
Seq Scan on patients  (cost=0.00..1234.00 rows=50 width=200) (actual time=0.5..45.2 rows=12 loops=1)
  Filter: (first_name ~~* '%ahmed%')
  Rows Removed by Filter: 9988
Planning Time: 0.1 ms
Execution Time: 45.3 ms
```

| Signal | Meaning | Action |
|--------|---------|--------|
| `Seq Scan` on large table | Full table scan | Add index |
| `Rows Removed by Filter: 9988` | Scanning many, returning few | Index needed on filter column |
| `Sort` with high cost | In-memory sort | Add index matching ORDER BY |
| `Nested Loop` with many rows | N+1 join pattern | Consider `Hash Join` or add index |
| `Execution Time > 100ms` | Slow query | Optimize |

### MySQL — EXPLAIN

```sql
EXPLAIN SELECT * FROM patients
WHERE first_name LIKE '%ahmed%'
ORDER BY created_at DESC
LIMIT 20;

-- Extended format
EXPLAIN FORMAT=JSON SELECT ...;
```

**Key columns:**
| Column | Good | Bad |
|--------|------|-----|
| `type` | `ref`, `range`, `const` | `ALL` (full table scan) |
| `key` | Shows index name | `NULL` (no index used) |
| `rows` | Small number | Large number (scanning many rows) |
| `Extra` | `Using index` | `Using filesort`, `Using temporary` |

### MongoDB — explain()

```javascript
// Query explanation
db.patients.find({ first_name: /ahmed/i }).sort({ createdAt: -1 }).limit(20).explain("executionStats")

// Key fields to check
// executionStats.totalDocsExamined vs executionStats.nReturned
// If examined >> returned, index is missing

// Check if index was used
// winningPlan.inputStage.stage should be "IXSCAN" not "COLLSCAN"
```

---

## Part 3: Index Analysis & Optimization

### PostgreSQL

```sql
-- List all indexes on a table
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'patients'
ORDER BY indexname;

-- Find MISSING indexes (columns used in WHERE but not indexed)
-- Run slow query log first, then check:
SELECT schemaname, relname, seq_scan, seq_tup_read,
       idx_scan, idx_tup_fetch
FROM pg_stat_user_tables
WHERE seq_scan > 100
ORDER BY seq_tup_read DESC
LIMIT 20;
-- High seq_scan + high seq_tup_read = needs index

-- Find UNUSED indexes (wasting disk + slowing writes)
SELECT schemaname, relname, indexrelname, idx_scan, idx_tup_read,
       pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Index usage ratio
SELECT relname,
       CASE WHEN idx_scan + seq_scan = 0 THEN 0
            ELSE round(100.0 * idx_scan / (idx_scan + seq_scan), 1)
       END AS index_usage_pct
FROM pg_stat_user_tables
ORDER BY index_usage_pct ASC;

-- Table and index sizes
SELECT relname,
       pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
       pg_size_pretty(pg_relation_size(relid)) AS table_size,
       pg_size_pretty(pg_indexes_size(relid)) AS index_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```

#### Creating Better Indexes
```sql
-- Single column index
CREATE INDEX idx_patients_first_name ON patients (first_name);

-- Composite index (order matters! most selective column first)
CREATE INDEX idx_patients_name_dob ON patients (last_name, first_name, date_of_birth);

-- Partial index (only index active records — saves space)
CREATE INDEX idx_patients_active ON patients (last_name, first_name)
WHERE deleted_at IS NULL;

-- GIN index for full-text / ILIKE search
CREATE INDEX idx_patients_name_gin ON patients
USING gin (first_name gin_trgm_ops, last_name gin_trgm_ops);
-- Requires: CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Expression index (for computed lookups)
CREATE INDEX idx_patients_email_lower ON patients (LOWER(email));

-- Covering index (includes extra columns to avoid table lookup)
CREATE INDEX idx_patients_search ON patients (last_name, first_name)
INCLUDE (phone, mrn);
```

### MySQL

```sql
-- List indexes
SHOW INDEX FROM patients;

-- Find tables without indexes (other than PK)
SELECT t.TABLE_NAME, t.TABLE_ROWS
FROM information_schema.TABLES t
LEFT JOIN information_schema.STATISTICS s
  ON t.TABLE_NAME = s.TABLE_NAME AND s.INDEX_NAME != 'PRIMARY'
WHERE t.TABLE_SCHEMA = DATABASE() AND s.INDEX_NAME IS NULL
  AND t.TABLE_ROWS > 1000;

-- Index suggestions via slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 0.5;
```

### MongoDB

```javascript
// List all indexes
db.patients.getIndexes()

// Index usage stats
db.patients.aggregate([{ $indexStats: {} }])

// Find queries not using indexes
db.setProfilingLevel(1, { slowms: 100 })
db.system.profile.find({ millis: { $gt: 100 } }).sort({ ts: -1 }).limit(10)

// Create indexes
db.patients.createIndex({ lastName: 1, firstName: 1 })
db.patients.createIndex({ phone: 1 }, { unique: true })
db.patients.createIndex({ firstName: "text", lastName: "text" }) // text search

// Drop unused index
db.patients.dropIndex("index_name")
```

---

## Part 4: Scale Assessment

### For Few Users (< 1K records) — Dev/Startup

```markdown
## Assessment Checklist
- [ ] All queries use indexes? (even small tables benefit)
- [ ] Foreign keys have ON DELETE action? (CASCADE, SET NULL, RESTRICT)
- [ ] NOT NULL constraints on required fields?
- [ ] Unique constraints where needed? (email, MRN, phone)
- [ ] created_at / updated_at timestamps on all tables?
- [ ] Soft deletes if applicable? (deleted_at column)
```

Focus: **correctness over performance**. Get the schema right now — it's expensive to change later.

### For Medium Scale (1K-100K records) — Growing Product

```sql
-- PostgreSQL: enable query stats
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Find top 10 slowest queries
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Find most called queries (optimize these first)
SELECT query, calls, mean_exec_time
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;
```

Focus: **add indexes, fix N+1 queries, add pagination everywhere**.

### For Production Scale (100K-10M+ records) — Enterprise

```sql
-- Table bloat check
SELECT schemaname, relname,
       n_live_tup, n_dead_tup,
       round(100.0 * n_dead_tup / GREATEST(n_live_tup + n_dead_tup, 1), 1) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
-- If dead_pct > 20%, run VACUUM ANALYZE

-- Connection pool usage
SELECT count(*), state FROM pg_stat_activity GROUP BY state;

-- Long-running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE state != 'idle'
  AND now() - pg_stat_activity.query_start > interval '30 seconds'
ORDER BY duration DESC;

-- Lock contention
SELECT blocked_locks.pid AS blocked_pid,
       blocking_locks.pid AS blocking_pid,
       blocked_activity.query AS blocked_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_locks blocking_locks
  ON blocking_locks.locktype = blocked_locks.locktype
  AND blocking_locks.relation = blocked_locks.relation
  AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocked_activity
  ON blocked_activity.pid = blocked_locks.pid;
```

#### Production Optimization Checklist
```markdown
## Schema
- [ ] Partitioning for tables > 10M rows (by date, tenant, etc.)
- [ ] Archive old data (move to cold storage)
- [ ] Appropriate column types (don't use TEXT for fixed-length data)

## Queries
- [ ] No SELECT * — specify only needed columns
- [ ] All WHERE/JOIN columns indexed
- [ ] No N+1 queries (use JOIN or batch loading)
- [ ] Pagination on all list endpoints (LIMIT/OFFSET or cursor)
- [ ] Cursor-based pagination for > 100K rows (OFFSET gets slow)

## Infrastructure
- [ ] Read replicas for read-heavy workloads
- [ ] Connection pooling (PgBouncer for PostgreSQL)
- [ ] Query result caching (Redis) for hot paths
- [ ] Monitoring dashboards (query time, connections, disk usage)
- [ ] Automated backups tested (can you actually restore?)

## Indexes
- [ ] Composite indexes match query patterns (column order matters)
- [ ] Partial indexes for filtered queries (WHERE active = true)
- [ ] Covering indexes for read-heavy queries (INCLUDE clause)
- [ ] No redundant indexes (idx on (a,b) makes idx on (a) redundant)
- [ ] Regular REINDEX for bloated indexes
```

---

## Part 5: DB QA Report

```markdown
# Database QA Report
**Date:** YYYY-MM-DD
**Database:** PostgreSQL 16 / MySQL 8 / MongoDB 7
**Host:** localhost:5432 / staging-db.example.com
**Total Records:** patients: 15,230 | appointments: 45,890

## Schema Health
- Tables: 12
- Missing indexes: 3 (listed below)
- Unused indexes: 2 (candidates for removal)
- Orphaned records: 0

## Missing Indexes
| Table | Column(s) | Query Pattern | Impact |
|-------|-----------|---------------|--------|
| patients | (last_name, first_name) | Search by name | Seq scan on 15K rows |
| appointments | (doctor_id, date) | Doctor schedule lookup | 450ms → ~5ms with index |
| audit_logs | (created_at) | Date range queries | Full scan on 200K rows |

## Unused Indexes (safe to drop)
| Index | Table | Size | Last Used |
|-------|-------|------|-----------|
| idx_patients_old_status | patients | 2.1 MB | Never |
| idx_temp_migration | appointments | 850 KB | Never |

## Slow Queries (p95 > 200ms)
| Query Pattern | Avg Time | Calls/day | Fix |
|---------------|----------|-----------|-----|
| SELECT * FROM patients WHERE name ILIKE... | 340ms | 2,400 | Add GIN trigram index |
| SELECT * FROM appointments JOIN... | 890ms | 180 | Add composite index on (doctor_id, date) |

## Optimization Actions
1. CREATE INDEX idx_patients_name ON patients (last_name, first_name);
2. CREATE INDEX idx_appointments_doctor_date ON appointments (doctor_id, appointment_date);
3. CREATE INDEX idx_audit_created ON audit_logs (created_at);
4. DROP INDEX idx_patients_old_status;
5. DROP INDEX idx_temp_migration;
6. VACUUM ANALYZE patients;

## Scale Readiness: [LEVEL]
- [ ] Dev (< 1K) — schema correct
- [x] Growth (1K-100K) — indexes needed (see above)
- [ ] Production (100K+) — needs connection pooling + caching
```

## Quick Reference

| Task | PostgreSQL | MySQL | MongoDB |
|------|-----------|-------|---------|
| Show tables | `\dt` | `SHOW TABLES` | `show collections` |
| Describe table | `\d patients` | `DESCRIBE patients` | `db.patients.findOne()` |
| List indexes | `\di` or `pg_indexes` | `SHOW INDEX FROM t` | `db.t.getIndexes()` |
| Query plan | `EXPLAIN ANALYZE` | `EXPLAIN` | `.explain()` |
| Table size | `pg_total_relation_size()` | `information_schema` | `db.t.stats()` |
| Active queries | `pg_stat_activity` | `SHOW PROCESSLIST` | `db.currentOp()` |
| Kill query | `pg_cancel_backend(pid)` | `KILL pid` | `db.killOp(opid)` |

## Specific References

* **PostgreSQL deep optimization** [references/postgresql-optimization.md](references/postgresql-optimization.md)
* **MongoDB patterns** [references/mongodb-patterns.md](references/mongodb-patterns.md)
* **Business logic knowledge base** [references/business-logic-knowledge-base.md](references/business-logic-knowledge-base.md)
