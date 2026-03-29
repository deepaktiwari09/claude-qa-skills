# DB Business Logic Knowledge Base — Reference

Every DB QA session should map the business logic living inside the database and cross-reference it with application service files. This reveals hidden rules, redundant enforcement, cost optimization opportunities, and stability risks.

## Knowledge Base Location

```
{project}/e2e/knowledge/db/
├── schema-map.md                  → tables, relationships, constraints
├── business-rules.md              → rules enforced by DB vs app vs frontend
├── service-db-mapping.md          → which service file hits which tables
├── query-patterns.md              → hot queries, cost, optimization status
├── cost-analysis.md               → infra cost tied to query patterns
└── stability-risks.md             → race conditions, missing transactions, cascades
```

---

## Step 1: Schema Discovery

### PostgreSQL — Extract Full Schema

```sql
-- All tables with row counts
SELECT schemaname, relname, n_live_tup AS row_count
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC;

-- All columns with types + constraints
SELECT t.table_name, c.column_name, c.data_type,
       c.is_nullable, c.column_default,
       tc.constraint_type
FROM information_schema.tables t
JOIN information_schema.columns c
  ON t.table_name = c.table_name
LEFT JOIN information_schema.constraint_column_usage ccu
  ON c.column_name = ccu.column_name AND c.table_name = ccu.table_name
LEFT JOIN information_schema.table_constraints tc
  ON ccu.constraint_name = tc.constraint_name
WHERE t.table_schema = 'public'
ORDER BY t.table_name, c.ordinal_position;

-- All foreign keys (relationships)
SELECT
  tc.table_name AS source_table,
  kcu.column_name AS source_column,
  ccu.table_name AS target_table,
  ccu.column_name AS target_column,
  rc.delete_rule,
  rc.update_rule
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
  ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage ccu
  ON tc.constraint_name = ccu.constraint_name
JOIN information_schema.referential_constraints rc
  ON tc.constraint_name = rc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY';

-- All CHECK constraints
SELECT conrelid::regclass AS table_name,
       conname AS constraint_name,
       pg_get_constraintdef(oid) AS definition
FROM pg_constraint
WHERE contype = 'c'
ORDER BY conrelid::regclass::text;

-- All triggers
SELECT event_object_table AS table_name,
       trigger_name,
       event_manipulation AS event,
       action_timing AS timing,
       action_statement
FROM information_schema.triggers
WHERE trigger_schema = 'public';

-- All stored procedures/functions
SELECT routine_name, routine_type, data_type
FROM information_schema.routines
WHERE routine_schema = 'public';
```

### MySQL — Extract Full Schema

```sql
-- All tables with engine and row count
SELECT TABLE_NAME, ENGINE, TABLE_ROWS, DATA_LENGTH, INDEX_LENGTH
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE()
ORDER BY TABLE_ROWS DESC;

-- All foreign keys
SELECT TABLE_NAME, COLUMN_NAME,
       REFERENCED_TABLE_NAME, REFERENCED_COLUMN_NAME,
       DELETE_RULE, UPDATE_RULE
FROM information_schema.KEY_COLUMN_USAGE kcu
JOIN information_schema.REFERENTIAL_CONSTRAINTS rc
  ON kcu.CONSTRAINT_NAME = rc.CONSTRAINT_NAME
WHERE kcu.TABLE_SCHEMA = DATABASE()
  AND kcu.REFERENCED_TABLE_NAME IS NOT NULL;

-- All triggers
SHOW TRIGGERS;
```

### MongoDB — Extract Schema Shape

```javascript
// Get schema shape from sample documents
db.getCollectionNames().forEach(name => {
  const sample = db[name].findOne();
  print(`\n--- ${name} ---`);
  print(JSON.stringify(Object.keys(sample || {})));

  // Check for validation rules
  const info = db.getCollectionInfos({ name: name })[0];
  if (info.options && info.options.validator) {
    print('Validator:', JSON.stringify(info.options.validator));
  }
});

// Check indexes as implicit business rules
db.getCollectionNames().forEach(name => {
  const indexes = db[name].getIndexes();
  indexes.forEach(idx => {
    if (idx.unique) print(`${name}: UNIQUE on ${JSON.stringify(idx.key)}`);
    if (idx.expireAfterSeconds) print(`${name}: TTL ${idx.expireAfterSeconds}s on ${JSON.stringify(idx.key)}`);
  });
});
```

---

## Step 2: Service File Discovery

Scan the codebase to find which service files interact with the database:

### Finding Service-DB Connections

