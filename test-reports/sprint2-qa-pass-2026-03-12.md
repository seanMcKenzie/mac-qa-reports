# Sprint 2 QA Pass — Mac McDonald
**Date:** 2026-03-12  
**Tester:** Mac McDonald  
**Stack:** API @ localhost:8080 | Keycloak @ localhost:8180 | PostgreSQL @ localhost:5432  
**Tickets Tested:** S2-01, S2-02, S2-03, S2-04, S2-05, S2-06, S2-12

---

## Summary Table

| Ticket | Name | Status | Notes |
|--------|------|--------|-------|
| S2-01 | Docker Compose Setup | ✅ PASS | Stack is live and running |
| S2-02 | Physician Data Load | ✅ PASS | 4500 physicians loaded |
| S2-03 | Keycloak JWT Auth | ✅ PASS | Token obtained, auth enforced |
| S2-04 | Physician Search API | ⚠️ PARTIAL | 2 bugs — see below |
| S2-05 | Physician Profile API | ⚠️ PARTIAL | 2 bugs — see below |
| S2-06 | Geo Distance Calc | ✅ PASS | distanceMiles present in results |
| S2-12 | Specialty Normalization | ⚠️ PARTIAL | Schema mismatch bug |

---

## S2-03 · Keycloak JWT Auth ✅ PASS

**Command:**
```bash
TOKEN=$(curl -s -X POST http://localhost:8180/realms/medsales-dev/protocol/openid-connect/token \
  -d "grant_type=password&client_id=medsales-api&username=testrep@medsales.dev&password=password" \
  | python3 -c "import sys,json; print(json.load(sys.stdin).get('access_token','FAILED'))")
echo "Token: ${TOKEN:0:40}..."
```

**Output:**
```
Token: eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUI...
```

**Unauthenticated Request Test:**
```bash
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
  "http://localhost:8080/api/v1/physicians/search?latitude=35.4676&longitude=-97.5164&radiusMiles=25")
echo "Unauthenticated response: $HTTP_CODE"
```
**Output:** `Unauthenticated response: 401`

✅ Token issued successfully with `password=password`  
✅ Unauthenticated requests correctly rejected with 401  

---

## S2-02 · Physician Data Load ✅ PASS

**Command:**
```sql
SELECT primary_specialty_desc, COUNT(*) as cnt 
FROM physicians 
GROUP BY primary_specialty_desc 
ORDER BY cnt DESC LIMIT 10;
```

**Output:**
```
 primary_specialty_desc  | cnt 
-------------------------+-----
 Internal Medicine       | 215
 Pediatrics              | 206
 Family Medicine         | 194
 Cardiology              | 194
 Orthopedic Surgery      | 192
 Dermatology             | 188
 Obstetrics & Gynecology | 184
 Psychiatry              | 181
 Anesthesiology          | 158
 Pain Medicine           | 157
```

**Total physicians:** 4500  
✅ Data loaded, specialty distribution looks healthy.

---

## S2-04 · Physician Search API ⚠️ PARTIAL

### Basic Search
```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8080/api/v1/physicians/search?latitude=35.4676&longitude=-97.5164&radiusMiles=25&page=0&size=5"
```
**Output:**
```
Top-level keys: ['results', 'page', 'size', 'totalElements', 'totalPages']
Result count: 3
First result keys: ['npi', 'firstName', 'lastName', 'credential', 'primarySpecialty', 'practiceCity', 'practiceState', 'distanceMiles', 'lastVisitDate']
```

✅ Search returns results  
✅ Specialty filter works (20 Cardiology results, 0 wrong specialty)  
✅ Text search (?q=Smith) returns results  

### 🐛 BUG-001 — P1: `totalElements` always equals `pageSize` (pagination broken)

**Steps:**
```bash
# 25mi, size=5 → totalElements: 5
# 25mi, size=50 → totalElements: 50
# 500mi, size=50 → totalElements: 50
```
**Expected:** `totalElements` = actual count of matching physicians in DB  
**Actual:** `totalElements` always mirrors the `size` param. 4500 physicians in DB, but `totalElements` never exceeds page size.  
**Severity:** P1 — pagination is completely non-functional. Any client relying on `totalElements`/`totalPages` to show results counts or navigate pages will be broken. A sales rep can't know how many docs are in their territory.

### 🐛 BUG-002 — P2: `fullName` is null in search results

