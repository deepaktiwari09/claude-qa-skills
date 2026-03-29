---
name: api-qa-testing
description: API QA testing, performance benchmarking, and improvement suggestions. Use when asked to "test the API", "check endpoints", "load test", "verify API response", "benchmark API", "test authentication flow", "chain API requests", or when QA-ing any backend service. Covers curl, Hurl (assertions + chaining), httpie, hey (load testing), and jq.
compatibility: Recommended tools - curl, hurl, hey, httpie, jq
metadata:
  author: deepaktiwari09
  version: "1.0.0"
---

# API QA Testing

You are a senior API QA engineer. Your job is to verify APIs work correctly, perform under load, and follow best practices. You use the right tool for each job.

## Tool Selection Guide

Pick the right tool based on the task:

| Task | Tool | Why |
|------|------|-----|
| Quick manual check | `curl` | Universal, already installed |
| Readable manual check | `httpie` / `xh` | Cleaner syntax, colored JSON |
| Assertions + chaining | `hurl` | Plain text test files with built-in assertions |
| Load / stress testing | `hey` / `oha` | Concurrent requests + latency stats |
| JSON parsing | `jq` | Filter, transform, validate JSON responses |
| WebSocket testing | `websocat` | Dedicated WebSocket CLI |
| GraphQL | `curl` + `jq` | POST with query body, parse response |

### Tool Check

Before starting, verify available tools:
```bash
curl --version | head -1
jq --version 2>/dev/null || echo "jq NOT installed — install: brew install jq"
hurl --version 2>/dev/null || echo "hurl NOT installed — install: brew install hurl"
hey 2>/dev/null | head -1 || echo "hey NOT installed — install: brew install hey"
httpie --version 2>/dev/null || http --version 2>/dev/null || echo "httpie NOT installed — install: brew install httpie"
```

## Phase 1: Discover & Document

Before testing, understand the API surface:

1. **Check for OpenAPI/Swagger docs** — look for `/docs`, `/swagger`, `/api-docs`
2. **Read route definitions** — grep codebase for endpoint patterns
3. **Identify auth mechanism** — JWT, API key, session cookie, OAuth
4. **List all endpoints** — method, path, required params, expected response

Create a test matrix:
```markdown
| # | Method | Endpoint | Auth | Body | Expected Status | Expected Response |
|---|--------|----------|------|------|-----------------|-------------------|
| 1 | GET | /api/v1/patients | JWT | — | 200 | { data: [], total: N } |
| 2 | POST | /api/v1/patients | JWT | Patient JSON | 201 | { id, ... } |
| 3 | GET | /api/v1/patients/:id | JWT | — | 200 | Patient object |
| 4 | GET | /api/v1/patients/:id | None | — | 401 | Unauthorized |
```

## Phase 2: Test with curl (Quick Manual Checks)

### Basic Patterns

```bash
# GET with auth header
curl -s -w "\n%{http_code} %{time_total}s" \
  -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/api/v1/patients | jq .

# POST with JSON body
curl -s -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"first_name":"Ahmed","last_name":"Al-Rashid","phone":"+971501234567"}' \
  http://localhost:8080/api/v1/patients | jq .

# PUT update
curl -s -X PUT \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"phone":"+971502222222"}' \
  http://localhost:8080/api/v1/patients/123 | jq .

# DELETE
curl -s -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/api/v1/patients/123 -w "\n%{http_code}"
```

### curl Power Tips

```bash
# Show response headers + body + timing
curl -s -i -w "\n---\nTime: %{time_total}s | Size: %{size_download} bytes | Status: %{http_code}\n" \
  http://localhost:8080/api/v1/patients

# Save response to file for comparison
curl -s http://localhost:8080/api/v1/patients | jq . > response-before.json
# ... make changes ...
curl -s http://localhost:8080/api/v1/patients | jq . > response-after.json
diff response-before.json response-after.json

# Test all HTTP methods on an endpoint
for method in GET POST PUT PATCH DELETE OPTIONS HEAD; do
  echo "$method: $(curl -s -o /dev/null -w '%{http_code}' -X $method http://localhost:8080/api/v1/patients)"
done

# Extract specific field with jq
curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/api/v1/patients | jq '.data | length'

# Verbose mode for debugging
curl -v http://localhost:8080/api/v1/health 2>&1
```

### Where curl Falls Short

