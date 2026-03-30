# MedSales API — Comprehensive curl Test Cases
**Author:** Mac McDonald, QA Engineer  
**Created:** 2026-03-12  
**Updated:** 2026-03-20 — Sprint 6 DMEPOS test cases added (TC-39–TC-48)
**API Version:** v1  

---

## ⚙️ Local Dev Setup Notes

> **Auth bypass is active on the `local` Spring profile.** No `Authorization: Bearer ...` header is required for curl tests against `http://localhost:8080`. In staging/production, every protected endpoint requires a valid JWT.  
>
> The `@CurrentOrg OrgSecurityContext` in CRM and Favorites endpoints is satisfied by a test/dev stub that injects a fixed `orgId` and `userId` when running locally — no auth needed.
>
> **NPPES Admin** (`POST /api/v1/admin/nppes/refresh`) uses `@PreAuthorize("hasRole('ADMIN')")` which may still require a role header or dev stub depending on Spring Security config — see TC-35/TC-36.
>
> **Reference NPI:** `1600024222` → Kevin Robinson, Oklahoma City, OK (seeded in local DB).

```bash
# Set once before running tests
BASE_URL=http://localhost:8080
```

---

## Test Case Index (38 total)

| # | Category | Test Name | Expected Status |
|---|----------|-----------|----------------|
| TC-01 | Health | Actuator health check | 200 |
| TC-02 | Health | Actuator info | 200 |
| TC-03 | Health | Actuator metrics | 200 |
| TC-04 | Physician Search | Happy path – default state | 200 |
| TC-05 | Physician Search | Full-text query by name | 200 |
| TC-06 | Physician Search | Filter by specialty | 200 |
| TC-07 | Physician Search | Filter by city | 200 |
| TC-08 | Physician Search | Filter by city + specialty | 200 |
| TC-09 | Physician Search | Geographic radius search | 200 |
| TC-10 | Physician Search | Pagination – page 1 | 200 |
| TC-11 | Physician Search | Pagination – custom page size | 200 |
| TC-12 | Physician Search | All filters combined | 200 |
| TC-13 | Physician Search | Query yields zero results | 200 (empty) |
| TC-14 | Physician Search | State-only (no query) | 200 |
| TC-15 | Physician Search | Edge – page size = 1 | 200 |
| TC-16 | Physician Search | Edge – very large page number | 200 (empty) |
| TC-17 | Physician Profile | Valid NPI (Kevin Robinson) | 200 |
| TC-18 | Physician Profile | Invalid / unknown NPI | 404 |
| TC-19 | Physician Profile | Malformed NPI (too short) | 404 or 400 |
| TC-20 | Physician Profile | Alpha-string as NPI | 404 or 400 |
| TC-21 | CRM Interactions | Create – visit interaction | 201 |
| TC-22 | CRM Interactions | Create – call interaction | 201 |
| TC-23 | CRM Interactions | Create – email interaction | 201 |
| TC-24 | CRM Interactions | Create – demo interaction | 201 |
| TC-25 | CRM Interactions | List interactions for physician | 200 |
| TC-26 | CRM Interactions | Create – missing required fields | 400 |
| TC-27 | CRM Interactions | Create – invalid interaction type | 400 |
| TC-28 | Favorites | List favorites (likely empty) | 200 |
| TC-29 | Favorites | Add physician to favorites | 201 |
| TC-30 | Favorites | Check favorite – is favorited | 200 |
| TC-31 | Favorites | Add same physician again (idempotent?) | 201 or 409 |
| TC-32 | Favorites | Check favorite – not favorited | 200 (false) |
| TC-33 | Favorites | Remove favorite | 204 |
| TC-34 | Favorites | Remove non-existent favorite | 204 or 404 |
| TC-35 | Rate Limiting | Rapid-fire requests trigger 429 | 429 |
| TC-36 | Admin / NPPES | Trigger NPPES refresh (local/ADMIN) | 202 |
| TC-37 | Admin / NPPES | Trigger refresh without ADMIN role | 403 |
| TC-38 | Error Cases | Unknown route (404) | 404 |
| TC-39 | DMEPOS | Snell DMEPOS items (1 item) | 200 |
| TC-40 | DMEPOS | Hirsch DMEPOS items (18 items) | 200 |
| TC-41 | DMEPOS | Unknown NPI DMEPOS | 404 |
| TC-42 | DMEPOS | Physician detail includes dmeposSummary | 200 |
| TC-43 | DMEPOS Search | Filter by dmeposCode (A6196) | 200 |
| TC-44 | DMEPOS Search | Filter by dmeposCode + minDmeposClaims | 200 |
| TC-45 | DMEPOS Search | High minDmeposClaims threshold (empty) | 200 (empty) |
| TC-46 | DMEPOS Search | Physician with no DMEPOS data — dmeposSummary null | 200 |
| TC-47 | DMEPOS Search | Invalid dmeposCode format | 200 (empty) or 400 |
| TC-48 | DMEPOS Search | dmeposCode + size pagination | 200 |

