# MongoDB QA & Optimization

## Full Health Check

```javascript
// 1. Database stats
db.stats()

// 2. Collection sizes
db.getCollectionNames().forEach(name => {
  const stats = db[name].stats();
  print(`${name}: ${(stats.size / 1024 / 1024).toFixed(2)} MB, ${stats.count} docs, ${stats.nindexes} indexes`);
});

// 3. Index usage across all collections
db.getCollectionNames().forEach(name => {
  print(`\n--- ${name} ---`);
  db[name].aggregate([{ $indexStats: {} }]).forEach(idx => {
    print(`  ${idx.name}: ${idx.accesses.ops} uses since ${idx.accesses.since}`);
  });
});

// 4. Current operations (find slow ones)
db.currentOp({ "secs_running": { $gt: 5 } })

// 5. Server status
db.serverStatus().connections   // connection counts
db.serverStatus().opcounters    // operation counts
db.serverStatus().mem           // memory usage
```

## Query Optimization

### Read explain() Output

```javascript
const result = db.patients.find({ lastName: "Al-Rashid" }).explain("executionStats");

// Key metrics:
// executionStats.executionTimeMillis  — total time
// executionStats.totalDocsExamined    — docs scanned
// executionStats.nReturned            — docs returned
//
// RULE: If totalDocsExamined >> nReturned, you need an index
//
// winningPlan.inputStage.stage:
//   "COLLSCAN" = BAD (full collection scan)
//   "IXSCAN"   = GOOD (using index)
//   "FETCH"    = OK (index found doc, fetching full document)
```

### Index Strategies

```javascript
// Single field
db.patients.createIndex({ lastName: 1 })

// Compound (ESR Rule: Equality, Sort, Range)
// For query: find({ status: "active", age: { $gt: 18 } }).sort({ lastName: 1 })
db.patients.createIndex({ status: 1, lastName: 1, age: 1 })  // E, S, R order

// Text search
db.patients.createIndex({ firstName: "text", lastName: "text" })
// Query: db.patients.find({ $text: { $search: "Ahmed" } })

// TTL index (auto-delete old documents)
db.sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 86400 })

// Unique
db.patients.createIndex({ mrn: 1 }, { unique: true })

// Sparse (only index documents that have the field)
db.patients.createIndex({ email: 1 }, { sparse: true })

// Partial (index subset of documents)
db.patients.createIndex(
  { lastName: 1 },
  { partialFilterExpression: { status: "active" } }
)

// Wildcard (for flexible schemas)
db.products.createIndex({ "attributes.$**": 1 })
```

### Covered Queries (Fastest Possible)

```javascript
// If all returned fields are in the index, MongoDB skips document fetch entirely
db.patients.createIndex({ lastName: 1, firstName: 1, phone: 1 })

// This query is "covered" — returns directly from index
db.patients.find(
  { lastName: "Al-Rashid" },
  { _id: 0, firstName: 1, phone: 1 }  // only indexed fields, exclude _id
)
```

## Aggregation Pipeline Optimization

```javascript
// BAD: $match after $lookup (scans all joined docs)
db.appointments.aggregate([
  { $lookup: { from: "patients", localField: "patientId", foreignField: "_id", as: "patient" } },
  { $match: { status: "confirmed" } }  // too late!
])

// GOOD: $match BEFORE $lookup (filters first, joins less)
db.appointments.aggregate([
  { $match: { status: "confirmed" } },  // filter early
  { $lookup: { from: "patients", localField: "patientId", foreignField: "_id", as: "patient" } }
])

// GOOD: Use $project early to reduce document size in pipeline
db.appointments.aggregate([
  { $match: { status: "confirmed" } },
  { $project: { patientId: 1, date: 1, doctorId: 1 } },  // reduce fields
  { $lookup: { from: "patients", localField: "patientId", foreignField: "_id", as: "patient" } }
])
```

## Scaling Checklist

### < 1K Documents
- [ ] Schema design follows embedding vs referencing best practices
- [ ] Unique indexes on identifying fields (email, mrn)
- [ ] Validation rules in place (`db.createCollection` with `validator`)

### 1K — 100K Documents
- [ ] Indexes on all query patterns
- [ ] Compound indexes follow ESR rule
- [ ] No unbounded arrays in documents
- [ ] Aggregation pipelines filter early

### 100K+ Documents (Production)
- [ ] Connection pooling configured (driver maxPoolSize)
- [ ] Read preference set for read replicas
- [ ] Sharding considered for > 10M docs
- [ ] Change streams instead of polling for real-time
- [ ] TTL indexes on temporary/session data
- [ ] Profiler enabled for slow query detection
- [ ] Monitoring via Atlas or PMM

## Data Integrity Checks

```javascript
// Find documents missing required fields
db.patients.find({
  $or: [
    { firstName: { $exists: false } },
    { lastName: { $exists: false } },
    { phone: { $exists: false } }
  ]
}).count()

// Find duplicate values (should be unique)
db.patients.aggregate([
  { $group: { _id: "$mrn", count: { $sum: 1 } } },
  { $match: { count: { $gt: 1 } } }
])

// Find orphaned references
db.appointments.aggregate([
  { $lookup: { from: "patients", localField: "patientId", foreignField: "_id", as: "patient" } },
  { $match: { patient: { $size: 0 } } },
  { $project: { _id: 1, patientId: 1 } }
])

// Schema consistency check (find unexpected field types)
db.patients.find({ phone: { $not: { $type: "string" } }, phone: { $exists: true } })
```
