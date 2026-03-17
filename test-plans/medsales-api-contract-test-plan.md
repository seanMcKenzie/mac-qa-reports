# MedSales API Contract Test Plan
**Document:** QA-03  
**Author:** Mac McDonald, QA Engineer  
**Date:** 2026-03-10  
**Project:** Medical Sales Rep Platform — Spring Boot Backend  
**Status:** Draft  

---

## Overview

This document defines the expected REST API contracts for the MedSales Spring Boot backend. Every endpoint is documented with its full contract: HTTP method, path, auth requirement, request parameters/body schema, success response shapes, and all error cases.

**This is the source of truth for API contract testing.** If the implementation deviates from this document, that's a bug. I don't care if it was "intentional" — if it wasn't documented and agreed-upon, it's a bug.

---

## Global Conventions

### Authentication
All endpoints require a **JWT Bearer token** in the `Authorization` header:
```
Authorization: Bearer <jwt_token>
```
Missing or invalid tokens → `401 Unauthorized`.  
Expired tokens → `401 Unauthorized`.

### Org ID Enforcement (**CRITICAL**)
Every authenticated request carries an `org_id` claim in the JWT. **All data queries MUST filter by the authenticated user's `org_id`.**  
- If a resource belongs to a different org → `403 Forbidden` (never `404` — we don't leak existence)  
- No cross-org data leakage is acceptable. Ever. This is a security requirement, not a preference.

### Pagination Contract
All list endpoints that support pagination share this contract:

**Request params:**
| Param | Type    | Default | Max  | Description          |
|-------|---------|---------|------|----------------------|
| page  | integer | 0       | —    | Zero-based page index |
| size  | integer | 20      | 100  | Items per page        |

**Response envelope (paginated):**
```json
{
  "content": [ ... ],
  "page": 0,
  "size": 20,
  "totalElements": 150,
  "totalPages": 8,
  "first": true,
  "last": false
}
```

### Error Response Shape
All error responses use this structure:
```json
{
  "status": 400,
  "error": "Bad Request",
  "message": "Human-readable description of the error",
  "path": "/api/v1/...",
  "timestamp": "2026-03-10T17:08:00.000Z"
}
```

### Content Type
All requests and responses: `Content-Type: application/json`

---

## Endpoints

---

## 1. Physician Search

### `GET /api/v1/physicians/search`

Search the physician directory. Supports keyword search, geographic filtering, and specialty filtering.

#### Auth
- **Required:** Yes — JWT Bearer
- **Org enforcement:** Results are scoped to the authenticated user's org territory configuration. No cross-org physician data is returned.

#### Request Parameters

| Param         | Type    | Required | Constraints                               | Description                        |
|---------------|---------|----------|-------------------------------------------|------------------------------------|
| `q`           | string  | No       | Max 200 chars                             | Keyword search (name, NPI, clinic) |
| `state`       | string  | No       | 2-letter uppercase US state code (e.g. `TX`) | Filter by state                 |
| `specialty`   | string  | No       | Max 100 chars                             | Filter by specialty (e.g. `Cardiology`) |
| `lat`         | double  | No*      | -90.0 to 90.0                             | Latitude for geo search            |
| `lng`         | double  | No*      | -180.0 to 180.0                           | Longitude for geo search           |
| `radius_miles`| double  | No*      | 0.1 to 500.0                              | Radius for geo search              |
| `page`        | integer | No       | ≥ 0                                       | Page index (default: 0)            |
| `size`        | integer | No       | 1–100                                     | Page size (default: 20)            |

> *`lat`, `lng`, and `radius_miles` must ALL be provided together or not at all.

#### Success Response — `200 OK`
```json
{
  "content": [
    {
      "npi": "1234567890",
      "firstName": "Jane",
      "lastName": "Smith",
      "credential": "MD",
      "specialty": "Cardiology",
      "practiceAddress": {
        "street1": "123 Medical Blvd",
        "street2": null,
        "city": "Austin",
        "state": "TX",
        "zip": "78701"
      },
      "phone": "512-555-0100",
      "distanceMiles": 4.2
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 87,
  "totalPages": 5,
  "first": true,
  "last": false
}
```
> `distanceMiles` is only present when geo search params are provided. Null otherwise.

#### Error Cases

| Status | Condition | Response `message` |
|--------|-----------|--------------------|
| `400`  | `state` is not a valid 2-letter US state code | `"Invalid state code: '{value}'"` |
| `400`  | `lat` provided without `lng` or `radius_miles` | `"Geo search requires lat, lng, and radius_miles"` |
| `400`  | `lat` out of range (-90 to 90) | `"lat must be between -90.0 and 90.0"` |
| `400`  | `lng` out of range (-180 to 180) | `"lng must be between -180.0 and 180.0"` |
| `400`  | `radius_miles` ≤ 0 or > 500 | `"radius_miles must be between 0.1 and 500.0"` |
| `400`  | `size` > 100 | `"size must not exceed 100"` |
| `400`  | `page` < 0 | `"page must be ≥ 0"` |
| `401`  | Missing or invalid JWT | `"Unauthorized"` |
| `200`  | No results found | Returns empty `content: []`, `totalElements: 0` — NOT a 404 |

---

## 2. Physician Profile

### `GET /api/v1/physicians/{npi}`

Retrieve full profile for a single physician by NPI number.

#### Auth
- **Required:** Yes — JWT Bearer
- **Org enforcement:** Physician records are global (NPI is a national registry), but interaction history and enrichment data must be scoped to the authenticated user's `org_id`. If the physician exists but has no org-scoped data, a 200 with null CRM fields is returned — NOT a 403.

#### Path Parameters

| Param | Type   | Required | Constraints                  | Description        |
|-------|--------|----------|------------------------------|--------------------|
| `npi` | string | Yes      | Exactly 10 numeric digits    | National Provider Identifier |

#### Success Response — `200 OK`
```json
{
  "npi": "1234567890",
  "firstName": "Jane",
  "lastName": "Smith",
  "credential": "MD",
  "specialty": "Cardiology",
  "subSpecialty": "Interventional Cardiology",
  "gender": "F",
  "practiceAddress": {
    "street1": "123 Medical Blvd",
    "street2": "Suite 400",
    "city": "Austin",
    "state": "TX",
    "zip": "78701",
    "lat": 30.2672,
    "lng": -97.7431
  },
  "mailingAddress": {
    "street1": "PO Box 1234",
    "city": "Austin",
    "state": "TX",
    "zip": "78710"
  },
  "phone": "512-555-0100",
  "fax": "512-555-0101",
  "enumerationDate": "2005-06-15",
  "lastUpdated": "2024-11-01",
  "crmSummary": {
    "lastCallDate": "2026-02-14",
    "totalCalls": 12,
    "openTasks": 2,
    "lastSampleDrop": "2026-01-20"
  }
}
```
> `crmSummary` fields are org-scoped and may be null if no activity exists.

#### Error Cases

| Status | Condition | Response `message` |
|--------|-----------|--------------------|
| `400`  | `npi` is not exactly 10 numeric digits | `"NPI must be exactly 10 numeric digits"` |
| `401`  | Missing or invalid JWT | `"Unauthorized"` |
| `404`  | NPI not found in the national registry | `"Physician not found: {npi}"` |

---

## 3. List CRM Calls

### `GET /api/v1/crm/calls`

Retrieve a paginated list of logged sales calls. Filterable by physician, rep, and date range.

#### Auth
- **Required:** Yes — JWT Bearer
- **Org enforcement:** Returns only calls where `org_id` matches the authenticated user's org. A rep may only see calls assigned to reps within their org. If `rep_id` belongs to a different org → `403 Forbidden`.

#### Request Parameters

| Param    | Type    | Required | Constraints                       | Description                            |
|----------|---------|----------|-----------------------------------|----------------------------------------|
| `npi`    | string  | No       | 10 numeric digits                 | Filter by physician NPI                |
| `rep_id` | string  | No       | Valid UUID                        | Filter by sales rep ID                 |
| `from`   | string  | No       | ISO 8601 date (`YYYY-MM-DD`)      | Filter calls on or after this date     |
| `to`     | string  | No       | ISO 8601 date (`YYYY-MM-DD`)      | Filter calls on or before this date    |
| `page`   | integer | No       | ≥ 0                               | Page index (default: 0)                |
| `size`   | integer | No       | 1–100                             | Page size (default: 20)                |

> `from` must be ≤ `to` when both are provided.

#### Success Response — `200 OK`
```json
{
  "content": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "npi": "1234567890",
      "physicianName": "Jane Smith, MD",
      "repId": "rep-uuid-here",
      "repName": "Bobby Boucher",
      "callType": "IN_PERSON",
      "callDate": "2026-02-14",
      "durationMin": 15,
      "notes": "Discussed new product line.",
      "outcome": "INTERESTED",
      "orgId": "org-uuid-here",
      "createdAt": "2026-02-14T14:30:00Z",
      "updatedAt": "2026-02-14T14:30:00Z"
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 47,
  "totalPages": 3,
  "first": true,
  "last": false
}
```

#### Error Cases

| Status | Condition | Response `message` |
|--------|-----------|--------------------|
| `400`  | `npi` is not 10 numeric digits | `"NPI must be exactly 10 numeric digits"` |
| `400`  | `rep_id` is not a valid UUID | `"rep_id must be a valid UUID"` |
| `400`  | `from` is not a valid ISO date | `"from must be a valid date in YYYY-MM-DD format"` |
| `400`  | `to` is not a valid ISO date | `"to must be a valid date in YYYY-MM-DD format"` |
| `400`  | `from` is after `to` | `"from must be before or equal to to"` |
| `400`  | `size` > 100 | `"size must not exceed 100"` |
| `401`  | Missing or invalid JWT | `"Unauthorized"` |
| `403`  | `rep_id` belongs to a different org | `"Access denied: rep does not belong to your organization"` |
| `200`  | No calls match filters | Returns empty `content: []` — NOT a 404 |

---

## 4. Log a Call

### `POST /api/v1/crm/calls`

Create a new CRM call record for a physician interaction.

#### Auth
- **Required:** Yes — JWT Bearer
- **Org enforcement:** The call is automatically associated with the authenticated user's `org_id`. The `npi` in the request body must resolve to a valid physician. The `rep_id` (if specified separately — otherwise defaults to the authenticated user) must belong to the same org → `403` if mismatch.

#### Request Body

```json
{
  "npi": "1234567890",
  "callType": "IN_PERSON",
  "callDate": "2026-02-14",
  "durationMin": 15,
  "notes": "Discussed new product line. Left samples.",
  "outcome": "INTERESTED"
}
```

#### Field Constraints

| Field        | Type    | Required | Constraints                                                             |
|--------------|---------|----------|-------------------------------------------------------------------------|
| `npi`        | string  | Yes      | Exactly 10 numeric digits; must exist in physician registry             |
| `callType`   | string  | Yes      | Enum: `IN_PERSON`, `PHONE`, `VIDEO`, `EMAIL`                            |
| `callDate`   | string  | Yes      | ISO 8601 date (`YYYY-MM-DD`); cannot be in the future                   |
| `durationMin`| integer | No       | 1–480 (max 8 hours); required if `callType` is `IN_PERSON`, `PHONE`, or `VIDEO` |
| `notes`      | string  | No       | Max 2000 characters                                                     |
| `outcome`    | string  | Yes      | Enum: `INTERESTED`, `NOT_INTERESTED`, `FOLLOW_UP`, `SAMPLE_REQUESTED`, `NO_CONTACT`, `LEFT_MESSAGE` |

#### Success Response — `201 Created`
```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "npi": "1234567890",
  "physicianName": "Jane Smith, MD",
  "repId": "rep-uuid-here",
  "repName": "Bobby Boucher",
  "callType": "IN_PERSON",
  "callDate": "2026-02-14",
  "durationMin": 15,
  "notes": "Discussed new product line. Left samples.",
  "outcome": "INTERESTED",
  "orgId": "org-uuid-here",
  "createdAt": "2026-03-10T17:08:00Z",
  "updatedAt": "2026-03-10T17:08:00Z"
}
```
> Response includes `Location` header: `/api/v1/crm/calls/{id}`

#### Error Cases

| Status | Condition | Response `message` |
|--------|-----------|--------------------|
| `400`  | `npi` missing or not 10 numeric digits | `"npi is required and must be exactly 10 numeric digits"` |
| `400`  | `callType` missing or invalid enum value | `"callType must be one of: IN_PERSON, PHONE, VIDEO, EMAIL"` |
| `400`  | `callDate` missing or invalid format | `"callDate is required and must be YYYY-MM-DD"` |
| `400`  | `callDate` is in the future | `"callDate cannot be in the future"` |
| `400`  | `durationMin` missing when required by `callType` | `"durationMin is required for IN_PERSON, PHONE, and VIDEO calls"` |
| `400`  | `durationMin` < 1 or > 480 | `"durationMin must be between 1 and 480"` |
| `400`  | `notes` > 2000 characters | `"notes must not exceed 2000 characters"` |
| `400`  | `outcome` missing or invalid enum value | `"outcome must be one of: INTERESTED, NOT_INTERESTED, FOLLOW_UP, SAMPLE_REQUESTED, NO_CONTACT, LEFT_MESSAGE"` |
| `401`  | Missing or invalid JWT | `"Unauthorized"` |
| `404`  | `npi` not found in physician registry | `"Physician not found: {npi}"` |

---

## 5. Create Task

### `POST /api/v1/crm/tasks`

Create a new follow-up task associated with a physician.

#### Auth
- **Required:** Yes — JWT Bearer
- **Org enforcement:** Task is automatically tagged with the authenticated user's `org_id`. Task is assigned to the authenticated user by default unless `assigneeId` is provided — `assigneeId` must belong to same org → `403` if mismatch.

#### Request Body

```json
{
  "npi": "1234567890",
  "title": "Follow up re: sample request",
  "dueDate": "2026-03-20",
  "priority": "HIGH",
  "notes": "Dr. Smith asked about efficacy data. Send whitepaper."
}
```

#### Field Constraints

| Field     | Type   | Required | Constraints                                          |
|-----------|--------|----------|------------------------------------------------------|
| `npi`     | string | Yes      | Exactly 10 numeric digits; must exist in registry    |
| `title`   | string | Yes      | 1–255 characters                                     |
| `dueDate` | string | Yes      | ISO 8601 date (`YYYY-MM-DD`); must be today or future |
| `priority`| string | Yes      | Enum: `LOW`, `MEDIUM`, `HIGH`, `URGENT`              |
| `notes`   | string | No       | Max 1000 characters                                  |

#### Success Response — `201 Created`
```json
{
  "id": "task-uuid-here",
  "npi": "1234567890",
  "physicianName": "Jane Smith, MD",
  "title": "Follow up re: sample request",
  "dueDate": "2026-03-20",
  "priority": "HIGH",
  "status": "OPEN",
  "notes": "Dr. Smith asked about efficacy data. Send whitepaper.",
  "assigneeId": "rep-uuid-here",
  "assigneeName": "Bobby Boucher",
  "orgId": "org-uuid-here",
  "createdAt": "2026-03-10T17:08:00Z",
  "updatedAt": "2026-03-10T17:08:00Z"
}
```
> Response includes `Location` header: `/api/v1/crm/tasks/{id}`  
> `status` is always `OPEN` on creation.

#### Error Cases

| Status | Condition | Response `message` |
|--------|-----------|--------------------|
| `400`  | `npi` missing or not 10 numeric digits | `"npi is required and must be exactly 10 numeric digits"` |
| `400`  | `title` missing or empty | `"title is required"` |
| `400`  | `title` > 255 characters | `"title must not exceed 255 characters"` |
| `400`  | `dueDate` missing or invalid format | `"dueDate is required and must be YYYY-MM-DD"` |
| `400`  | `dueDate` is in the past | `"dueDate must be today or a future date"` |
| `400`  | `priority` missing or invalid enum | `"priority must be one of: LOW, MEDIUM, HIGH, URGENT"` |
| `400`  | `notes` > 1000 characters | `"notes must not exceed 1000 characters"` |
| `401`  | Missing or invalid JWT | `"Unauthorized"` |
| `403`  | `assigneeId` belongs to different org | `"Access denied: assignee does not belong to your organization"` |
| `404`  | `npi` not found in physician registry | `"Physician not found: {npi}"` |

---

## 6. List Tasks

### `GET /api/v1/crm/tasks`

Retrieve CRM tasks. Filterable by physician, status, and rep.

#### Auth
- **Required:** Yes — JWT Bearer
- **Org enforcement:** Returns only tasks where `org_id` matches the authenticated user's org. If `rep_id` belongs to a different org → `403 Forbidden`.

#### Request Parameters

| Param    | Type   | Required | Constraints                                        | Description              |
|----------|--------|----------|----------------------------------------------------|--------------------------|
| `npi`    | string | No       | 10 numeric digits                                  | Filter by physician NPI  |
| `status` | string | No       | Enum: `OPEN`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED` | Filter by task status |
| `rep_id` | string | No       | Valid UUID                                         | Filter by assignee rep   |

> No pagination required per spec, but implementation SHOULD support `page`/`size` for large result sets (>100 tasks). If implemented, follows the standard pagination contract.

#### Success Response — `200 OK`
```json
{
  "tasks": [
    {
      "id": "task-uuid-here",
      "npi": "1234567890",
      "physicianName": "Jane Smith, MD",
      "title": "Follow up re: sample request",
      "dueDate": "2026-03-20",
      "priority": "HIGH",
      "status": "OPEN",
      "notes": "Dr. Smith asked about efficacy data. Send whitepaper.",
      "assigneeId": "rep-uuid-here",
      "assigneeName": "Bobby Boucher",
      "orgId": "org-uuid-here",
      "createdAt": "2026-03-10T17:08:00Z",
      "updatedAt": "2026-03-10T17:08:00Z"
    }
  ],
  "total": 14
}
```

#### Error Cases

| Status | Condition | Response `message` |
|--------|-----------|--------------------|
| `400`  | `npi` is not 10 numeric digits | `"NPI must be exactly 10 numeric digits"` |
| `400`  | `status` is not a valid enum value | `"status must be one of: OPEN, IN_PROGRESS, COMPLETED, CANCELLED"` |
| `400`  | `rep_id` is not a valid UUID | `"rep_id must be a valid UUID"` |
| `401`  | Missing or invalid JWT | `"Unauthorized"` |
| `403`  | `rep_id` belongs to a different org | `"Access denied: rep does not belong to your organization"` |
| `200`  | No tasks match filters | Returns `tasks: []`, `total: 0` — NOT a 404 |

---

## 7. Log Sample Drop

### `POST /api/v1/crm/samples`

Record a pharmaceutical sample drop to a physician's practice.

#### Auth
- **Required:** Yes — JWT Bearer
- **Org enforcement:** Sample record is automatically tagged with the authenticated user's `org_id`. The authenticated user is the rep by default.

#### Request Body

```json
{
  "npi": "1234567890",
  "productName": "Cardivex 10mg",
  "quantity": 5,
  "lotNumber": "LOT-2026-0042",
  "dropDate": "2026-03-10"
}
```

#### Field Constraints

| Field         | Type    | Required | Constraints                                         |
|---------------|---------|----------|-----------------------------------------------------|
| `npi`         | string  | Yes      | Exactly 10 numeric digits; must exist in registry   |
| `productName` | string  | Yes      | 1–255 characters                                    |
| `quantity`    | integer | Yes      | 1–1000                                              |
| `lotNumber`   | string  | Yes      | 1–100 characters; alphanumeric and hyphens only     |
| `dropDate`    | string  | Yes      | ISO 8601 date (`YYYY-MM-DD`); cannot be in the future |

#### Success Response — `201 Created`
```json
{
  "id": "sample-uuid-here",
  "npi": "1234567890",
  "physicianName": "Jane Smith, MD",
  "productName": "Cardivex 10mg",
  "quantity": 5,
  "lotNumber": "LOT-2026-0042",
  "dropDate": "2026-03-10",
  "repId": "rep-uuid-here",
  "repName": "Bobby Boucher",
  "orgId": "org-uuid-here",
  "createdAt": "2026-03-10T17:08:00Z"
}
```
> Response includes `Location` header: `/api/v1/crm/samples/{id}`

#### Error Cases

| Status | Condition | Response `message` |
|--------|-----------|--------------------|
| `400`  | `npi` missing or not 10 numeric digits | `"npi is required and must be exactly 10 numeric digits"` |
| `400`  | `productName` missing or empty | `"productName is required"` |
| `400`  | `productName` > 255 characters | `"productName must not exceed 255 characters"` |
| `400`  | `quantity` missing, < 1, or > 1000 | `"quantity must be between 1 and 1000"` |
| `400`  | `quantity` is not an integer | `"quantity must be an integer"` |
| `400`  | `lotNumber` missing or empty | `"lotNumber is required"` |
| `400`  | `lotNumber` > 100 characters | `"lotNumber must not exceed 100 characters"` |
| `400`  | `lotNumber` contains invalid characters | `"lotNumber may only contain alphanumeric characters and hyphens"` |
| `400`  | `dropDate` missing or invalid format | `"dropDate is required and must be YYYY-MM-DD"` |
| `400`  | `dropDate` is in the future | `"dropDate cannot be in the future"` |
| `401`  | Missing or invalid JWT | `"Unauthorized"` |
| `404`  | `npi` not found in physician registry | `"Physician not found: {npi}"` |

---

## 8. Get Opportunities

### `GET /api/v1/crm/opportunities`

Retrieve CRM opportunity records for pipeline tracking. Filterable by stage and rep.

#### Auth
- **Required:** Yes — JWT Bearer
- **Org enforcement:** Returns only opportunities where `org_id` matches the authenticated user's org. If `rep_id` belongs to a different org → `403 Forbidden`.

#### Request Parameters

| Param    | Type   | Required | Constraints                                              | Description              |
|----------|--------|----------|----------------------------------------------------------|--------------------------|
| `stage`  | string | No       | Enum: `PROSPECT`, `ENGAGED`, `SAMPLE_DROPPED`, `PRESCRIBING`, `LOST` | Filter by pipeline stage |
| `rep_id` | string | No       | Valid UUID                                               | Filter by sales rep      |

> No pagination required per spec, but implementation SHOULD support `page`/`size`. If implemented, follows the standard pagination contract.

#### Success Response — `200 OK`
```json
{
  "opportunities": [
    {
      "id": "opp-uuid-here",
      "npi": "1234567890",
      "physicianName": "Jane Smith, MD",
      "specialty": "Cardiology",
      "stage": "SAMPLE_DROPPED",
      "repId": "rep-uuid-here",
      "repName": "Bobby Boucher",
      "lastActivityDate": "2026-03-10",
      "nextFollowUpDate": "2026-03-20",
      "notes": "Strong interest in Cardivex line.",
      "orgId": "org-uuid-here",
      "createdAt": "2026-01-15T09:00:00Z",
      "updatedAt": "2026-03-10T17:08:00Z"
    }
  ],
  "total": 6
}
```

#### Error Cases

| Status | Condition | Response `message` |
|--------|-----------|--------------------|
| `400`  | `stage` is not a valid enum value | `"stage must be one of: PROSPECT, ENGAGED, SAMPLE_DROPPED, PRESCRIBING, LOST"` |
| `400`  | `rep_id` is not a valid UUID | `"rep_id must be a valid UUID"` |
| `401`  | Missing or invalid JWT | `"Unauthorized"` |
| `403`  | `rep_id` belongs to a different org | `"Access denied: rep does not belong to your organization"` |
| `200`  | No opportunities match filters | Returns `opportunities: []`, `total: 0` — NOT a 404 |

---

## Security Test Cases (Cross-Cutting)

These test cases apply across ALL endpoints and must be verified:

### Auth Tests
1. **No token** → All endpoints return `401`
2. **Malformed token** (not a valid JWT) → `401`
3. **Expired token** → `401`
4. **Tampered token** (signature invalid) → `401`
5. **Valid token, wrong service** (audience mismatch) → `401`

### Org Isolation Tests
1. **Rep from Org A authenticates, requests data belonging to Org B** → `403` (never `200`, never `404`)
2. **Rep from Org A searches physicians** — must not see calls/tasks/samples from Org B, even for the same physician NPI
3. **Rep filters by `rep_id` that belongs to Org B** → `403`
4. **No org_id in JWT claims** → `403` or `401` (must not default to a permissive org)

### Input Validation Tests
1. SQL injection attempt in `q`, `notes`, any string field → `400` or sanitized/escaped (never executed)
2. XSS payload in `notes`, `title` fields → must be stored/returned escaped
3. Extremely large payloads (notes at exactly 2001 chars) → `400`
4. Numeric fields sent as strings → `400` with clear message
5. Enum fields with mixed casing (e.g., `"in_person"` vs `"IN_PERSON"`) → document expected behavior (reject or normalize — pick one, be consistent)

---

## Contract Test Matrix

| Endpoint | Happy Path | 400 Validation | 401 Auth | 403 Org | 404 Not Found | Pagination |
|----------|-----------|---------------|---------|---------|--------------|-----------|
| GET /physicians/search | ✅ | ✅ | ✅ | ✅ | N/A | ✅ |
| GET /physicians/{npi} | ✅ | ✅ | ✅ | N/A | ✅ | N/A |
| GET /crm/calls | ✅ | ✅ | ✅ | ✅ | N/A | ✅ |
| POST /crm/calls | ✅ | ✅ | ✅ | N/A | ✅ | N/A |
| POST /crm/tasks | ✅ | ✅ | ✅ | ✅ | ✅ | N/A |
| GET /crm/tasks | ✅ | ✅ | ✅ | ✅ | N/A | Optional |
| POST /crm/samples | ✅ | ✅ | ✅ | N/A | ✅ | N/A |
| GET /crm/opportunities | ✅ | ✅ | ✅ | ✅ | N/A | Optional |

---

## Implementation Notes for the Dev Team

1. **Org ID is non-negotiable.** It must come from the JWT — not from a request parameter. If it comes from a request param, that's a security vulnerability. That's a critical bug. I will file it.

2. **Enum validation must be case-sensitive** unless we explicitly document otherwise. `"in_person"` ≠ `"IN_PERSON"`. Pick a convention and stick to it.

3. **Date validations must be server-side.** Client-side validation is nice but it is not a contract. The server is the contract.

4. **Error messages must not leak stack traces or internal details** in production. The `message` field should be human-readable and safe to display. No `java.lang.NullPointerException` in API responses. If I see that, I'm filing a critical.

5. **Location headers on 201s.** Every `POST` that creates a resource must return a `Location` header pointing to the created resource. This is standard REST. We follow the standard.

6. **Empty result sets are 200, not 404.** A search that finds nothing is a successful search. Don't confuse "no data" with "not found."

---

*— Mac McDonald, QA Engineer. I test everything. I find everything. No exceptions.*
