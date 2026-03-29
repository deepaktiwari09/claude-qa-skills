# API Performance Testing Workflow

## Step 1: Baseline Measurement

Test each critical endpoint individually:

```bash
# GET list endpoint
hey -n 200 -c 10 \
  -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/api/v1/patients

# POST create endpoint
hey -n 100 -c 5 -m POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"first_name":"Load","last_name":"Test","phone":"+971500000000"}' \
  http://localhost:8080/api/v1/patients

# Search endpoint
hey -n 200 -c 20 \
  -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8080/api/v1/patients?search=Ahmed"
```

Record baseline numbers.

## Step 2: Gradual Load Increase

```bash
# Light load (10 concurrent)
hey -n 500 -c 10 -H "Authorization: Bearer $TOKEN" $ENDPOINT

# Medium load (50 concurrent)
hey -n 1000 -c 50 -H "Authorization: Bearer $TOKEN" $ENDPOINT

# Heavy load (100 concurrent)
hey -n 2000 -c 100 -H "Authorization: Bearer $TOKEN" $ENDPOINT

# Stress test (200 concurrent)
hey -n 5000 -c 200 -H "Authorization: Bearer $TOKEN" $ENDPOINT
```

## Step 3: Duration-Based Tests

```bash
# Sustained load for 1 minute
hey -z 60s -c 50 -H "Authorization: Bearer $TOKEN" $ENDPOINT

# 5-minute endurance test
hey -z 300s -c 30 -H "Authorization: Bearer $TOKEN" $ENDPOINT
```

## Step 4: Analyze Results

Look for:
- **Latency increase** — does p99 spike under load?
- **Error rate** — do requests start failing at high concurrency?
- **Throughput ceiling** — at what concurrency does RPS stop growing?
- **Memory leaks** — does sustained load cause degradation over time?

## Performance Report Template

```markdown
# Performance Test Report
**Date:** YYYY-MM-DD
**Endpoint:** GET /api/v1/patients
**Server:** localhost:8080 / staging

## Results by Concurrency Level

| Concurrency | Requests | RPS | p50 | p95 | p99 | Errors |
|-------------|----------|-----|-----|-----|-----|--------|
| 10 | 500 | 420 | 22ms | 45ms | 89ms | 0% |
| 50 | 1000 | 680 | 68ms | 190ms | 450ms | 0% |
| 100 | 2000 | 720 | 130ms | 890ms | 2.1s | 0.5% |
| 200 | 5000 | 650 | 290ms | 3.2s | 8.5s | 4.2% |

## Findings
- Throughput peaks at ~720 RPS with 100 concurrent users
- p99 exceeds 1s at 100 concurrent — needs optimization
- Errors begin at 200 concurrent — connection pool likely exhausted

## Recommendations
1. Add connection pooling (current: 10 → recommended: 50)
2. Add Redis cache for patient list (TTL: 30s)
3. Paginate response — currently returns all fields, use sparse fieldsets
```

## Quick Scripts

### Compare before/after optimization
```bash
echo "=== BEFORE ===" > perf-comparison.txt
hey -n 500 -c 50 -H "Authorization: Bearer $TOKEN" $ENDPOINT >> perf-comparison.txt 2>&1
echo ""
echo "=== Apply optimization, then press Enter ==="
read
echo "=== AFTER ===" >> perf-comparison.txt
hey -n 500 -c 50 -H "Authorization: Bearer $TOKEN" $ENDPOINT >> perf-comparison.txt 2>&1
cat perf-comparison.txt
```