---

## 1. Health / Actuator

### TC-01 — Actuator health check
**Description:** Verify the Spring Boot actuator health endpoint is up and reports `UP`.

```bash
curl -s -o /dev/null -w "%{http_code}" \
  "${BASE_URL}/actuator/health"
```
**Expected status:** `200`  
**Expected response:**
```json
{
  "status": "UP",
  "components": {
    "db": { "status": "UP" },
    "diskSpace": { "status": "UP" }
  }
}
```
**Notes:** Run this first. If it's not 200, don't bother with the rest — something is fundamentally broken.

---

### TC-02 — Actuator info
**Description:** Verify the `/actuator/info` endpoint responds.

```bash
curl -s "${BASE_URL}/actuator/info"
```
**Expected status:** `200`  
**Expected response:** JSON object (may be empty `{}` or contain build info depending on config).  
**Notes:** Low priority but confirms actuator exposure is configured correctly.

---

### TC-03 — Actuator metrics
**Description:** Verify metrics endpoint is accessible (used to confirm rate-limit counters are trackable).

```bash
curl -s "${BASE_URL}/actuator/metrics"
```
**Expected status:** `200`  
**Expected response:**
```json
{
  "names": ["jvm.memory.used", "http.server.requests", ...]
}
```
**Notes:** If `management.endpoints.web.exposure.include` is locked down, this may return 404 — flag as a finding.

---

## 2. Physician Search

### TC-04 — Happy path – default state (OK)
**Description:** Minimal search — just default state. Should return physicians.

```bash
curl -s "${BASE_URL}/api/v1/physicians/search"
```
**Expected status:** `200`  
**Expected response:**
```json
{
  "content": [
    {
      "npi": "1600024222",
      "firstName": "Kevin",
      "lastName": "Robinson",
      "specialty": "...",
      "city": "Oklahoma City",
      "state": "OK"
    }
  ],
  "totalElements": 1,
  "totalPages": 1,
  "page": 0,
  "size": 20
}
```
**Notes:** Default state is `OK`. Kevin Robinson (NPI `1600024222`) should appear. Verify `content` is not empty.

---

### TC-05 — Full-text query by name
**Description:** Search by physician name using free-text query param.

```bash
curl -s "${BASE_URL}/api/v1/physicians/search?query=Robinson"
```
**Expected status:** `200`  
**Expected response:** `content` array includes physician with `lastName=Robinson`.  
**Notes:** Tests the free-text search path. Also try first name: `?query=Kevin`.

---

### TC-06 — Filter by specialty
**Description:** Filter physicians by medical specialty.

```bash
curl -s "${BASE_URL}/api/v1/physicians/search?specialty=Family+Medicine&state=OK"
```
**Expected status:** `200`  
**Expected response:**
```json
{
  "content": [
    { "specialty": "Family Medicine", "state": "OK" }
  ]
}
```
**Notes:** All results should have matching specialty. Verify no cross-specialty contamination.

---

### TC-07 — Filter by city
**Description:** Filter physicians by city name.

