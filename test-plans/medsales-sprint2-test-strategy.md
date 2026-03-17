# MedSales Sprint 2 — QA Test Strategy
**Document:** QA-04  
**Author:** Mac McDonald, QA Engineer  
**Date:** 2026-03-11  
**Project:** Medical Sales Rep Platform — Spring Boot Backend  
**Sprint:** Sprint 2  
**Status:** Active  
**References:** [QA-03 MedSales API Contract Test Plan](./medsales-api-contract-test-plan.md)

---

## Overview

Sprint 2 delivers four major components: Keycloak JWT authentication (S2-03), the Physician Search API with geo-radius (S2-04), the Physician Profile API with CRM enrichment (S2-05), and NPI data ingestion (S2-02). This document covers the complete test strategy for all four.

I'm testing every one of these like I'm trying to break them. Because I am. That's how you know they actually work.

---

## Global Test Conventions

All test cases in this document follow the error response shape defined in QA-03:
```json
{
  "status": 400,
  "error": "Bad Request",
  "message": "Human-readable description of the error",
  "path": "/api/v1/...",
  "timestamp": "2026-03-11T..."
}
```

All authenticated endpoints require:
```
Authorization: Bearer <jwt_token>
```

All JWT tokens in test scenarios are sourced from the Keycloak test realm unless otherwise noted.

---

## S2-03 — Keycloak JWT Integration Test Plan

### Context

Sprint 2 wires the Spring Boot backend to Keycloak for JWT-based authentication. Every protected endpoint must validate the incoming JWT, extract `org_id` from claims, and enforce org-level data isolation. This is not optional. This is the security backbone of the entire platform.

### Keycloak Test Realm Setup

- **Realm:** `medsales-test`
- **Client:** `medsales-api`
- **Test orgs:** `org-alpha` (ID: `org-uuid-alpha`), `org-beta` (ID: `org-uuid-beta`)
- **Test users:** `rep-alpha@test.com` (org-alpha), `rep-beta@test.com` (org-beta)

### JWT Auth Test Matrix

All test cases below should be run against at minimum: `GET /api/v1/physicians/search?q=Smith` as the representative protected endpoint. Critical cases (marked ⚠️) must be run against ALL protected endpoints.

| TC# | Scenario | Token State | Expected Status | Expected Error `message` | Severity | Notes |
|-----|----------|-------------|-----------------|--------------------------|----------|-------|
| JWT-01 | Valid token, all claims present | Valid, signed by Keycloak, `org_id` present | `200 OK` | — | — | Happy path. Must work. |
| JWT-02 ⚠️ | Expired token | `exp` in the past (e.g. 60 seconds ago) | `401 Unauthorized` | `"Unauthorized"` | **Critical** | Expired tokens must NEVER grant access. |
| JWT-03 ⚠️ | Missing token | No `Authorization` header | `401 Unauthorized` | `"Unauthorized"` | **Critical** | Blank requests must be rejected. |
| JWT-04 ⚠️ | Malformed token — not a JWT | `Authorization: Bearer not-a-real-jwt` | `401 Unauthorized` | `"Unauthorized"` | **Critical** | Garbage in, 401 out. |
| JWT-05 ⚠️ | Valid JWT, tampered signature | Valid header+payload, signature replaced | `401 Unauthorized` | `"Unauthorized"` | **Critical** | Signature validation must catch this. |
| JWT-06 ⚠️ | Token without `org_id` claim | Valid JWT but `org_id` claim missing from payload | `403 Forbidden` | `"Access denied: org_id claim missing from token"` | **Critical** | Must NEVER default to permissive org. |
| JWT-07 ⚠️ | Token with wrong `org_id` (attempts cross-org access) | Valid JWT, `org_id` = `org-uuid-beta`, requests data belonging to `org-alpha` | `403 Forbidden` | `"Access denied"` | **Critical** | Cross-org must fail. See cross-org isolation tests. |
| JWT-08 | Token issued for wrong audience | Valid JWT, `aud` claim = `other-service` | `401 Unauthorized` | `"Unauthorized"` | High | Audience mismatch must reject. |
| JWT-09 | Token issued before `nbf` (not before) | Valid JWT, `nbf` = 5 minutes in the future | `401 Unauthorized` | `"Unauthorized"` | High | Token not yet valid must be rejected. |
| JWT-10 | Valid token, `Bearer` prefix missing | `Authorization: <token>` (no "Bearer " prefix) | `401 Unauthorized` | `"Unauthorized"` | Medium | Scheme enforcement. |
| JWT-11 | Valid token, extra whitespace in header | `Authorization: Bearer  <token>` (double space) | `401` or `200` | Document expected behavior | Low | Robustness — pick a behavior and be consistent. |

