# MedSales CRM — Tenant Isolation Test Plan

**Task ID:** QA-02  
**Author:** Mac McDonald, QA Engineer  
**Date:** 2026-03-10  
**Status:** Draft  
**Severity Note:** Multi-tenancy failures are CRITICAL — data leakage between orgs is a catastrophic SaaS failure. Every test in this plan is high-stakes. I am treating all isolation tests as P0.

---

## 1. Overview

The MedSales CRM is a multi-tenant SaaS application. Every CRM table (`crm_reps`, `crm_calls`, `crm_tasks`, `crm_opportunities`, `crm_samples`) includes an `org_id` column that is a foreign key to the `organizations` table. The contract is simple: **a rep from Org A must never see, create, modify, or delete data belonging to Org B.** This test plan systematically proves that contract is enforced — or breaks it trying.

### Tables Under Test

| Table | PK | Tenant Key | Rep Link |
|---|---|---|---|
| `crm_reps` | `rep_id` | `org_id` | self |
| `crm_calls` | `call_id` | `org_id` | `rep_id` |
| `crm_tasks` | `task_id` | `org_id` | `rep_id` |
| `crm_opportunities` | `opportunity_id` | `org_id` | `rep_id` |
| `crm_samples` | `sample_id` | `org_id` | `rep_id` |

### Test Fixtures Required

```
Org A (org_id: ORG_A_UUID)
  └── Rep A1 (rep_id: REP_A1_UUID, role: 'rep')
  └── Rep A2 (rep_id: REP_A2_UUID, role: 'rep')
  └── Admin A (rep_id: ADMIN_A_UUID, role: 'admin')

Org B (org_id: ORG_B_UUID)
  └── Rep B1 (rep_id: REP_B1_UUID, role: 'rep')

Shared Physician (npi: '1234567893')  ← valid NPI, no tenant
```

---

## 2. Section 1 — org_id Isolation: Read Operations

> A rep from Org A MUST NOT be able to read any CRM data belonging to Org B.

---

### TC-ISO-001 — Rep cannot list another org's reps

**Table:** `crm_reps`  
**Operation:** READ (list)  
**Actor:** Rep A1 (authenticated, org_id = ORG_A_UUID)

**Precondition:** Rep B1 exists in `crm_reps` with `org_id = ORG_B_UUID`

**Steps:**
1. Authenticate as Rep A1
2. Send `GET /api/v1/crm/reps` (or equivalent list endpoint)
3. Inspect response

**Expected:**
- Response contains ONLY reps where `org_id = ORG_A_UUID`
- Rep B1 does NOT appear in the response
- HTTP 200 with filtered result set

**Failure Condition (CRITICAL BUG):** Rep B1 appears in the response

---

### TC-ISO-002 — Rep cannot fetch another org's rep by ID

**Table:** `crm_reps`  
**Operation:** READ (by ID)  
**Actor:** Rep A1

**Steps:**
1. Authenticate as Rep A1
2. Send `GET /api/v1/crm/reps/{REP_B1_UUID}`
3. Inspect response

**Expected:**
- HTTP 403 Forbidden OR HTTP 404 Not Found
- Response body: error message (no rep data)

**Failure Condition (CRITICAL BUG):** HTTP 200 with Rep B1's data returned

---

### TC-ISO-003 — Rep cannot list another org's calls

**Table:** `crm_calls`  
**Operation:** READ (list)  
**Actor:** Rep A1

**Precondition:** Org B has call records in `crm_calls`

**Steps:**
1. Authenticate as Rep A1
2. Send `GET /api/v1/crm/calls`
3. Inspect response

**Expected:**
- All returned calls have `org_id = ORG_A_UUID`
- Zero calls from Org B present

**Failure Condition (CRITICAL BUG):** Any call with `org_id = ORG_B_UUID` returned

---

### TC-ISO-004 — Rep cannot fetch another org's call by ID

**Table:** `crm_calls`  
**Operation:** READ (by ID)  
**Actor:** Rep A1

**Steps:**
1. Authenticate as Rep A1
2. Send `GET /api/v1/crm/calls/{ORG_B_CALL_ID}`