```bash
# Find ORM model definitions
grep -rn "model\|entity\|schema\|Table\|Collection" src/ --include="*.ts" --include="*.go" --include="*.py" | grep -i "table\|collection\|model"

# Find query patterns
grep -rn "SELECT\|INSERT\|UPDATE\|DELETE\|findOne\|findMany\|create\|aggregate" src/ --include="*.ts" --include="*.go" --include="*.py"

# Find transaction usage
grep -rn "transaction\|BEGIN\|COMMIT\|ROLLBACK\|startSession" src/ --include="*.ts" --include="*.go" --include="*.py"

# Find raw SQL queries
grep -rn "raw\|exec\|query\|Exec\|Raw" src/ --include="*.ts" --include="*.go" --include="*.py"
```

### ORM-Specific Patterns

```bash
# GORM (Go)
grep -rn "db\.\(Find\|First\|Create\|Save\|Delete\|Where\|Joins\|Preload\)" src/ --include="*.go"

# Prisma (TypeScript)
grep -rn "prisma\.\w\+\.\(findUnique\|findMany\|create\|update\|delete\|upsert\)" src/ --include="*.ts"

# Drizzle (TypeScript)
grep -rn "db\.\(select\|insert\|update\|delete\|query\)" src/ --include="*.ts"

# TypeORM (TypeScript)
grep -rn "\.\(find\|findOne\|save\|remove\|createQueryBuilder\)" src/ --include="*.ts"

# Mongoose (TypeScript/JavaScript)
grep -rn "\.\(find\|findById\|findOne\|create\|updateOne\|deleteOne\|aggregate\)" src/ --include="*.ts" --include="*.js"
```

---

## Step 3: Business Rules Audit

### Rule Enforcement Matrix

For each business rule, document WHERE it's enforced:

| Rule | DB | Backend | Frontend | Risk Level |
|------|-----|---------|----------|------------|
| Email unique | UNIQUE constraint | Middleware check | Form validation | Low |
| Password min 8 | None! | Zod schema | Form minLength | **High** — DB allows any |
| Age >= 0 | CHECK constraint | Not checked | Input min=0 | Low |
| Order total > 0 | None | Calculated field | Not shown | **Medium** — edge case |
| Can't delete admin | None | Role guard | Button hidden | **High** — API bypass |

### Risk Levels

| Level | Meaning | Example |
|-------|---------|---------|
| **Low** | DB is source of truth, other layers are convenience | UNIQUE email in DB + form validation |
| **Medium** | Rule exists only in code, DB allows violation | Min password length only in Zod |
| **High** | Rule exists only in frontend or nowhere | Admin deletion only blocked by hidden button |
| **Critical** | Race condition or inconsistency possible | Stock decrement without row lock |

### Common Rule Gaps

Look for these patterns:

1. **Frontend-only rules** — button hidden but API endpoint unprotected
2. **Code-only validation** — Zod/Joi validates but DB schema allows anything
3. **Missing NOT NULL** — code assumes value exists but column is nullable
4. **Missing ON DELETE** — foreign key exists but no cascade/restrict behavior defined
5. **Soft delete inconsistency** — some tables use `deleted_at`, others hard-delete
6. **Timestamp gaps** — some tables missing `created_at`/`updated_at`
7. **Status enum drift** — DB CHECK has fewer statuses than code enum

---

## Step 4: Query Cost Analysis

### Identifying Expensive Queries

```sql
-- PostgreSQL: Top queries by total time
SELECT query,
       calls,
       round(total_exec_time::numeric, 2) AS total_ms,
       round(mean_exec_time::numeric, 2) AS avg_ms,
       rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- PostgreSQL: Queries with worst avg time
SELECT query,
       calls,
       round(mean_exec_time::numeric, 2) AS avg_ms
FROM pg_stat_statements
WHERE calls > 10
ORDER BY mean_exec_time DESC
LIMIT 10;
```

### Mapping Queries to Infrastructure Cost

```markdown
## Cost Formula

Daily CPU time = calls_per_day × avg_query_time_ms / 1000

## Cost Categories

| Daily CPU Seconds | Category | Action |
|-------------------|----------|--------|
| < 60s | Green | No action needed |
| 60-300s | Yellow | Add index or optimize |
| 300-1000s | Orange | Priority optimization |
| > 1000s | Red | Architectural change needed |

## Infrastructure Impact

| Optimization | Before | After | Monthly Savings |
|-------------|--------|-------|-----------------|
| Add GIN index on name search | 2,400 × 340ms = 816s/day | 2,400 × 15ms = 36s/day | ~$30 (smaller instance) |
| Materialized view for dashboard | 200 × 1.2s = 240s/day | 1 × 1.2s = 1.2s/day | ~$15 |
| Connection pooling | 500 connections peak | 50 connections peak | ~$40 (fewer replicas) |
```

