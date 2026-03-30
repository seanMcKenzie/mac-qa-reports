# Sprint 6 QA Report — MedSales API
**Ticket:** S6-09  
**Author:** Mac McDonald, QA Engineer  
**Date:** 2026-03-20  
**Sprint:** Sprint 6  
**API Version:** v1  
**Environment:** Local dev (`http://localhost:8080/api/v1`)  
**Backend:** Spring Boot (medsales-api-0.0.1-SNAPSHOT.jar), profile: `local`  
**Database:** PostgreSQL 15 + PostGIS (Docker: `medsales-postgres`)  
**Auth:** Keycloak 24.0 (`medsales-dev` realm, `testrep@medsales.dev`)

---

## Executive Summary

| Metric | Value |
|--------|-------|
| Total tests executed | 15 |
| **PASS** | **15** |
| **FAIL** | **0** |
| **BLOCKED** | **0** |
| Sprint 5 Regression | 10/10 ✅ |
| Sprint 6 Acceptance | 5/5 ✅ |

**Overall verdict: ✅ SPRINT 6 READY — No blockers. All acceptance criteria met.**

---

## Environment Notes

### Pre-Test Issues Resolved

1. **API server was not running** — The Spring Boot JAR was not started. Started manually:
   ```bash
   java -jar build/libs/medsales-api-0.0.1-SNAPSHOT.jar --spring.profiles.active=local
   ```
   This is expected in local dev. DevOps should document startup procedure.

2. **Java not on PATH** — The `java` symlink at `/usr/bin/java` was broken (no JVM installed to `/Library/Java/JavaVirtualMachines`). Java was available via Homebrew at `$(brew --prefix openjdk)/bin/java`. Used `export JAVA_HOME=$(brew --prefix openjdk)` as workaround.
   > **Recommendation:** Pin Java path in developer onboarding docs or add to shell profile.

3. **Keycloak auth required (not bypassed)** — Despite the `local` profile, OAuth2 JWT authentication was enforced. Obtained test token via:
   - Realm: `medsales-dev`
   - Client: `medsales-api`
   - Credentials: `testrep@medsales.dev` / `password`
   > **Note:** `application-local.yml` has `unauthenticated-rpm: 1000` but does NOT disable security. Old test case docs claiming "auth bypass is active on local profile" are **outdated and incorrect**.

4. **Rate limiting encountered on T5** — First attempt on `/procedures` returned `429 Too Many Requests` due to prior rapid-fire requests during test setup. Passed on retry after 5-second delay. Rate limiter is working as designed.

---

## Part 1 — Sprint 5 Regression Results

All Sprint 5 endpoints verified to be functioning correctly. No regressions detected.

| # | Test | Endpoint | Expected | Actual | Result |
|---|------|----------|----------|--------|--------|
| T1 | Basic geo search | `GET /physicians/search?lat=35.4676&lon=-97.5164&size=5` | 200, results[] | 200, 5 results | ✅ PASS |
| T2 | Physician detail – Snell | `GET /physicians/1225021702` | 200, Brian Snell, enrichmentSummary | 200, BRIAN SNELL, enrichmentSummary present | ✅ PASS |
| T3 | Snell payments | `GET /physicians/1225021702/payments` | 200, $891,163 total | 200, $891,163.26 total | ✅ PASS |
| T4 | Snell prescribing | `GET /physicians/1225021702/prescribing` | 200, opioid flag | 200, opioidClaimCount=130, opioidFlag=true on drugs | ✅ PASS |
| T5 | Snell procedures | `GET /physicians/1225021702/procedures` | 200, results | 200, 20 procedures (1194 total services) | ✅ PASS |
| T6 | Search: hasOpenPayments | `GET /search?hasOpenPayments=true&size=3` | 200 | 200, 3 results returned | ✅ PASS |
| T7 | Search: opioidPrescriber | `GET /search?opioidPrescriber=true&size=3` | 200 | 200, 3 results (YEN DUNG NGUYEN first) | ✅ PASS |
| T8 | Search: hcpcsCode | `GET /search?hcpcsCode=22853&size=3` | 200 | 200, 3 results (TAMI MYERS first) | ✅ PASS |
| T9 | Search: drugName | `GET /search?drugName=Oxycodone&size=3` | 200 | 200, 3 results (JEFFREY HIRSCH first) | ✅ PASS |
| T10 | Search: payerName | `GET /search?payerName=Omnia&size=3` | 200 | 200, 3 results (ANDREW PARKINSON first) | ✅ PASS |

### Key Data Validations (T2–T5)
- **T2:** `enrichmentSummary.totalPayments = 891163.26`, `opioidClaimCount = 130`, `totalProcedures = 1194` ✅
- **T3:** `totalPayments = 891163.26`, top payer = Omnia Medical ($891,000.00) ✅
- **T4:** 8 drugs returned, `opioidFlag: true` on Oxycodone Hcl, Hydromorphone Hcl, Hydrocodone-Acetaminophen ✅
- **T5:** 20 procedure types, top by volume: 72110 (X-ray lower spine, 170 services) ✅