**Expected:** HTTP 403 or HTTP 404 — no call data returned

---

### TC-ISO-005 — Rep cannot list another org's tasks

**Table:** `crm_tasks`  
**Operation:** READ (list)  
**Actor:** Rep A1

**Steps:**
1. Authenticate as Rep A1
2. `GET /api/v1/crm/tasks`

**Expected:** Only tasks where `org_id = ORG_A_UUID` returned

---

### TC-ISO-006 — Rep cannot fetch another org's task by ID

**Table:** `crm_tasks`  
**Actor:** Rep A1

**Steps:**
1. `GET /api/v1/crm/tasks/{ORG_B_TASK_ID}`

**Expected:** HTTP 403 or 404

---

### TC-ISO-007 — Rep cannot list another org's opportunities

**Table:** `crm_opportunities`  
**Actor:** Rep A1

**Steps:**
1. `GET /api/v1/crm/opportunities`

**Expected:** Only opportunities where `org_id = ORG_A_UUID`

---

### TC-ISO-008 — Rep cannot fetch another org's opportunity by ID

**Table:** `crm_opportunities`  
**Actor:** Rep A1

**Steps:**
1. `GET /api/v1/crm/opportunities/{ORG_B_OPP_ID}`

**Expected:** HTTP 403 or 404

---

### TC-ISO-009 — Rep cannot list another org's samples

**Table:** `crm_samples`  
**Actor:** Rep A1

**Steps:**
1. `GET /api/v1/crm/samples`

**Expected:** Only samples where `org_id = ORG_A_UUID`

---

### TC-ISO-010 — Rep cannot fetch another org's sample by ID

**Table:** `crm_samples`  
**Actor:** Rep A1

**Steps:**
1. `GET /api/v1/crm/samples/{ORG_B_SAMPLE_ID}`

**Expected:** HTTP 403 or 404

---

## 3. Section 2 — org_id Isolation: Write Operations

> A rep from Org A MUST NOT be able to create, update, or delete data in Org B's tenant space.

---

### TC-ISO-011 — Rep cannot create a call record under another org

**Table:** `crm_calls`  
**Operation:** CREATE  
**Actor:** Rep A1

**Steps:**
1. Authenticate as Rep A1
2. POST `{"org_id": "ORG_B_UUID", "npi": "...", "call_type": "phone", ...}` to `/api/v1/crm/calls`

**Expected:**
- HTTP 403 Forbidden — org_id in payload does not match authenticated rep's org
- OR: server ignores payload `org_id` and always uses auth token's org — record is saved under ORG_A_UUID, not ORG_B_UUID

**Failure Condition (CRITICAL BUG):** Record created in DB with `org_id = ORG_B_UUID`

---

### TC-ISO-012 — Rep cannot update another org's call record

**Table:** `crm_calls`  
**Operation:** UPDATE  
**Actor:** Rep A1

**Steps:**
1. Authenticate as Rep A1
2. `PATCH /api/v1/crm/calls/{ORG_B_CALL_ID}` with `{"notes": "injected by cross-tenant attack"}`

**Expected:** HTTP 403 or 404

**Failure Condition (CRITICAL BUG):** HTTP 200 — Org B's call record was modified

---

### TC-ISO-013 — Rep cannot delete another org's call record

**Table:** `crm_calls`  
**Operation:** DELETE  
**Actor:** Rep A1

**Steps:**
1. Authenticate as Rep A1
2. `DELETE /api/v1/crm/calls/{ORG_B_CALL_ID}`

**Expected:** HTTP 403 or 404

**Failure Condition (CRITICAL BUG):** HTTP 200/204 — Org B's record deleted

---

### TC-ISO-014 — Rep cannot create a task for another org

**Table:** `crm_tasks`  
**Actor:** Rep A1  
**Steps:** POST to `/api/v1/crm/tasks` with `org_id = ORG_B_UUID`  
**Expected:** 403 or record saved under ORG_A_UUID

---

### TC-ISO-015 — Rep cannot update another org's task

**Table:** `crm_tasks`  
**Actor:** Rep A1  
**Steps:** `PATCH /api/v1/crm/tasks/{ORG_B_TASK_ID}`  
**Expected:** 403 or 404