---

## Step 5: Stability Risk Assessment

### What to Check

```sql
-- PostgreSQL: Find tables without primary keys (data integrity risk)
SELECT t.table_name
FROM information_schema.tables t
LEFT JOIN information_schema.table_constraints tc
  ON t.table_name = tc.table_name AND tc.constraint_type = 'PRIMARY KEY'
WHERE t.table_schema = 'public'
  AND t.table_type = 'BASE TABLE'
  AND tc.constraint_name IS NULL;

-- Find foreign keys without indexes (JOIN performance risk)
SELECT conrelid::regclass AS table_name,
       a.attname AS column_name,
       'Missing index on FK column' AS issue
FROM pg_constraint c
JOIN pg_attribute a ON a.attnum = ANY(c.conkey) AND a.attrelid = c.conrelid
LEFT JOIN pg_index i ON i.indrelid = c.conrelid AND a.attnum = ANY(i.indkey)
WHERE c.contype = 'f' AND i.indexrelid IS NULL;

-- Find tables without updated_at (audit trail risk)
SELECT table_name
FROM information_schema.tables t
WHERE table_schema = 'public'
  AND table_type = 'BASE TABLE'
  AND table_name NOT IN (
    SELECT table_name FROM information_schema.columns
    WHERE column_name = 'updated_at'
  );
```

### Transaction Boundary Analysis

Scan service files for operations that SHOULD be transactional but aren't:

| Pattern | Risk | Example |
|---------|------|---------|
| Multiple writes without transaction | Partial update | Create order + decrement stock without BEGIN/COMMIT |
| Delete + cascade without transaction | Orphaned records | Delete user but payment records remain |
| Read-modify-write without lock | Race condition | Two users buying last item simultaneously |
| External API call inside transaction | Long-held lock | Payment API timeout holds DB lock for 30s |

### Cascade Risk Map

Document what happens when each entity is deleted:

```markdown
## Cascade Map

### Delete User
- orders → SET NULL (orders.user_id becomes NULL)
- comments → CASCADE (all comments deleted)
- sessions → CASCADE (all sessions deleted)
- payments → RESTRICT (can't delete user with payments)
- **Risk**: Comments silently deleted. User might want to archive, not cascade.

### Delete Product
- order_items → RESTRICT (can't delete if in any order)
- cart_items → CASCADE (removed from all carts)
- reviews → CASCADE (all reviews deleted)
- **Risk**: Reviews lost permanently. Should soft-delete instead.
```

---

## Step 6: Recommendations Report Template

After completing the DB QA, generate this report:

```markdown
# DB Business Logic Report — [Project Name]

**Date:** YYYY-MM-DD
**Database:** PostgreSQL 16 / MySQL 8 / MongoDB 7
**Tables:** N | Total Rows: N | Total Size: X GB

## Business Rules Summary
- Rules in DB only: N
- Rules in code only: N (⚠ these need DB enforcement)
- Rules in both: N
- Unprotected rules: N (⚠ high risk)

## Service-DB Mapping
- Service files analyzed: N
- Unique query patterns: N
- Unindexed queries: N
- Missing transactions: N

## Cost Optimization
- Current daily query CPU: Xs
- Optimized projection: Ys (Z% reduction)
- Top 3 optimization actions: [listed]

## Stability Risks
- Critical: N (must fix before production)
- Warning: N (fix in next sprint)
- Info: N (nice to have)

## Action Items
1. [Highest impact action]
2. [Second highest]
3. [Third highest]

Want me to create Jira tickets for these?
```

---

## Cross-Referencing with UI Knowledge Base

The DB knowledge base connects with UI QA knowledge base:

| UI Flow | DB Impact | Optimization |
|---------|-----------|-------------|
| User registration (5 steps) | INSERT users + INSERT profiles + INSERT audit_log | Batch inserts in single transaction |
| Product search | ILIKE on 3 columns, full scan | GIN trigram index |
| Dashboard load | 6 aggregate queries | Materialized view, refresh hourly |
| Order checkout | INSERT order + N × INSERT order_items + UPDATE inventory | Transaction + row-level lock on inventory |

This mapping helps answer: **"If we simplify the UI flow, how does it affect the DB load?"**

Example: Combining registration steps 3+4 (from UI knowledge base) means one fewer INSERT + one fewer page navigation API call → saves ~15ms DB time + ~200ms network time per registration.