### JWT Error Response Shape Validation

For all `401` responses, verify:
```json
{
  "status": 401,
  "error": "Unauthorized",
  "message": "Unauthorized",
  "path": "/api/v1/physicians/search",
  "timestamp": "..."
}
```

For all `403` responses, verify:
```json
{
  "status": 403,
  "error": "Forbidden",
  "message": "Access denied: org_id claim missing from token",
  "path": "/api/v1/physicians/search",
  "timestamp": "..."
}
```

**Key constraint:** No stack traces. No `java.lang.` anything. No internal service names. Safe messages only.

### Regression Scope

After JWT integration is wired, re-run the Security Test Cases from QA-03 (Auth Tests section) against all 8 endpoints. JWT-01 through JWT-05 map directly to those cases. Cross-org tests (JWT-06, JWT-07) map to the Org Isolation section.

---

## S2-04 — Physician Search API Test Cases

### Context

The Search API (`GET /api/v1/physicians/search`) supports keyword search, geo-radius filtering, specialty filtering, and pagination. CRM enrichment data (`lastVisitDate`) must be org-scoped — a rep from Org A cannot see Org B's visit history for the same physician.

> Full contract definition: QA-03, Section 1 — Physician Search.

### Geo-Radius Boundary Conditions

These are the tests everyone skips. I don't skip them.

Test setup: Use a physician at a known fixed coordinate (e.g., Austin, TX: `lat=30.2672, lng=-97.7431`). Query from a reference point at an exactly-known distance.

| TC# | Scenario | Params | Expected Status | Expected Behavior | Notes |
|-----|----------|--------|-----------------|-------------------|-------|
| SRCH-GEO-01 | Physician is exactly at the radius boundary | `lat/lng` positioned so physician is exactly `radius_miles` away (within floating-point tolerance ±0.01 mi) | `200` | Physician **included** in results | Inclusive boundary. Must be IN. |
| SRCH-GEO-02 | Physician is just outside the radius | Same physician, radius shortened by 0.1 miles | `200` | Physician **excluded** from results | Exclusive boundary. Must be OUT. |
| SRCH-GEO-03 | Physician is 0.0 miles away (same point) | `lat/lng` exactly matches physician coordinates, `radius_miles=0.1` | `200` | Physician included | Zero-distance edge case. |
| SRCH-GEO-04 | Very large radius (500 miles) | `radius_miles=500` | `200` | Returns all physicians in range, no error | Max valid value. |
| SRCH-GEO-05 | Radius at max limit exactly | `radius_miles=500.0` | `200` | Valid | Boundary — should pass. |
| SRCH-GEO-06 | Radius just over max | `radius_miles=500.1` | `400` | `"radius_miles must be between 0.1 and 500.0"` | Just over the line — must fail. |
| SRCH-GEO-07 | Radius at zero | `radius_miles=0` | `400` | `"radius_miles must be between 0.1 and 500.0"` | Zero is not a valid radius. |
| SRCH-GEO-08 | Negative radius | `radius_miles=-10` | `400` | `"radius_miles must be between 0.1 and 500.0"` | Negative is nonsense. |
| SRCH-GEO-09 | `lat` at north pole boundary | `lat=90.0` | `200` | Valid (if any physicians there) | Boundary latitude — must not error. |
| SRCH-GEO-10 | `lat` just over boundary | `lat=90.001` | `400` | `"lat must be between -90.0 and 90.0"` | Out of range. |
| SRCH-GEO-11 | `lng` at boundary | `lng=180.0` | `200` | Valid | Boundary longitude. |
| SRCH-GEO-12 | `lng` just over boundary | `lng=180.001` | `400` | `"lng must be between -180.0 and 180.0"` | Out of range. |

### Missing / Partial Geo Parameters

Per contract: `lat`, `lng`, `radius_miles` must ALL be present or ALL absent.