---

### TC-ISO-016 — Rep cannot delete another org's task

**Table:** `crm_tasks`  
**Actor:** Rep A1  
**Steps:** `DELETE /api/v1/crm/tasks/{ORG_B_TASK_ID}`  
**Expected:** 403 or 404

---

### TC-ISO-017 — Rep cannot create an opportunity for another org

**Table:** `crm_opportunities`  
**Actor:** Rep A1  
**Steps:** POST to `/api/v1/crm/opportunities` with `org_id = ORG_B_UUID`  
**Expected:** 403 or record saved under ORG_A_UUID

---

### TC-ISO-018 — Rep cannot update another org's opportunity

**Actor:** Rep A1  
**Steps:** `PATCH /api/v1/crm/opportunities/{ORG_B_OPP_ID}`  
**Expected:** 403 or 404

---

### TC-ISO-019 — Rep cannot delete another org's opportunity

**Actor:** Rep A1  
**Steps:** `DELETE /api/v1/crm/opportunities/{ORG_B_OPP_ID}`  
**Expected:** 403 or 404

---

### TC-ISO-020 — Rep cannot log a sample drop for another org

**Table:** `crm_samples`  
**Actor:** Rep A1  
**Steps:** POST to `/api/v1/crm/samples` with `org_id = ORG_B_UUID`  
**Expected:** 403 or record saved under ORG_A_UUID

---

### TC-ISO-021 — Rep cannot update another org's sample record

**Actor:** Rep A1  
**Steps:** `PATCH /api/v1/crm/samples/{ORG_B_SAMPLE_ID}`  
**Expected:** 403 or 404

---

### TC-ISO-022 — Rep cannot delete another org's sample record

**Actor:** Rep A1  
**Steps:** `DELETE /api/v1/crm/samples/{ORG_B_SAMPLE_ID}`  
**Expected:** 403 or 404

---

## 4. Section 3 — CRM CRUD Operations (Happy Path, All 5 Tables)

> Verify all CRUD operations work correctly for authenticated, in-tenant reps.

---

### 4.1 crm_reps CRUD

#### TC-CRUD-001 — Create a rep (admin only)

**Actor:** Admin A  
**Steps:**
1. Authenticate as Admin A
2. `POST /api/v1/crm/reps` with `{ "email": "newrep@orga.com", "full_name": "New Rep", "territory": "MN", "role": "rep" }`

**Expected:**
- HTTP 201 Created
- Response body contains `rep_id` (UUID), `org_id = ORG_A_UUID`, `is_active: true`
- `email` matches input, `role = "rep"`, `created_at` is set

**Validation:**
- `email` uniqueness enforced — duplicate email returns 409 Conflict
- `org_id` auto-derived from auth token, not from request body
- `role` must be a valid value (not arbitrary strings)

---

#### TC-CRUD-002 — Read rep by ID

**Actor:** Rep A1  
**Steps:** `GET /api/v1/crm/reps/{REP_A1_UUID}`  
**Expected:** HTTP 200, full rep profile returned

---

#### TC-CRUD-003 — List reps in org

**Actor:** Rep A1  
**Steps:** `GET /api/v1/crm/reps`  
**Expected:** HTTP 200, array of reps all with `org_id = ORG_A_UUID`

---

#### TC-CRUD-004 — Update rep profile

**Actor:** Admin A  
**Steps:** `PATCH /api/v1/crm/reps/{REP_A1_UUID}` with `{ "territory": "WI" }`  
**Expected:** HTTP 200, `territory` updated, `org_id` unchanged

---

#### TC-CRUD-005 — Deactivate a rep

**Actor:** Admin A  
**Steps:** `PATCH /api/v1/crm/reps/{REP_A1_UUID}` with `{ "is_active": false }`  
**Expected:** HTTP 200, `is_active = false`; deactivated rep cannot authenticate

---

### 4.2 crm_calls CRUD

#### TC-CRUD-006 — Log a call (Create)

**Actor:** Rep A1  
**Steps:**
1. `POST /api/v1/crm/calls` with:
```json
{
  "npi": "1234567893",
  "call_type": "in-person",
  "call_date": "2026-03-10T14:00:00Z",
  "duration_min": 30,
  "outcome": "positive",
  "notes": "Discussed new product line"
}
```