```bash
curl -s "${BASE_URL}/api/v1/physicians/search?city=Oklahoma+City&state=OK"
```
**Expected status:** `200`  
**Expected response:** All `content[*].city` should equal `"Oklahoma City"`.  
**Notes:** Kevin Robinson (NPI `1600024222`) should appear. Case sensitivity of city param — worth noting.

---

### TC-08 — Filter by city + specialty
**Description:** Combined city + specialty filter.

```bash
curl -s "${BASE_URL}/api/v1/physicians/search?city=Oklahoma+City&specialty=Internal+Medicine&state=OK"
```
**Expected status:** `200`  
**Expected response:** Narrowed result set — only OKC Internal Medicine physicians.  
**Notes:** Intersection behavior. If empty results, confirm filter logic is AND (not OR).

---

### TC-09 — Geographic radius search
**Description:** Search by lat/lon within 25-mile radius (Oklahoma City coordinates).

```bash
curl -s "${BASE_URL}/api/v1/physicians/search?latitude=35.4676&longitude=-97.5164&radiusMiles=25&state=OK"
```
**Expected status:** `200`  
**Expected response:** Physicians within 25 miles of OKC. Kevin Robinson should appear.  
**Notes:** Uses `latitude`, `longitude`, `radiusMiles` params. Test with smaller radius to verify filtering tightens.

---

### TC-10 — Pagination – page 1
**Description:** Retrieve the second page of results (zero-based).

```bash
curl -s "${BASE_URL}/api/v1/physicians/search?state=OK&page=1&size=20"
```
**Expected status:** `200`  
**Expected response:**
```json
{
  "page": 1,
  "size": 20,
  "content": [...]
}
```
**Notes:** Verify `page` field in response matches requested page. If DB has <21 records, `content` may be empty — that's valid.

---

### TC-11 — Pagination – custom page size
**Description:** Reduce page size to 5.

```bash
curl -s "${BASE_URL}/api/v1/physicians/search?state=OK&page=0&size=5"
```
**Expected status:** `200`  
**Expected response:** `content` array has at most 5 elements. `size` field = `5`.  
**Notes:** Confirm the response accurately reports page size even when fewer results exist.

---

### TC-12 — All filters combined
**Description:** Kitchen-sink request with every filter param.

```bash
curl -s "${BASE_URL}/api/v1/physicians/search?query=Kevin&specialty=Internal+Medicine&city=Oklahoma+City&state=OK&latitude=35.4676&longitude=-97.5164&radiusMiles=50&page=0&size=10"
```
**Expected status:** `200`  
**Expected response:** Filtered result set. May return 0 results if Kevin Robinson's specialty differs — that's informative.  
**Notes:** Validates all params are accepted without 400. If query+geo+filters combined cause unexpected behavior, that's a bug.

---

### TC-13 — Query yields zero results
**Description:** Search for a name that definitely doesn't exist.

```bash
curl -s "${BASE_URL}/api/v1/physicians/search?query=xyzzy_nonexistent_physician_12345&state=OK"
```
**Expected status:** `200`  
**Expected response:**
```json
{
  "content": [],
  "totalElements": 0,
  "totalPages": 0
}
```
**Notes:** Should be 200 with empty results, NOT 404. A 404 here would be a bug.

---

### TC-14 — State-only search (no other params)
**Description:** Omit `state` override — should default to `OK`.

```bash
curl -s "${BASE_URL}/api/v1/physicians/search?state=TX"
```
**Expected status:** `200`  
**Expected response:** Physicians in Texas (if any seeded). Validates `state` param works beyond default.  
**Notes:** If TX has no data, `content` will be empty — acceptable.

---

### TC-15 — Edge: page size = 1
**Description:** Minimum meaningful page size.

```bash
curl -s "${BASE_URL}/api/v1/physicians/search?state=OK&page=0&size=1"
```
**Expected status:** `200`  
**Expected response:** Exactly 1 result in `content`. `size=1`, `totalElements >= 1`.  
**Notes:** Off-by-one errors love to hide at boundaries.

---