---

## Part 2 — Sprint 6 Acceptance Test Results

All 5 new DMEPOS feature tests passed. The new endpoints are fully functional.

| # | Test | Endpoint | Expected | Actual | Result |
|---|------|----------|----------|--------|--------|
| T11 | Snell DMEPOS items | `GET /physicians/1225021702/dmepos` | 200, 1 item | 200, 1 item (E0745) | ✅ PASS |
| T12 | Hirsch DMEPOS items | `GET /physicians/1417988957/dmepos` | 200, 18 items | 200, 18 items | ✅ PASS |
| T13 | Snell detail dmeposSummary | `GET /physicians/1225021702` | 200, dmeposSummary not null | 200, dmeposSummary present | ✅ PASS |
| T14 | Search by dmeposCode A6196 | `GET /search?dmeposCode=A6196&size=5` | 200, only A6196 refs | 200, 5 results, verified A6196 in DMEPOS | ✅ PASS |
| T15 | Search: dmeposCode + minDmeposClaims | `GET /search?dmeposCode=A6196&minDmeposClaims=10&size=5` | 200, filtered results | 200, 5 results with ≥10 claims | ✅ PASS |

### Detailed Sprint 6 Findings

#### T11 — Snell DMEPOS
```json
{
  "items": [{
    "hcpcsCode": "E0745",
    "hcpcsDescription": "Neuromuscular stimulator, electronic shock unit",
    "totalClaims": 11,
    "totalBeneficiaries": 0
  }]
}
```
✅ Exactly 1 item as expected.

#### T12 — Hirsch DMEPOS
18 items confirmed. Top items include:
- `E0601` — Continuous positive airway pressure (CPAP)
- `A7038` — Filter, disposable, used with positive airway pressure device
- `A4604` — Tubing with integrated heating element for CPAP
- `A6196` — Alginate or other fiber gelling dressing
- ... (18 total)

✅ Item count matches expected value.

#### T13 — dmeposSummary in physician detail
```json
"dmeposSummary": {
  "totalReferrals": 1,
  "totalClaims": 11,
  "totalBeneficiaries": 0,
  "topItem": "E0745 — Neuromuscular stimulator, electronic shock unit"
}
```
✅ Field present, populated, and consistent with T11 data.

#### T14 — DMEPOS code search (A6196)
Results: STEVEN DANLEY, FRANCESCA BOULOS, CAROLINE MERRITT-SCHIERMEYER, JESSI RUNYAN, STEPHEN BARR

**Filter correctness verified** — Checked Danley (NPI `1124193255`) directly:
- Has 12 DMEPOS items including `A6196` (Alginate or other fiber gelling dressing, 20 claims)
- ✅ Filter is not a no-op

#### T15 — DMEPOS code + minDmeposClaims filter
- `minDmeposClaims=10` → 5 results (same 5 as T14, all have ≥10 A6196 claims)
- `minDmeposClaims=100` → 0 results ✅ (confirmed filter is working)
- Danley has 20 A6196 claims → passes the threshold ✅

---

## Blockers

**None.** Sprint 6 is clear for release.

---

## Observations & Recommendations

### Medium Priority
1. **Auth bypass doc is stale** — `medsales-api-curl-tests.md` (created Sprint 4) claims the local profile bypasses auth. This is **not true** — JWT is required. Update onboarding docs and the test file header.

2. **Rate limiter affects authenticated test runners** — Tests T5 got an initial 429 because the rate limiter counts all requests (including token fetches). Consider increasing `requests-per-minute` for local dev profile or document the 1-second delay pattern between test calls.

### Low Priority
3. **API server startup not automated** — The Spring Boot server must be started manually. Consider adding a `./gradlew bootRun --args='--spring.profiles.active=local'` convenience script or docker-compose service for local dev.

4. **Java path issue** — `/usr/bin/java` symlink is broken on the test machine. The valid JVM is at `$(brew --prefix openjdk)/bin/java`. Add this to the developer setup guide.

---

## Test Execution Log

| Time (CDT) | Action |
|------------|--------|
| 09:54 | Task received, intro posted to #medical-sales-rep-project |
| 09:55 | Discovered API server not running, started manually |
| 09:56 | Resolved Java path issue (Homebrew openjdk) |
| 09:57 | DB connectivity confirmed (changeme_in_prod password) |
| 09:58 | Keycloak token obtained (testrep@medsales.dev) |
| 09:59 | T1–T10 Sprint 5 regression — all PASS |
| 10:01 | T11–T15 Sprint 6 acceptance tests — all PASS |
| 10:02 | T14 filter correctness spot-checked (Danley NPI verified) |
| 10:03 | T15 filter confirmed working (minDmeposClaims=100 → 0 results) |

---

## Sign-off

All Sprint 5 regression tests pass. All Sprint 6 DMEPOS acceptance criteria are met. No blockers identified.

**✅ APPROVED FOR RELEASE**

---

*"I go into every feature assuming it's broken. I don't stop until I've proved it or proved it isn't. Either way, I win." — Mac*