| TC# | Scenario | Params | Expected Status | Expected `message` |
|-----|----------|--------|-----------------|--------------------|
| SRCH-GEO-13 | `lat` only (missing `lng` and `radius_miles`) | `lat=30.2672` | `400` | `"Geo search requires lat, lng, and radius_miles"` |
| SRCH-GEO-14 | `lng` only | `lng=-97.7431` | `400` | `"Geo search requires lat, lng, and radius_miles"` |
| SRCH-GEO-15 | `radius_miles` only | `radius_miles=25` | `400` | `"Geo search requires lat, lng, and radius_miles"` |
| SRCH-GEO-16 | `lat` + `lng`, missing `radius_miles` | `lat=30.2672, lng=-97.7431` | `400` | `"Geo search requires lat, lng, and radius_miles"` |
| SRCH-GEO-17 | `lat` + `radius_miles`, missing `lng` | `lat=30.2672, radius_miles=25` | `400` | `"Geo search requires lat, lng, and radius_miles"` |
| SRCH-GEO-18 | `lng` + `radius_miles`, missing `lat` | `lng=-97.7431, radius_miles=25` | `400` | `"Geo search requires lat, lng, and radius_miles"` |

### Empty Results

| TC# | Scenario | Params | Expected Status | Expected Behavior |
|-----|----------|--------|-----------------|--------------------|
| SRCH-EMPTY-01 | Keyword search with no matches | `q=ZZZNOMATCH9999` | `200` | `content: []`, `totalElements: 0` |
| SRCH-EMPTY-02 | Geo search in remote area with no physicians | `lat=48.0, lng=-110.0, radius_miles=0.1` | `200` | `content: []`, `totalElements: 0` |
| SRCH-EMPTY-03 | Specialty filter with no matching physicians | `specialty=AstroPhysics` | `200` | `content: []`, `totalElements: 0` |

**Critical:** Empty results must NEVER return `404`. A `404` here is a bug. I will file it.

### Specialty Filter

| TC# | Scenario | Params | Expected Status | Expected Behavior |
|-----|----------|--------|-----------------|--------------------|
| SRCH-SPEC-01 | Exact specialty match | `specialty=Cardiology` | `200` | Only physicians with specialty = `Cardiology` returned |
| SRCH-SPEC-02 | Partial match (if spec supports it) | `specialty=Cardio` | `200` or `400` | Document expected behavior — if partial, verify only matching specialties returned |
| SRCH-SPEC-03 | Case-insensitive specialty | `specialty=cardiology` | `200` | Clarify: same as `Cardiology` or empty? Document and verify. |
| SRCH-SPEC-04 | Specialty with trailing whitespace | `specialty=Cardiology ` | `200` (trimmed) or `400` | Trimming behavior — should normalize, not fail |
| SRCH-SPEC-05 | Multiple specialties (if supported) | `specialty=Cardiology&specialty=Oncology` | Document | Verify multi-value behavior |

### Keyword Search — Partial Name Match

| TC# | Scenario | Params | Expected Behavior |
|-----|----------|--------|-------------------|
| SRCH-KW-01 | Full last name | `q=Smith` | Returns all physicians with last name Smith |
| SRCH-KW-02 | Partial last name | `q=Smi` | Returns physicians where last name starts with "Smi" (prefix match) |
| SRCH-KW-03 | First name partial | `q=Jan` | Returns physicians with first name matching "Jan..." |
| SRCH-KW-04 | NPI exact search | `q=1234567890` | Returns exact NPI match |
| SRCH-KW-05 | Clinic name partial | `q=Austin Heart` | Returns physicians associated with that clinic |
| SRCH-KW-06 | `q` at max length (200 chars) | 200-char string | `200` — valid request |
| SRCH-KW-07 | `q` exceeds max (201 chars) | 201-char string | `400` — validation error |
| SRCH-KW-08 | SQL injection attempt | `q='; DROP TABLE physicians;--` | `400` or sanitized — never executed as SQL |
| SRCH-KW-09 | XSS payload | `q=<script>alert(1)</script>` | Returned escaped in response — never raw HTML |

### Pagination