### TC-16 — Edge: very large page number
**Description:** Request a page far beyond what exists.

```bash
curl -s "${BASE_URL}/api/v1/physicians/search?state=OK&page=9999&size=20"
```
**Expected status:** `200`  
**Expected response:**
```json
{ "content": [], "page": 9999, "totalElements": <N> }
```
**Notes:** Should return empty content gracefully. Should NOT 500. Verify `totalElements` is still populated.

---

## 3. Physician Profile

### TC-17 — Valid NPI: Kevin Robinson
**Description:** Retrieve full profile for known NPI.

```bash
curl -s "${BASE_URL}/api/v1/physicians/1600024222"
```
**Expected status:** `200`  
**Expected response:**
```json
{
  "npi": "1600024222",
  "firstName": "Kevin",
  "lastName": "Robinson",
  "city": "Oklahoma City",
  "state": "OK",
  "specialty": "...",
  "address": "...",
  "phone": "...",
  "interactions": [...]
}
```
**Notes:** This is the golden-path test. Confirm `npi` field matches. Confirm `interactions` array is present (even if empty on fresh DB).

---

### TC-18 — Invalid / unknown NPI (404)
**Description:** Request a valid-format NPI that doesn't exist in the DB.

```bash
curl -s -o /dev/null -w "%{http_code}" \
  "${BASE_URL}/api/v1/physicians/9999999999"
```
**Expected status:** `404`  
**Expected response:**
```json
{
  "status": 404,
  "error": "Not Found",
  "message": "Physician not found"
}
```
**Notes:** The controller explicitly returns `ResponseEntity.notFound().build()`. Verify the response body is a structured error (not a stack trace).

---

### TC-19 — Malformed NPI (too short)
**Description:** NPI with fewer than 10 digits.

```bash
curl -s -o /dev/null -w "%{http_code}" \
  "${BASE_URL}/api/v1/physicians/12345"
```
**Expected status:** `404` (path still matches `/{npi}`, lookup fails) or `400` if validated.  
**Notes:** NPIs are exactly 10 digits. Check whether the service validates format before DB lookup. If a 500 comes back — that's a critical bug.

---

### TC-20 — Alpha-string as NPI
**Description:** Pass a non-numeric string as NPI path variable.

```bash
curl -s -o /dev/null -w "%{http_code}" \
  "${BASE_URL}/api/v1/physicians/notanNPI"
```
**Expected status:** `404` or `400`  
**Notes:** Shouldn't 500. If service tries to parse or use "notanNPI" directly in a DB query, check for SQL injection handling. Spring should return a clean error.

---

## 4. CRM Interactions

### TC-21 — Create interaction: visit
**Description:** Log a sales visit interaction for Kevin Robinson.

```bash
curl -s -X POST "${BASE_URL}/api/v1/interactions" \
  -H "Content-Type: application/json" \
  -d '{
    "physicianNpi": "1600024222",
    "type": "VISIT",
    "notes": "Discussed new cardio product line. Very receptive.",
    "interactionDate": "2026-03-12"
  }'
```
**Expected status:** `201`  
**Expected response:**
```json
{
  "id": "<uuid>",
  "physicianNpi": "1600024222",
  "type": "VISIT",
  "notes": "Discussed new cardio product line. Very receptive.",
  "interactionDate": "2026-03-12",
  "createdAt": "..."
}
```
**Notes:** Save the returned `id` for verification. Confirm `createdAt` is auto-populated.

---

### TC-22 — Create interaction: call
**Description:** Log a phone call interaction.

```bash
curl -s -X POST "${BASE_URL}/api/v1/interactions" \
  -H "Content-Type: application/json" \
  -d '{
    "physicianNpi": "1600024222",
    "type": "CALL",
    "notes": "Follow-up call after sample drop. Left voicemail.",
    "interactionDate": "2026-03-10"
  }'
```
**Expected status:** `201`  
**Notes:** Tests `CALL` enum value. Verify `type` is stored and returned correctly.

---

### TC-23 — Create interaction: email
**Description:** Log an email interaction.