| Limitation | Example | Better Tool |
|-----------|---------|-------------|
| **No assertions** | Can't auto-check `status == 200` | Use `hurl` |
| **Verbose chaining** | Login → get token → use in next request = complex bash | Use `hurl` with `[Captures]` |
| **No load testing** | Can't send 1000 concurrent requests | Use `hey` |
| **Ugly output** | Raw JSON wall of text | Pipe through `jq` or use `httpie` |
| **No test reports** | No JUnit/HTML output for CI | Use `hurl --report-junit` |
| **Hard to version control** | Long curl commands don't read well in git | Use `.hurl` files |
| **No response time assertions** | Can't fail if response > 500ms | Use `hurl` `duration < 500` |

## Phase 3: Test with Hurl (Assertions + Chaining)

Hurl is the **CLI Postman replacement**. Plain text files with built-in assertions.

### Install
```bash
brew install hurl
```

### Basic Hurl File

Create `tests/api/patients.hurl`:
```hurl
# Health check
GET http://localhost:8080/api/v1/health
HTTP 200
[Asserts]
jsonpath "$.status" == "ok"
duration < 500

# Login and capture token
POST http://localhost:8080/api/v1/auth/login
Content-Type: application/json
{
  "email": "doctor@hospital.com",
  "password": "password123"
}
HTTP 200
[Captures]
auth_token: jsonpath "$.token"
[Asserts]
jsonpath "$.token" isString
jsonpath "$.user.role" == "doctor"

# Use captured token for authenticated request
GET http://localhost:8080/api/v1/patients?page=1&limit=10
Authorization: Bearer {{auth_token}}
HTTP 200
[Asserts]
jsonpath "$.data" isCollection
jsonpath "$.data" count > 0
jsonpath "$.total" isInteger
jsonpath "$.page" == 1
duration < 1000

# Test 401 without token
GET http://localhost:8080/api/v1/patients
HTTP 401
[Asserts]
jsonpath "$.message" contains "unauthorized"
```

### Run Hurl Tests
```bash
# Run single file
hurl tests/api/patients.hurl --test

# Run all test files
hurl tests/api/*.hurl --test

# With variables (environment-specific)
hurl tests/api/patients.hurl --variable base_url=http://localhost:8080 --test

# Generate JUnit report for CI
hurl tests/api/*.hurl --test --report-junit report.xml

# Generate HTML report
hurl tests/api/*.hurl --test --report-html reports/

# Verbose mode for debugging
hurl tests/api/patients.hurl --very-verbose
```

### Hurl Assertion Reference

```hurl
# Status
HTTP 200
HTTP 201
HTTP 404

# JSON assertions
[Asserts]
jsonpath "$.name" == "Ahmed"
jsonpath "$.age" >= 18
jsonpath "$.tags" includes "vip"
jsonpath "$.data" isEmpty
jsonpath "$.data" count == 10
jsonpath "$.id" isInteger
jsonpath "$.email" matches "^[a-z]+@[a-z]+\\.[a-z]+$"

# Header assertions
header "Content-Type" contains "application/json"
header "X-Request-Id" exists
header "Cache-Control" == "no-cache"

# Performance
duration < 500
duration < 1000

# Body assertions
body contains "success"
bytes count > 0

# Cookie
cookie "session" exists
cookie "session[HttpOnly]" exists
```

### Hurl CRUD Test File

```hurl
# CREATE patient
POST http://localhost:8080/api/v1/patients
Authorization: Bearer {{auth_token}}
Content-Type: application/json
{
  "first_name": "Test",
  "last_name": "Patient",
  "phone": "+971509999999",
  "gender": "male",
  "date_of_birth": "1990-01-15"
}
HTTP 201
[Captures]
patient_id: jsonpath "$.id"
[Asserts]
jsonpath "$.first_name" == "Test"
jsonpath "$.id" isInteger

# READ patient
GET http://localhost:8080/api/v1/patients/{{patient_id}}
Authorization: Bearer {{auth_token}}
HTTP 200
[Asserts]
jsonpath "$.first_name" == "Test"
jsonpath "$.last_name" == "Patient"
jsonpath "$.phone" == "+971509999999"

# UPDATE patient
PUT http://localhost:8080/api/v1/patients/{{patient_id}}
Authorization: Bearer {{auth_token}}
Content-Type: application/json
{
  "phone": "+971508888888"
}
HTTP 200
[Asserts]
jsonpath "$.phone" == "+971508888888"

# DELETE patient
DELETE http://localhost:8080/api/v1/patients/{{patient_id}}
Authorization: Bearer {{auth_token}}
HTTP 200

# VERIFY deleted
GET http://localhost:8080/api/v1/patients/{{patient_id}}
Authorization: Bearer {{auth_token}}
HTTP 404
```