**Expected:**
- HTTP 201
- Response: `call_id`, `org_id = ORG_A_UUID`, `rep_id = REP_A1_UUID`, all fields echoed
- `call_type` must be one of: `in-person`, `phone`, `virtual`

**Validation:**
- Invalid `call_type` → 400 Bad Request
- Missing required `npi` → 400 Bad Request
- Invalid/nonexistent `npi` → 400 or 422

---

#### TC-CRUD-007 — Get call by ID (Read)

**Actor:** Rep A1  
**Steps:** `GET /api/v1/crm/calls/{CALL_ID}`  
**Expected:** HTTP 200, full call record

---

#### TC-CRUD-008 — List calls for rep

**Actor:** Rep A1  
**Steps:** `GET /api/v1/crm/calls?rep_id={REP_A1_UUID}`  
**Expected:** HTTP 200, only Rep A1's calls

---

#### TC-CRUD-009 — List calls for a physician (by NPI)

**Actor:** Rep A1  
**Steps:** `GET /api/v1/crm/calls?npi=1234567893`  
**Expected:** HTTP 200, calls for that NPI scoped to Org A only

---

#### TC-CRUD-010 — Update a call record

**Actor:** Rep A1 (owns the call)  
**Steps:** `PATCH /api/v1/crm/calls/{CALL_ID}` with `{ "notes": "Updated notes", "outcome": "follow-up required" }`  
**Expected:** HTTP 200, fields updated

---

#### TC-CRUD-011 — Delete a call record

**Actor:** Rep A1 or Admin A  
**Steps:** `DELETE /api/v1/crm/calls/{CALL_ID}`  
**Expected:** HTTP 204 No Content, record removed from DB

---

### 4.3 crm_tasks CRUD

#### TC-CRUD-012 — Create a task

**Actor:** Rep A1  
**Steps:** `POST /api/v1/crm/tasks` with:
```json
{
  "npi": "1234567893",
  "title": "Schedule follow-up meeting",
  "due_date": "2026-03-20",
  "priority": "high"
}
```
**Expected:** HTTP 201, `status = "open"`, `org_id = ORG_A_UUID`

---

#### TC-CRUD-013 — Read a task by ID

**Actor:** Rep A1  
**Steps:** `GET /api/v1/crm/tasks/{TASK_ID}`  
**Expected:** HTTP 200, task data

---

#### TC-CRUD-014 — List tasks

**Actor:** Rep A1  
**Steps:** `GET /api/v1/crm/tasks`  
**Expected:** HTTP 200, tasks scoped to Org A

---

#### TC-CRUD-015 — Update task status

**Actor:** Rep A1  
**Steps:** `PATCH /api/v1/crm/tasks/{TASK_ID}` with `{ "status": "done" }`  
**Expected:** HTTP 200, `status = "done"`

**Validation:**
- `status` must be `open`, `done`, or `overdue` — invalid value returns 400

---

#### TC-CRUD-016 — Delete a task

**Actor:** Rep A1 or Admin A  
**Steps:** `DELETE /api/v1/crm/tasks/{TASK_ID}`  
**Expected:** HTTP 204

---

### 4.4 crm_opportunities CRUD

#### TC-CRUD-017 — Create an opportunity

**Actor:** Rep A1  
**Steps:** `POST /api/v1/crm/opportunities` with:
```json
{
  "npi": "1234567893",
  "product_name": "CardioMax 50mg",
  "stage": "discovery",
  "value": 12000.00,
  "close_date": "2026-06-30"
}
```
**Expected:** HTTP 201, `stage = "discovery"`, `org_id = ORG_A_UUID`

**Validation:**
- `stage` must be: `discovery`, `proposal`, `negotiation`, `closed-won`, `closed-lost`
- Invalid stage → 400

---

#### TC-CRUD-018 — Read opportunity by ID

**Actor:** Rep A1  
**Steps:** `GET /api/v1/crm/opportunities/{OPP_ID}`  
**Expected:** HTTP 200

---

#### TC-CRUD-019 — List opportunities