```bash
curl -s -X POST "${BASE_URL}/api/v1/interactions" \
  -H "Content-Type: application/json" \
  -d '{
    "physicianNpi": "1600024222",
    "type": "EMAIL",
    "notes": "Sent product brochure PDF and pricing sheet.",
    "interactionDate": "2026-03-08"
  }'
```
**Expected status:** `201`  
**Notes:** Tests `EMAIL` enum value.

---

### TC-24 — Create interaction: demo
**Description:** Log a product demo interaction.

```bash
curl -s -X POST "${BASE_URL}/api/v1/interactions" \
  -H "Content-Type: application/json" \
  -d '{
    "physicianNpi": "1600024222",
    "type": "DEMO",
    "notes": "Full product demo in office. Attending: 3 staff members.",
    "interactionDate": "2026-03-05"
  }'
```
**Expected status:** `201`  
**Notes:** Tests `DEMO` enum value. After creating TC-21 through TC-24, run TC-25 to confirm they all persist.

---

### TC-25 — List interactions for a physician
**Description:** Retrieve all logged interactions for Kevin Robinson.

```bash
curl -s "${BASE_URL}/api/v1/interactions/physician/1600024222"
```
**Expected status:** `200`  
**Expected response:**
```json
[
  {
    "id": "<uuid>",
    "physicianNpi": "1600024222",
    "type": "VISIT",
    "notes": "...",
    "interactionDate": "2026-03-12"
  },
  { "type": "CALL", ... },
  { "type": "EMAIL", ... },
  { "type": "DEMO", ... }
]
```
**Notes:** After TC-21–24, should return at least 4 records. Verify ordering (most recent first?).

---

### TC-26 — Create interaction: missing required fields (400)
**Description:** POST with empty body — should fail validation.

```bash
curl -s -o /dev/null -w "%{http_code}" \
  -X POST "${BASE_URL}/api/v1/interactions" \
  -H "Content-Type: application/json" \
  -d '{}'
```
**Expected status:** `400`  
**Expected response:** Validation error with field details (e.g., `physicianNpi` is required).  
**Notes:** Controller uses `@Valid` — Spring should fire `MethodArgumentNotValidException`. Confirm error body is structured (not a 500 stack trace).

---

### TC-27 — Create interaction: invalid type value (400)
**Description:** POST with an unrecognized interaction type enum.

```bash
curl -s -o /dev/null -w "%{http_code}" \
  -X POST "${BASE_URL}/api/v1/interactions" \
  -H "Content-Type: application/json" \
  -d '{
    "physicianNpi": "1600024222",
    "type": "BRIBE",
    "notes": "This should not work.",
    "interactionDate": "2026-03-12"
  }'
```
**Expected status:** `400`  
**Notes:** `BRIBE` is not a valid enum. Jackson should fail deserialization. Confirm 400, NOT 500.

---

## 5. Favorites

### TC-28 — List favorites (fresh/empty)
**Description:** Retrieve favorites list before adding any.

```bash
curl -s "${BASE_URL}/api/v1/favorites"
```
**Expected status:** `200`  
**Expected response:**
```json
[]
```
**Notes:** On a fresh DB, should return empty array. NOT null. NOT 404.

---

### TC-29 — Add physician to favorites
**Description:** Add Kevin Robinson to favorites.

```bash
curl -s -o /dev/null -w "%{http_code}" \
  -X POST "${BASE_URL}/api/v1/favorites/1600024222"
```
**Expected status:** `201`  
**Expected response:** Empty body (201 Created, no content).  
**Notes:** No request body needed — NPI is path variable.

---

### TC-30 — Check favorite: is favorited
**Description:** Verify check endpoint returns `true` after adding.

```bash
curl -s "${BASE_URL}/api/v1/favorites/1600024222/check"
```
**Expected status:** `200`  
**Expected response:**
```json
{ "favorited": true }
```
**Notes:** Run after TC-29. Verify field name is exactly `favorited`.

---

### TC-31 — Add same physician again (idempotency)
**Description:** Add the same NPI to favorites a second time.