| TC# | Scenario | Params | Expected Behavior |
|-----|----------|--------|-------------------|
| SRCH-PAGE-01 | Page 0 (default) | No `page` param | Returns first page, `first: true`, `page: 0` |
| SRCH-PAGE-02 | Page 0 explicit | `page=0` | Same as above |
| SRCH-PAGE-03 | Page 1 | `page=1` | Returns second page of results |
| SRCH-PAGE-04 | Last page | `page=<totalPages-1>` | `last: true`, partial `content` may have fewer than `size` items |
| SRCH-PAGE-05 | Page beyond last | `page=99999` | `200`, `content: []`, `last: true` (not 404, not error) |
| SRCH-PAGE-06 | Negative page | `page=-1` | `400` — `"page must be ≥ 0"` |
| SRCH-PAGE-07 | Default page size | No `size` param | Returns max 20 results |
| SRCH-PAGE-08 | Custom page size | `size=10` | Returns max 10 results |
| SRCH-PAGE-09 | Max page size | `size=100` | Returns max 100 results — valid |
| SRCH-PAGE-10 | Page size over max | `size=101` | `400` — `"size must not exceed 100"` |
| SRCH-PAGE-11 | Page size of 0 | `size=0` | `400` — size must be ≥ 1 |
| SRCH-PAGE-12 | Pagination metadata accuracy | `size=10`, result set of 47 items | `totalElements: 47`, `totalPages: 5`, correct `first`/`last` flags |

### Cross-Org Data Isolation — `lastVisitDate`

This is the most critical set of tests in this entire document. Cross-org data leakage is a security breach, not a bug. I treat these with Critical severity.

**Setup:**
- Physician NPI `1234567890` exists in the national registry
- Org Alpha rep (`rep-alpha`) has made 3 calls to this physician; last visit = `2026-02-01`
- Org Beta rep (`rep-beta`) has made 5 calls; last visit = `2026-02-28`

| TC# | Scenario | Auth | Expected Behavior |
|-----|----------|------|-------------------|
| SRCH-ISO-01 | Org Alpha rep searches, sees own `lastVisitDate` | JWT: `rep-alpha` | `lastVisitDate: "2026-02-01"` in result for NPI 1234567890 |
| SRCH-ISO-02 | Org Beta rep searches, sees own `lastVisitDate` | JWT: `rep-beta` | `lastVisitDate: "2026-02-28"` — Org Beta's data only |
| SRCH-ISO-03 | Org Alpha rep cannot see Org Beta's `lastVisitDate` | JWT: `rep-alpha` | Must NOT return `2026-02-28`. Must return Org Alpha's value. |
| SRCH-ISO-04 | Org with no CRM history for a physician | JWT: new org rep | `lastVisitDate: null` — not Org Alpha's or Beta's value |
| SRCH-ISO-05 | `org_id` from token, not from request param | JWT: `rep-alpha`, no `org_id` request param | Server uses JWT `org_id` exclusively — request param cannot override |

---

## S2-05 — Physician Profile API Test Cases

### Context

`GET /api/v1/physicians/{npi}` returns the full physician profile plus an org-scoped `crmSummary`. The physician record itself is from the national NPI registry (global). The `crmSummary` block is org-scoped.

> Full contract definition: QA-03, Section 2 — Physician Profile.

### NPI Not Found

| TC# | Scenario | NPI | Expected Status | Expected `message` |
|-----|----------|-----|-----------------|---------------------|
| PROF-01 | NPI that does not exist in registry | `GET /api/v1/physicians/9999999999` | `404` | `"Physician not found: 9999999999"` |
| PROF-02 | NPI is all zeros | `GET /api/v1/physicians/0000000000` | `404` | `"Physician not found: 0000000000"` |
| PROF-03 | Invalid NPI — only 9 digits | `GET /api/v1/physicians/123456789` | `400` | `"NPI must be exactly 10 numeric digits"` |
| PROF-04 | Invalid NPI — 11 digits | `GET /api/v1/physicians/12345678901` | `400` | `"NPI must be exactly 10 numeric digits"` |
| PROF-05 | Invalid NPI — contains letters | `GET /api/v1/physicians/123456789A` | `400` | `"NPI must be exactly 10 numeric digits"` |
| PROF-06 | Invalid NPI — contains special chars | `GET /api/v1/physicians/1234-56789` | `400` or `404` | Must not cause a 500 — document expected behavior |

### NPI Exists, No CRM History for Org

| TC# | Scenario | Auth | Expected Status | Expected Response |
|-----|----------|------|-----------------|-------------------|
| PROF-07 | NPI exists in registry, org has zero CRM activity | JWT: new org rep | `200` | Full physician profile returned, `crmSummary` fields all `null` |
| PROF-08 | Verify `crmSummary` structure when empty | JWT: new org rep | `200` | `crmSummary: { "lastCallDate": null, "totalCalls": 0, "openTasks": 0, "lastSampleDrop": null }` |