**Actor:** Rep A1  
**Steps:** `GET /api/v1/crm/opportunities`  
**Expected:** Org A opportunities only

---

#### TC-CRUD-020 — Advance opportunity stage

**Actor:** Rep A1  
**Steps:** `PATCH /api/v1/crm/opportunities/{OPP_ID}` with `{ "stage": "proposal" }`  
**Expected:** HTTP 200, `stage` updated

---

#### TC-CRUD-021 — Delete an opportunity

**Actor:** Admin A  
**Steps:** `DELETE /api/v1/crm/opportunities/{OPP_ID}`  
**Expected:** HTTP 204

---

### 4.5 crm_samples CRUD

#### TC-CRUD-022 — Log a sample drop (Create)

**Actor:** Rep A1  
**Steps:** `POST /api/v1/crm/samples` with:
```json
{
  "npi": "1234567893",
  "product_name": "CardioMax 50mg",
  "quantity": 5,
  "drop_date": "2026-03-10",
  "lot_number": "LOT-2026-001"
}
```
**Expected:** HTTP 201, `quantity = 5`, `org_id = ORG_A_UUID`

**Validation:**
- `quantity` must be ≥ 1 (INT, NOT NULL DEFAULT 1)
- `product_name` required (NOT NULL)
- `npi` must reference a valid physician

---

#### TC-CRUD-023 — Read sample by ID

**Actor:** Rep A1  
**Steps:** `GET /api/v1/crm/samples/{SAMPLE_ID}`  
**Expected:** HTTP 200, sample data

---

#### TC-CRUD-024 — List samples for a physician

**Actor:** Rep A1  
**Steps:** `GET /api/v1/crm/samples?npi=1234567893`  
**Expected:** HTTP 200, samples scoped to Org A for that physician

---

#### TC-CRUD-025 — Update a sample record

**Actor:** Rep A1  
**Steps:** `PATCH /api/v1/crm/samples/{SAMPLE_ID}` with `{ "notes": "corrected quantity" }`  
**Expected:** HTTP 200

---

#### TC-CRUD-026 — Delete a sample record

**Actor:** Admin A  
**Steps:** `DELETE /api/v1/crm/samples/{SAMPLE_ID}`  
**Expected:** HTTP 204

---

## 5. Section 4 — Rep User Role Enforcement

> The system defines roles (`rep`, `admin`). Certain operations must be restricted to admins.

---

### TC-ROLE-001 — Rep cannot create another rep

**Actor:** Rep A1 (role: 'rep')  
**Steps:**
1. Authenticate as Rep A1
2. `POST /api/v1/crm/reps` with new rep data

**Expected:** HTTP 403 Forbidden — only admins can create reps

---

### TC-ROLE-002 — Rep cannot deactivate another rep

**Actor:** Rep A1  
**Steps:** `PATCH /api/v1/crm/reps/{REP_A2_UUID}` with `{ "is_active": false }`  
**Expected:** HTTP 403

---

### TC-ROLE-003 — Rep cannot delete another rep's records (cross-rep)

**Actor:** Rep A1  
**Steps:** `DELETE /api/v1/crm/calls/{REP_A2_CALL_ID}` (call owned by Rep A2, same org)  
**Expected:** HTTP 403 — reps can only delete their own records (or admin permission required)

---

### TC-ROLE-004 — Admin can perform all CRUD operations within their org

**Actor:** Admin A  
**Steps:**
1. Create a rep → 201
2. Update any rep in Org A → 200
3. Delete any CRM record in Org A → 204

**Expected:** All operations succeed

---

### TC-ROLE-005 — Admin cannot perform operations in another org (even as admin)

**Actor:** Admin A  
**Steps:** `GET /api/v1/crm/calls` → must return only Org A calls, not Org B  
**Expected:** Admin role does NOT grant cross-tenant access

---

### TC-ROLE-006 — Unauthenticated request rejected on all CRM endpoints

**Actor:** No auth token

**Steps:**
1. `GET /api/v1/crm/calls` (no Authorization header)
2. `POST /api/v1/crm/calls` (no Authorization header)
3. `DELETE /api/v1/crm/calls/{ID}` (no Authorization header)