**Steps:**
```
npi=1600024222, firstName=Kevin, lastName=Robinson, fullName=None
npi=1600038568, firstName=Edward, lastName=Singh, fullName=None
npi=1600029173, firstName=David, lastName=Davis, fullName=None
```
**Expected:** `fullName` = "Kevin Robinson"  
**Actual:** `fullName: null` — firstName/lastName are present but composite field not populated  
**Severity:** P2 — UI components that display `fullName` will show blank names for every physician.

---

## S2-05 · Physician Profile API ⚠️ PARTIAL

**Command:**
```bash
NPI=1600024222
curl -s -H "Authorization: Bearer $TOKEN" "http://localhost:8080/api/v1/physicians/$NPI"
```

**Output:**
```
npi: 1600024222
firstName: Kevin
lastName: Robinson
credential: DO, FACOI
primarySpecialty: General Surgery
practiceAddressLine1: 975 Poplar St
practiceCity: Oklahoma City
practiceState: OK
practiceZip: 73101
practicePhone: (899) 1714231
latitude: 35.46791375505617
longitude: -97.51567712239326
crmSummary: {'callCount': 0, 'lastCallDate': None, 'openTasks': 0, 'openOpportunities': 0}
```

✅ Profile lookup by NPI works  
✅ CRM summary block returned  
✅ Invalid NPI returns 404  

### 🐛 BUG-003 — P2: `fullName` is null in profile response

Same as BUG-002 — composite fullName field not computed server-side.  
**Expected:** `fullName: "Kevin Robinson"`  
**Actual:** `fullName: null`  
**Severity:** P2

### 🐛 BUG-004 — P3: `city` field is null in profile response

**Expected:** `city: "Oklahoma City"`  
**Actual:** `city: null` — field is stored as `practiceCity` in response but API contract specifies `city`  
**Severity:** P3 — field name mismatch. Frontend mapping will fail unless it falls back to `practiceCity`.

---

## S2-12 · Specialty Normalization ⚠️ PARTIAL

**Specialty filter accuracy:**
```
Specialty filter: Total: 20, Wrong specialty: 0
```
✅ Filter is accurate — zero wrong-specialty results in 20-result sample

**Mapping table exists:** 68 records ✅

### 🐛 BUG-005 — P2: `specialty_mapping` schema doesn't match spec

**Expected columns:** `raw_specialty`, `normalized_specialty`  
**Actual schema:**
```
 Column       | Type
--------------+------
 code         | text  (PK)
 display_name | text
 color_hex    | text
 category     | text
```

The test spec and presumably app code reference `raw_specialty` / `normalized_specialty` columns which do not exist. The table has been implemented with a different schema (`code`/`display_name`). Either the spec is wrong, the migration is wrong, or both. Needs reconciliation.  
**Severity:** P2 — any code path that queries `raw_specialty` or `normalized_specialty` will throw a SQL error at runtime.

---

## S2-01 · Docker Compose Setup ✅ PASS (inferred)

Stack is fully operational:  
- API responding at :8080 ✅  
- Keycloak responding at :8180 ✅  
- PostgreSQL responding (via docker exec) ✅  
- medsales-postgres container running ✅  

---

## S2-06 · Geo Distance Calculation ✅ PASS

`distanceMiles` field present in all search results. Physicians returned are within requested radius. No results returned outside radius in manual spot checks.

---

## Bug Summary

| Bug ID | Ticket | Severity | Description |
|--------|--------|----------|-------------|
| BUG-001 | S2-04 | **P1** | `totalElements` always equals `pageSize` — pagination broken |
| BUG-002 | S2-04 | **P2** | `fullName` null in search results |
| BUG-003 | S2-05 | **P2** | `fullName` null in profile response |
| BUG-004 | S2-05 | **P3** | `city` field null (stored as `practiceCity`) |
| BUG-005 | S2-12 | **P2** | `specialty_mapping` schema mismatch (`raw_specialty` col missing) |

---

## Notes

- Response envelope uses `results` key (not Spring's standard `content`). Non-standard but consistent. Not a bug, just worth noting for frontend team.
- No `number` field in pagination response (current page number). Spring Page standard includes this; it's missing here.
- `practicePhone` format `(899) 1714231` looks malformed — 7 digits after area code instead of 7 with dash. Low priority but worth flagging.
- Token expiry is 5 minutes (standard). Fine for dev. Production should be reviewed.