## Phase 4: Load Testing with hey

### Install
```bash
brew install hey
```

### Basic Load Test
```bash
# 200 requests, 20 concurrent
hey -n 200 -c 20 \
  -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/api/v1/patients

# POST load test
hey -n 100 -c 10 -m POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"search":"Ahmed"}' \
  http://localhost:8080/api/v1/patients/search

# Duration-based (30 seconds)
hey -z 30s -c 50 \
  -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/api/v1/patients
```

### Reading hey Output
```
Summary:
  Total:        2.5340 secs        ← total time
  Slowest:      0.3210 secs        ← worst case
  Fastest:      0.0120 secs        ← best case
  Average:      0.0450 secs        ← mean response time
  Requests/sec: 789.45             ← throughput

Status code distribution:
  [200] 200 responses              ← all succeeded

Latency distribution:
  10% in 0.0150 secs
  50% in 0.0380 secs               ← p50 (median)
  90% in 0.0890 secs               ← p90
  95% in 0.1200 secs               ← p95
  99% in 0.2800 secs               ← p99 (tail latency)
```

### Performance Benchmarks

| Metric | Good | Warning | Critical |
|--------|------|---------|----------|
| p50 latency | < 100ms | 100-500ms | > 500ms |
| p95 latency | < 300ms | 300ms-1s | > 1s |
| p99 latency | < 1s | 1-3s | > 3s |
| Error rate | 0% | < 1% | > 1% |
| Throughput | > 100 rps | 50-100 rps | < 50 rps |

## Phase 5: QA Report & Improvement Suggestions

After testing, generate a structured report:

```markdown
# API QA Report
**Date:** YYYY-MM-DD
**Base URL:** http://localhost:8080
**Auth:** JWT Bearer token

## Endpoint Test Results

| Endpoint | Method | Status | Response Time | Assertions | Result |
|----------|--------|--------|---------------|------------|--------|
| /api/v1/health | GET | 200 | 45ms | status=ok | PASS |
| /api/v1/patients | GET | 200 | 230ms | data array | PASS |
| /api/v1/patients | POST | 201 | 180ms | returns id | PASS |
| /api/v1/patients | GET (no auth) | 401 | 12ms | unauthorized | PASS |

## Load Test Results

| Endpoint | Concurrency | RPS | p50 | p95 | p99 | Errors |
|----------|-------------|-----|-----|-----|-----|--------|
| GET /patients | 20 | 450 | 38ms | 120ms | 280ms | 0% |
| POST /patients | 10 | 210 | 85ms | 350ms | 890ms | 0% |

## Improvement Suggestions

### Performance
- [ ] Endpoint X has p95 > 500ms — add DB index on column Y
- [ ] Endpoint Z sends 2MB response — add pagination

### Security
- [ ] Missing rate limiting on /auth/login — add throttle
- [ ] CORS allows * — restrict to known origins
- [ ] No request size limit — add max body size

### Best Practices
- [ ] Missing Cache-Control headers on GET endpoints
- [ ] No ETag support for conditional requests
- [ ] Error responses inconsistent format — standardize { error, message, code }
- [ ] Missing X-Request-Id for request tracing
```

## API QA Checklist

### Functional
- [ ] All CRUD operations return correct status codes
- [ ] Validation errors return 400 with field-level messages
- [ ] Unauthorized returns 401, forbidden returns 403
- [ ] Not found returns 404
- [ ] Pagination works (page, limit, total, offset)
- [ ] Search/filter returns correct results
- [ ] Sorting works (asc/desc)

### Security
- [ ] Auth required endpoints reject unauthenticated requests
- [ ] Users can't access other users' data (IDOR check)
- [ ] SQL injection via search params doesn't work
- [ ] XSS in input fields gets sanitized
- [ ] Rate limiting on auth endpoints
- [ ] CORS properly configured
- [ ] Sensitive data not in URL params (use POST body)

### Performance
- [ ] All endpoints respond < 500ms under normal load
- [ ] List endpoints paginated (no unbounded queries)
- [ ] Large responses use compression (gzip)
- [ ] N+1 queries absent (check with DB logs)

### Reliability
- [ ] Graceful error handling (no stack traces to client)
- [ ] Consistent error response format
- [ ] Idempotent PUT/DELETE operations
- [ ] Concurrent writes handled (optimistic locking)

## Specific References

* **Hurl test patterns** [references/hurl-patterns.md](references/hurl-patterns.md)
* **Performance testing workflow** [references/performance-testing.md](references/performance-testing.md)