```bash
curl -s -o /dev/null -w "%{http_code}" \
  -X POST "${BASE_URL}/api/v1/favorites/1600024222"
```
**Expected status:** `201` (idempotent) or `409` (conflict)  
**Notes:** Both are defensible but document the actual behavior. Idempotent is preferred for a favorites feature. If this 500s, that's a critical bug.

---

### TC-32 — Check favorite: NOT in favorites (false)
**Description:** Check an NPI that has NOT been favorited.

```bash
curl -s "${BASE_URL}/api/v1/favorites/9999999999/check"
```
**Expected status:** `200`  
**Expected response:**
```json
{ "favorited": false }
```
**Notes:** Should return `false`, NOT 404. The endpoint is a check, not a lookup.

---

### TC-33 — Remove favorite
**Description:** Remove Kevin Robinson from favorites.

```bash
curl -s -o /dev/null -w "%{http_code}" \
  -X DELETE "${BASE_URL}/api/v1/favorites/1600024222"
```
**Expected status:** `204`  
**Expected response:** No content body.  
**Notes:** Run TC-30 after this to confirm `favorited` returns `false`.

---

### TC-34 — Remove non-existent favorite
**Description:** Delete an NPI that was never favorited.

```bash
curl -s -o /dev/null -w "%{http_code}" \
  -X DELETE "${BASE_URL}/api/v1/favorites/9876543210"
```
**Expected status:** `204` (idempotent) or `404`  
**Notes:** Document actual behavior. A 500 here is a critical bug. Idempotent (204) is preferred.

---

## 6. Rate Limiting

### TC-35 — Rapid-fire requests trigger 429
**Description:** Hammer the search endpoint in a loop to trigger the rate limiter.

```bash
# Fire 60 requests rapidly — rate limit is likely ~30/min or similar
for i in $(seq 1 60); do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
    "${BASE_URL}/api/v1/physicians/search?state=OK")
  echo "Request $i: $STATUS"
  if [ "$STATUS" = "429" ]; then
    echo "✅ Rate limit triggered at request $i"
    break
  fi
done
```
**Expected status:** `429` at some point in the loop  
**Expected response:**
```json
{
  "status": 429,
  "error": "Too Many Requests",
  "message": "Rate limit exceeded. Try again in X seconds."
}
```
**Notes:** Check `Retry-After` header in the 429 response. If no 429 is ever returned after 60 requests, rate limiting may not be configured — flag as a finding. Adjust loop count based on known limit config.

---

## 7. Admin / NPPES

### TC-36 — Trigger NPPES refresh (local / ADMIN role)
**Description:** Manually trigger the NPPES data refresh job.

```bash
curl -s -X POST "${BASE_URL}/api/v1/admin/nppes/refresh" \
  -H "Content-Type: application/json"
```
**Expected status:** `202`  
**Expected response:**
```json
{ "status": "NPPES refresh job launched" }
```
**Notes:** On local profile with security relaxed OR if dev stub grants ADMIN role, this should return 202. If Spring Security is still active for ADMIN-scoped endpoints, you may need `Authorization: Bearer <admin-token>` — see TC-37. Confirm the batch job actually starts (check logs).

---

### TC-37 — Trigger NPPES refresh without ADMIN role (403)
**Description:** Verify non-admin access is blocked.

```bash
# This simulates a request with no auth / non-admin context
# On local if auth is fully bypassed this may return 202 — document what actually happens
curl -s -o /dev/null -w "%{http_code}" \
  -X POST "${BASE_URL}/api/v1/admin/nppes/refresh" \
  -H "Authorization: Bearer invalid_token_here"
```
**Expected status:** `403` (forbidden) or `401` (unauthorized)  
**Notes:** Uses `@PreAuthorize("hasRole('ADMIN')")`. On full bypass, this test may not reflect production behavior — **flag this for staging re-validation**. Production must return 403/401 for non-admin callers.

---

## 8. General Error Cases

### TC-38 — Unknown route (404)
**Description:** Request a completely nonexistent path.