**Key constraint:** `200` with null CRM summary — NOT `404`, NOT `403`. The physician exists. The org just has no history. That's a valid state.

### NPI Exists with CRM History

| TC# | Scenario | Auth | Expected Status | Expected Response |
|-----|----------|------|-----------------|-------------------|
| PROF-09 | NPI exists, org has 12 calls, 2 open tasks, last sample drop | JWT: `rep-alpha` | `200` | `crmSummary: { "lastCallDate": "2026-02-14", "totalCalls": 12, "openTasks": 2, "lastSampleDrop": "2026-01-20" }` |
| PROF-10 | `totalCalls` increments after new call logged | Log a call, then re-fetch profile | `200` | `totalCalls` increased by 1 |
| PROF-11 | `openTasks` decrements after task completed | Complete a task, then re-fetch profile | `200` | `openTasks` decreased by 1 |
| PROF-12 | `lastCallDate` updates after more recent call | Log call with today's date, re-fetch | `200` | `lastCallDate` = today's date |

### Cross-Org Isolation — `crmSummary`

Same physician NPI, two orgs, two different histories.

**Setup:**
- NPI `1234567890` — Dr. Jane Smith
- Org Alpha: 12 total calls, last call `2026-02-14`, 2 open tasks
- Org Beta: 3 total calls, last call `2026-01-01`, 0 open tasks

| TC# | Scenario | Auth | Expected `crmSummary` |
|-----|----------|------|-----------------------|
| PROF-ISO-01 | Org Alpha rep fetches profile | JWT: `rep-alpha` | `totalCalls: 12`, `lastCallDate: "2026-02-14"`, `openTasks: 2` |
| PROF-ISO-02 | Org Beta rep fetches same NPI | JWT: `rep-beta` | `totalCalls: 3`, `lastCallDate: "2026-01-01"`, `openTasks: 0` |
| PROF-ISO-03 | Org Alpha cannot see Org Beta's `totalCalls` | JWT: `rep-alpha` | `totalCalls` must be `12`, NOT `3`, NOT `15` (combined) |
| PROF-ISO-04 | No CRM history org cannot see other orgs' data | JWT: new org rep | All `crmSummary` fields null — not Alpha's, not Beta's |
| PROF-ISO-05 | CRM summary is scoped to JWT `org_id` only | JWT: `rep-alpha` with spoofed `org_id` attempt | Server uses JWT claim exclusively — cannot override via request |

**Critical:** If Org Beta's `lastCallDate` appears in Org Alpha's response, that is a **Critical security bug**. File immediately, halt release.

---

## S2-02 — NPI Data Ingestion Validation Checklist

### Context

Sprint 2 includes bulk ingestion of NPI registry data into PostGIS. Expected record count: 20,000–30,000. This is the data foundation the Search and Profile APIs depend on. Bad ingestion = bad API = everything is broken. I test this first, before S2-04 and S2-05 test runs.

### Row Count Validation

| Check | Method | Pass Criteria | Fail Action |
|-------|--------|---------------|-------------|
| Total row count after ingestion | `SELECT COUNT(*) FROM physicians` | Count is between 20,000 and 30,000 | Flag as Critical — do not proceed with API tests |
| Pre vs post comparison | Run ingestion twice; compare counts | Counts are equal (idempotency) | See Idempotency tests below |
| Row count matches source file | `wc -l <source_file>` minus header row | DB count = file rows (minus header, minus known invalid rows) | Investigate discrepancy |

### NPI Format Checks

| Check | SQL / Method | Pass Criteria |
|-------|-------------|---------------|
| NPI is exactly 10 characters | `SELECT COUNT(*) FROM physicians WHERE LENGTH(npi) != 10` | Count = 0 |
| NPI is numeric only | `SELECT COUNT(*) FROM physicians WHERE npi !~ '^[0-9]{10}$'` | Count = 0 |
| No leading/trailing whitespace in NPI | `SELECT COUNT(*) FROM physicians WHERE npi != TRIM(npi)` | Count = 0 |

Any non-conforming NPI rows must have been **rejected at ingestion** with a logged error, not silently stored.

### Duplicate NPI Detection

| Check | Method | Pass Criteria |
|-------|--------|---------------|
| No duplicate NPIs | `SELECT npi, COUNT(*) FROM physicians GROUP BY npi HAVING COUNT(*) > 1` | Returns 0 rows |
| Duplicate in source file handled | Introduce a known duplicate in test file, run ingestion | Only one record stored; duplicate logged as a warning/error |
| Duplicate ingestion does not corrupt existing record | Ingest same record twice | Record unchanged after second ingestion |