**Expected:** All return HTTP 401 Unauthorized

---

### TC-ROLE-007 — Expired/invalid token rejected

**Actor:** Rep A1 with malformed JWT

**Steps:** Any CRM endpoint with `Authorization: Bearer invalid.token.here`

**Expected:** HTTP 401

---

### TC-ROLE-008 — Rep from inactive org cannot authenticate

**Precondition:** Set `organizations.status = 'inactive'` for Org A  
**Actor:** Rep A1 (belongs to now-inactive org)  
**Steps:** Attempt to authenticate / call any endpoint  
**Expected:** HTTP 401 or 403 — inactive org access denied

---

### TC-ROLE-009 — Deactivated rep cannot authenticate

**Precondition:** `crm_reps.is_active = false` for Rep A1  
**Actor:** Rep A1  
**Steps:** Attempt to log in or call any API endpoint  
**Expected:** HTTP 401 or 403

---

## 6. Section 5 — Cross-Tenant Data Leakage Scenarios

> These are adversarial tests. I am attempting to find leakage. If any of these pass, it's a critical defect.

---

### TC-LEAK-001 — IDOR via predictable ID (Insecure Direct Object Reference)

**Attack:** Rep A1 guesses or observes a valid UUID from Org B and makes a direct request by ID.

**Steps:**
1. Rep A1 authenticates
2. Crafts `GET /api/v1/crm/calls/{KNOWN_ORG_B_UUID}` directly
3. Also tests: `GET /api/v1/crm/tasks/{KNOWN_ORG_B_UUID}`
4. Also tests: `GET /api/v1/crm/opportunities/{KNOWN_ORG_B_UUID}`
5. Also tests: `GET /api/v1/crm/samples/{KNOWN_ORG_B_UUID}`

**Expected:** HTTP 403 or 404 on ALL — never 200 with Org B data

**Note:** Server MUST validate that the requested record's `org_id` matches the auth token's org BEFORE returning data. 404 preferred over 403 to not confirm existence.

---

### TC-LEAK-002 — org_id parameter injection in list endpoint

**Attack:** Rep A1 passes `?org_id=ORG_B_UUID` as a query parameter to try to pull Org B's data.

**Steps:**
1. Authenticate as Rep A1
2. `GET /api/v1/crm/calls?org_id={ORG_B_UUID}`

**Expected:**
- Query parameter `org_id` is IGNORED — server always uses auth token's org
- Response contains only Org A data OR 400 if parameter is rejected

**Failure Condition (CRITICAL BUG):** Response contains Org B calls

---

### TC-LEAK-003 — org_id injection in request body (POST)

**Attack:** Rep A1 crafts a POST body with `org_id` set to Org B.

**Steps:**
1. Authenticate as Rep A1
2. POST `/api/v1/crm/calls` with `{ "org_id": "ORG_B_UUID", "npi": "...", ... }`

**Expected:**
- `org_id` in request body is IGNORED — server derives from auth token
- Record created with `org_id = ORG_A_UUID` only

**Failure Condition (CRITICAL BUG):** Record saved with `org_id = ORG_B_UUID`

---

### TC-LEAK-004 — Pagination boundary leakage

**Attack:** A paginated list endpoint exposes data from other tenants at page boundaries.

**Steps:**
1. Create 100 calls in Org A, 100 calls in Org B (interleaved timestamps)
2. Authenticate as Rep A1
3. Paginate through all pages of `GET /api/v1/crm/calls?page=1&size=10`, page=2, ... etc.
4. Check every response for `org_id = ORG_B_UUID`

**Expected:** Every single record on every page has `org_id = ORG_A_UUID`

---

### TC-LEAK-005 — Aggregate/count endpoint leakage

**Attack:** An analytics or summary endpoint leaks cross-tenant record counts.

**Steps:**
1. If any endpoint returns count/summary data (e.g., `GET /api/v1/crm/stats`)
2. Authenticate as Rep A1
3. Compare returned counts against DB counts for Org A only

**Expected:** Counts match Org A data only — Org B data NOT included

---

### TC-LEAK-006 — NPI-scoped query exposes cross-tenant data

