# Hurl Test Patterns

## Authentication Flow

```hurl
# Login
POST {{base_url}}/api/v1/auth/login
Content-Type: application/json
{
  "email": "{{email}}",
  "password": "{{password}}"
}
HTTP 200
[Captures]
token: jsonpath "$.token"
user_id: jsonpath "$.user.id"
[Asserts]
jsonpath "$.token" isString
jsonpath "$.token" matches "^eyJ"
jsonpath "$.user.email" == "{{email}}"

# Use token in subsequent requests
GET {{base_url}}/api/v1/profile
Authorization: Bearer {{token}}
HTTP 200
[Asserts]
jsonpath "$.id" == {{user_id}}
```

## Pagination Testing

```hurl
# Page 1
GET {{base_url}}/api/v1/patients?page=1&limit=10
Authorization: Bearer {{token}}
HTTP 200
[Captures]
total: jsonpath "$.total"
[Asserts]
jsonpath "$.data" count == 10
jsonpath "$.page" == 1
jsonpath "$.total" >= 10

# Page 2
GET {{base_url}}/api/v1/patients?page=2&limit=10
Authorization: Bearer {{token}}
HTTP 200
[Asserts]
jsonpath "$.data" count > 0
jsonpath "$.page" == 2

# Beyond last page
GET {{base_url}}/api/v1/patients?page=9999&limit=10
Authorization: Bearer {{token}}
HTTP 200
[Asserts]
jsonpath "$.data" count == 0
```

## Validation Testing

```hurl
# Missing required field
POST {{base_url}}/api/v1/patients
Authorization: Bearer {{token}}
Content-Type: application/json
{
  "last_name": "No-First-Name"
}
HTTP 400
[Asserts]
jsonpath "$.errors" count > 0
jsonpath "$.errors[0].field" == "first_name"

# Invalid email format
POST {{base_url}}/api/v1/users
Authorization: Bearer {{token}}
Content-Type: application/json
{
  "email": "not-an-email",
  "name": "Test"
}
HTTP 400
[Asserts]
jsonpath "$.errors[0].field" == "email"

# Empty body
POST {{base_url}}/api/v1/patients
Authorization: Bearer {{token}}
Content-Type: application/json
{}
HTTP 400
```

## File Upload Testing

```hurl
# Upload profile image
POST {{base_url}}/api/v1/patients/123/avatar
Authorization: Bearer {{token}}
[MultipartFormData]
file: file,test-avatar.jpg; image/jpeg
HTTP 200
[Asserts]
jsonpath "$.avatar_url" matches "^https?://"

# Upload wrong file type
POST {{base_url}}/api/v1/patients/123/avatar
Authorization: Bearer {{token}}
[MultipartFormData]
file: file,malicious.exe; application/octet-stream
HTTP 400
[Asserts]
jsonpath "$.error" contains "file type"
```

## Search & Filter Testing

```hurl
# Search by name
GET {{base_url}}/api/v1/patients?search=Ahmed
Authorization: Bearer {{token}}
HTTP 200
[Asserts]
jsonpath "$.data[0].first_name" contains "Ahmed"
jsonpath "$.total" >= 1

# Filter by status
GET {{base_url}}/api/v1/patients?status=active
Authorization: Bearer {{token}}
HTTP 200
[Asserts]
jsonpath "$.data[*].status" includes "active"

# Sort by created_at desc
GET {{base_url}}/api/v1/patients?sort=created_at&order=desc
Authorization: Bearer {{token}}
HTTP 200
[Asserts]
jsonpath "$.data" count > 1
```

## Error Response Consistency

```hurl
# 404
GET {{base_url}}/api/v1/patients/999999
Authorization: Bearer {{token}}
HTTP 404
[Asserts]
jsonpath "$.error" exists
jsonpath "$.message" isString

# 401
GET {{base_url}}/api/v1/patients
HTTP 401
[Asserts]
jsonpath "$.error" exists
jsonpath "$.message" isString

# 403
DELETE {{base_url}}/api/v1/admin/users/1
Authorization: Bearer {{nurse_token}}
HTTP 403
[Asserts]
jsonpath "$.error" exists
```

## Running with Variables

```bash
# Local development
hurl tests/api/*.hurl --test \
  --variable base_url=http://localhost:8080 \
  --variable email=doctor@hospital.com \
  --variable password=password123

# Staging
hurl tests/api/*.hurl --test \
  --variable base_url=https://staging-api.example.com \
  --variable email=test@hospital.com \
  --variable password=testpass123

# From env file
hurl tests/api/*.hurl --test --variables-file .env.test
```

## CI Pipeline Integration

```bash
# Run tests + generate reports
hurl tests/api/*.hurl \
  --test \
  --report-junit reports/api-test-results.xml \
  --report-html reports/api/ \
  --variable base_url=$API_URL \
  --variable email=$TEST_EMAIL \
  --variable password=$TEST_PASSWORD
```