### Null Geometry Records

Per spec: null `geom` records must be **logged, not skipped** (they are still ingested for NPI lookup purposes, just not geo-searchable).

| Check | Method | Pass Criteria |
|-------|--------|---------------|
| Null geom records are stored | `SELECT COUNT(*) FROM physicians WHERE geom IS NULL` | Count > 0 if source data has nulls |
| Null geom records are logged | Check ingestion log output | Each null geom record appears in log with NPI and reason |
| Null geom records do NOT appear in geo search results | Run geo search with tight radius | No null-geom physicians returned by `lat/lng/radius_miles` search |
| Non-null geom records are searchable | Run geo search covering known non-null physicians | Those physicians appear in results |

### Idempotency Test

Run the ingestion pipeline **twice** on the same source data.

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Run ingestion (first pass) | N records inserted. Log shows success. |
| 2 | Check row count | Count = N |
| 3 | Run ingestion again (second pass) on same data | No new records created. Existing records updated or skipped (not duplicated). |
| 4 | Check row count again | Count still = N (no increase) |
| 5 | Verify data integrity | Spot-check 10 NPIs — values unchanged after second ingestion |
| 6 | Check logs | Second run logs indicate "already exists" or "no change" for all records |

**Pass criteria:** Row count after second ingestion equals row count after first ingestion. Zero duplicates created.

### Additional Ingestion Sanity Checks

| Check | Pass Criteria |
|-------|---------------|
| Ingestion completes without fatal errors | Exit code 0; no uncaught exceptions in log |
| All fields mapped correctly (spot check 10 records) | NPI, name, specialty, address, coordinates match source data |
| PostGIS geometry stored in correct SRID | `SELECT ST_SRID(geom) FROM physicians LIMIT 1` returns `4326` (WGS84) |
| Geo search returns expected results post-ingestion | `lat/lng/radius_miles` search near Austin returns Austin physicians |

---

## Test Execution Order

1. **S2-02 Ingestion** — Run and validate data first. Nothing else works without it.
2. **S2-03 JWT Auth** — Validate auth layer before hitting any business endpoints.
3. **S2-04 Search API** — Depends on ingestion data and auth.
4. **S2-05 Profile API** — Depends on ingestion data and auth.

---

## Test Environment Requirements

| Requirement | Detail |
|-------------|--------|
| Keycloak test realm | `medsales-test` realm configured with test users for org-alpha and org-beta |
| PostGIS database | Loaded with Sprint 2 ingestion pipeline output |
| Spring Boot app | Running with Keycloak integration enabled |
| Test data | Minimum 2 physicians in different geo locations; at least 1 physician with CRM history per org |
| Isolation | Org Alpha and Org Beta test users must be distinct, non-overlapping |

---

## Bug Severity Reference

| Severity | Definition | Examples in This Sprint |
|----------|------------|------------------------|
| **Critical** | Security breach, data loss, app down | Cross-org data leakage, expired token accepted, null org_id allowed |
| **High** | Major feature broken, no workaround | Geo search returns wrong results, profile 404s for valid NPI |
| **Medium** | Feature partially broken, workaround exists | Pagination metadata wrong, `distanceMiles` missing from geo results |
| **Low** | Cosmetic, minor | Inconsistent whitespace trimming, non-standard error message wording |

All Critical bugs go to K2S0 **immediately**. I don't wait. I don't sleep on it. Immediately.

---

## Sign-Off Criteria

Sprint 2 QA sign-off requires:
- [ ] All JWT matrix test cases (JWT-01 through JWT-11) executed and documented
- [ ] All geo-radius boundary tests (SRCH-GEO-01 through SRCH-GEO-18) pass
- [ ] Cross-org isolation verified for Search (SRCH-ISO-01 through SRCH-ISO-05)
- [ ] Cross-org isolation verified for Profile (PROF-ISO-01 through PROF-ISO-05)
- [ ] All ingestion checklist items verified
- [ ] Idempotency test passed
- [ ] Zero open Critical bugs
- [ ] Zero open High bugs (or waiver approved by K2S0 + PM)

If any Critical is open, we do not ship. Non-negotiable.

---

*— Mac McDonald, QA Engineer. Sprint 2 doesn't ship until it passes. Period.*