```bash
curl -s -o /dev/null -w "%{http_code}" \
  "${BASE_URL}/api/v1/this-does-not-exist"
```
**Expected status:** `404`  
**Expected response:**
```json
{
  "status": 404,
  "error": "Not Found",
  "path": "/api/v1/this-does-not-exist"
}
```
**Notes:** Verifies Spring's default error handling is working. Should NOT return a Whitelabel error page with stack trace. If it does — configure `server.error.include-stacktrace=never` in production.

---

## 9. Sprint 6 — DMEPOS Endpoints

> **Auth required:** All DMEPOS endpoints require a valid JWT. Obtain a token first:
> ```bash
> TOKEN=$(curl -s -X POST "http://localhost:8180/realms/medsales-dev/protocol/openid-connect/token" \
>   -H "Content-Type: application/x-www-form-urlencoded" \
>   -d "grant_type=password&client_id=medsales-api&username=testrep@medsales.dev&password=password" \
>   | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
> ```
> **Reference NPIs:**
> - `1225021702` — Brian Snell, MD (Neurological Surgery) — 1 DMEPOS item
> - `1417988957` — Jeffrey Hirsch, MD (Family Medicine) — 18 DMEPOS items

### TC-39 — DMEPOS items for Snell (1 item)
**Description:** Retrieve DMEPOS referral data for Brian Snell. Should return exactly 1 item.

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "${BASE_URL}/api/v1/physicians/1225021702/dmepos"
```
**Expected status:** `200`  
**Expected response:**
```json
{
  "npi": "1225021702",
  "items": [
    {
      "hcpcsCode": "E0745",
      "hcpcsDescription": "Neuromuscular stimulator, electronic shock unit",
      "totalClaims": 11,
      "totalBeneficiaries": 0,
      "avgMedicarePayment": <number>
    }
  ]
}
```
**Notes:** Confirm `items` array has exactly 1 element. Confirm `hcpcsCode` is `E0745`.

---

### TC-40 — DMEPOS items for Hirsch (18 items)
**Description:** Retrieve DMEPOS referral data for Jeffrey Hirsch. Should return 18 items (CPAP and wound care supplies).

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "${BASE_URL}/api/v1/physicians/1417988957/dmepos"
```
**Expected status:** `200`  
**Expected response:** `items` array with 18 elements, including `E0601` (CPAP) and `A6196` (alginate dressing).  
**Notes:** Hirsch is the primary DMEPOS test physician. Verify item count exactly = 18.

---

### TC-41 — DMEPOS for unknown NPI
**Description:** Request DMEPOS data for a valid-format NPI that doesn't exist.

```bash
curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer $TOKEN" \
  "${BASE_URL}/api/v1/physicians/9999999999/dmepos"
```
**Expected status:** `404`  
**Notes:** Should return structured error, not a 500.

---

### TC-42 — Physician detail includes dmeposSummary
**Description:** GET /physicians/:npi for Snell should include `dmeposSummary` object (not null) since he has DMEPOS data.

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "${BASE_URL}/api/v1/physicians/1225021702"
```
**Expected status:** `200`  
**Expected response (excerpt):**
```json
{
  "npi": "1225021702",
  "dmeposSummary": {
    "totalReferrals": 1,
    "totalClaims": 11,
    "totalBeneficiaries": 0,
    "topItem": "E0745 — Neuromuscular stimulator, electronic shock unit"
  }
}
```
**Notes:** `dmeposSummary` must be present and non-null. Verify `totalReferrals` = 1.

---

### TC-43 — Search by DMEPOS code (A6196)
**Description:** Filter physician search by DMEPOS HCPCS code A6196 (alginate wound dressing).

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "${BASE_URL}/api/v1/physicians/search?latitude=35.4676&longitude=-97.5164&dmeposCode=A6196&size=5"
```
**Expected status:** `200`  
**Expected response:** Results array containing only physicians who refer A6196. Verify by spot-checking NPI `/dmepos` endpoint.  
**Notes:** All returned physicians must have A6196 in their DMEPOS items — random spot-check verification recommended. Hirsch (NPI `1417988957`) should appear if within radius.