**Attack:** Querying by NPI (which is a shared, non-tenant resource) leaks calls from another org for the same physician.

**Scenario:** Dr. Jane Smith (NPI: 1234567893) is called by both Rep A1 and Rep B1.

**Steps:**
1. Authenticate as Rep A1
2. `GET /api/v1/crm/calls?npi=1234567893`

**Expected:** Returns ONLY Org A's calls to Dr. Smith — Org B's calls invisible

**Failure Condition (CRITICAL BUG):** Rep A1 can see that Org B's reps also visited Dr. Smith (competitive intelligence leak)

---

### TC-LEAK-007 — Sort/filter parameter exposes ordering artifacts

**Attack:** Sorting by `created_at` DESC on a list endpoint exposes a record from another tenant because the WHERE clause is applied after ORDER BY.

**Steps:**
1. Org B creates a call record at T+1 (after all Org A records)
2. Authenticate as Rep A1
3. `GET /api/v1/crm/calls?sort=created_at&order=desc`

**Expected:** The Org B record (most recently created overall) does NOT appear — query scopes by `org_id` before sorting

---

### TC-LEAK-008 — HTTP response header leakage

**Attack:** Response headers (X-Total-Count, Link headers for pagination) expose total record counts across tenants.

**Steps:**
1. Authenticate as Rep A1
2. `GET /api/v1/crm/calls`
3. Inspect headers: `X-Total-Count`, `Content-Range`, `Link`

**Expected:** Count in headers matches Org A count only — not global DB count

---

### TC-LEAK-009 — Error message leakage

**Attack:** Error messages reveal the existence of another org's records.

**Steps:**
1. Authenticate as Rep A1
2. `GET /api/v1/crm/calls/{ORG_B_CALL_ID}`

**Expected:** HTTP 404 with generic message like `"Record not found"` — NOT `"Access denied to org ORG_B_UUID"` or `"Call exists but belongs to different organization"`

**Note:** Prefer 404 over 403 for non-existent-to-caller resources to avoid confirming existence.

---

### TC-LEAK-010 — Concurrent request race condition (tenant context bleeding)

**Attack:** Simultaneous requests from two different org reps share a thread/context that leaks tenant state.

**Steps:**
1. Fire 50 concurrent requests: 25 as Rep A1, 25 as Rep B1, all to `GET /api/v1/crm/calls`
2. Inspect all 50 responses

**Expected:** Every Rep A1 response contains only Org A data; every Rep B1 response contains only Org B data. Zero cross-contamination.

**Note:** This tests for thread-local or request-scoped tenant context bugs — a common failure mode in Spring Boot applications.

---

## 7. Test Matrix Summary

| Test ID | Table(s) | Category | Severity |
|---|---|---|---|
| TC-ISO-001 to TC-ISO-010 | All | Read Isolation | P0 Critical |
| TC-ISO-011 to TC-ISO-022 | All | Write Isolation | P0 Critical |
| TC-CRUD-001 to TC-CRUD-026 | All 5 | CRUD Happy Path | P1 High |
| TC-ROLE-001 to TC-ROLE-009 | crm_reps + auth | Role Enforcement | P1 High |
| TC-LEAK-001 to TC-LEAK-010 | All | Adversarial Leakage | P0 Critical |

**Total Test Cases:** 47

---

## 8. Pass/Fail Criteria

- **PASS:** All TC-ISO and TC-LEAK tests pass. Zero cross-tenant data visible.
- **FAIL (BLOCK RELEASE):** Any TC-ISO or TC-LEAK test fails. Multi-tenancy is broken. DO NOT SHIP.
- **FAIL (NON-BLOCKING):** TC-CRUD or TC-ROLE failures — bugs but not a release blocker pending severity review.

---

## 9. Notes from Mac

Listen — I cannot stress this enough. A single test failure in the ISO or LEAK categories is a **CRITICAL defect**. We are talking about Pharma company A's sales strategy being visible to Pharma company B. That is a lawsuit. That is an FDA situation. That is the company burning down.

I will test every one of these cases. I will not sign off on this feature until every single ISO and LEAK test passes. No exceptions. God gave me this instinct for a reason.

---

*End of QA-02 Test Plan*