---

### TC-44 — Search by DMEPOS code + minDmeposClaims
**Description:** Filter by code AND minimum claim threshold (≥10 claims for A6196).

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "${BASE_URL}/api/v1/physicians/search?latitude=35.4676&longitude=-97.5164&dmeposCode=A6196&minDmeposClaims=10&size=5"
```
**Expected status:** `200`  
**Expected response:** Narrower result set than TC-43 — only physicians with ≥10 A6196 claims.  
**Notes:** Danley (NPI `1124193255`) has 20 A6196 claims — should appear. Physicians with <10 claims should be excluded.

---

### TC-45 — minDmeposClaims threshold filters out all results
**Description:** Set an impossibly high claim threshold to verify the filter actually excludes results.

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "${BASE_URL}/api/v1/physicians/search?latitude=35.4676&longitude=-97.5164&dmeposCode=A6196&minDmeposClaims=100&size=5"
```
**Expected status:** `200`  
**Expected response:**
```json
{ "results": [] }
```
**Notes:** Confirms minDmeposClaims filtering is not a no-op. Should return 0 results with threshold=100.

---

### TC-46 — Physician with no DMEPOS data — dmeposSummary should be null
**Description:** Fetch a physician who has no DMEPOS referral history. The `dmeposSummary` field should be null or absent.

```bash
# Use a physician known to have no DMEPOS data
curl -s -H "Authorization: Bearer $TOKEN" \
  "${BASE_URL}/api/v1/physicians/1154404648"
```
**Expected status:** `200`  
**Expected response:**
```json
{ "dmeposSummary": null }
```
**Notes:** Verifies the field is included but null (not missing entirely). Absence of the field is also acceptable — document the actual behavior.

---

### TC-47 — Invalid DMEPOS code format
**Description:** Search with a clearly invalid DMEPOS code.

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "${BASE_URL}/api/v1/physicians/search?latitude=35.4676&longitude=-97.5164&dmeposCode=INVALID_CODE&size=5"
```
**Expected status:** `200` (empty results) or `400` (validation error)  
**Notes:** Document actual behavior. An empty results set (200) is acceptable if no validation exists. A 400 with clear error message is preferred. A 500 is a bug.

---

### TC-48 — DMEPOS code search with pagination
**Description:** Verify pagination works correctly with DMEPOS code filter.

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "${BASE_URL}/api/v1/physicians/search?latitude=35.4676&longitude=-97.5164&dmeposCode=A6196&size=3"
```
**Expected status:** `200`  
**Expected response:** Exactly 3 results (or fewer if <3 match).  
**Notes:** Confirm `size` param is respected alongside DMEPOS filter.

---

## Run Order Recommendation

For a clean test run against a fresh DB:

```
TC-01 → TC-02 → TC-03          (health first)
TC-04 → TC-17                  (confirm seeded data exists)
TC-05 through TC-16            (search variations)
TC-18 through TC-20            (profile error cases)
TC-21 through TC-24            (create interactions)
TC-25                          (list — verify creates persisted)
TC-26 → TC-27                  (interaction error cases)
TC-28                          (favorites empty check)
TC-29 → TC-30 → TC-31          (add + check + add again)
TC-32 → TC-33 → TC-34          (check false, remove, remove nonexistent)
TC-35                          (rate limit — run last, may temporarily block other tests)
TC-36 → TC-37                  (admin — run after everything else)
TC-38                          (catch-all 404)
```

---

## Bug Report Template

When a test fails, use this format:

```
**[BUG] TC-XX — Short Description**
**Severity:** Critical / High / Medium / Low
**Steps:**
1. Run: curl ...
2. Observe: ...
**Expected:** HTTP XXX with { ... }
**Actual:** HTTP YYY with { ... }
**Environment:** local profile, Spring Boot 3.x, macOS
**Evidence:** [paste response body / headers]
```

---

*"I go into every feature assuming it's broken. I don't stop until I've proved it or proved it isn't. Either way, I win." — Mac*
