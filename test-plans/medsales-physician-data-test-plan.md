# QA Test Plan: Physician Data Accuracy, Search & Filter Validation
## Medical Sales Intelligence & CRM Platform

**Author:** Mac McDonald, QA Engineer  
**Date:** March 1, 2026  
**Version:** 1.0  
**Status:** Active  
**Task ID:** 20  
**Project:** MedSales Intelligence & CRM Platform  

---

## 1. Overview & Scope

This test plan covers validation of the physician data layer — the foundation upon which every feature in this platform rests. If the data is wrong, nothing else matters. A sales rep hands this to a physician during a call and says "the app told me you prescribe Eliquis." If that's wrong, the rep looks like an idiot and we lose a customer.

I'm not letting that happen.

### 1.1 In Scope

- NPI data validation (format, Luhn checksum, deduplication)
- Physician profile field accuracy against raw CMS source files
- Specialty normalization and taxonomy mapping
- Geospatial radius search accuracy (PostGIS / ST_DWithin)
- All 13 filter types from FR-015
- Search sort order correctness
- Search response time under load (NFR-001, NFR-002)
- NPPES delta/incremental update processing (weekly deltas)
- Open Payments NPI match rate measurement
- Data freshness indicator accuracy (FR-010)

### 1.2 Out of Scope (for this test plan)

- CRM call logging, tasks, opportunities (separate QA cycle)
- Route export / AI prompt feature (separate test plan)
- Mobile UI rendering
- Authentication / RBAC (security test plan)

### 1.3 References

| Document | Author |
|---|---|
| Medical Sales Platform PRD v1.1 | Dennis Reynolds |
| FR-001 through FR-018 | PRD Section 3.1–3.3 |
| NFR-001, NFR-002, NFR-005, NFR-030, NFR-031 | PRD Section 4.1, 4.7 |
| RISK-001, RISK-002, RISK-005, RISK-006 | PRD Section 7 |

---

## 2. Test Data Requirements

### 2.1 Source Files Required

The following raw CMS source files must be available in a test environment before execution. Do not run these tests against production source files unless explicitly approved.

| Dataset | Version | Storage Location |
|---|---|---|
| NPPES NPI Full Replacement Download | Most recent monthly | `/test-data/nppes/npi_full_YYYYMMDD.csv` |
| NPPES NPI Weekly Incremental (delta) | 2 consecutive weeks | `/test-data/nppes/delta_YYYYMMDD.csv` |
| CMS Provider Data Catalog — National Downloadable File | Most recent quarterly | `/test-data/cms-pdc/ndf_YYYYQQ.csv` |
| CMS Facility Affiliation Data | Most recent quarterly | `/test-data/cms-fad/fad_YYYYQQ.csv` |
| Hospital Compare General Information | Most recent quarterly | `/test-data/hospital-compare/hcgi_YYYYQQ.csv` |
| Medicare Part B — By Provider & Service | Most recent annual | `/test-data/partb/partb_provider_service_YYYY.csv` |
| Medicare Part D — By Provider & Drug | Most recent annual | `/test-data/partd/partd_provider_drug_YYYY.csv` |
| Open Payments — General | Most recent annual | `/test-data/openpay/op_general_YYYY.csv` |
| Open Payments — Research | Most recent annual | `/test-data/openpay/op_research_YYYY.csv` |

### 2.2 Reference NPI Test Set

Before testing, assemble a **reference set of 50 NPIs** with the following characteristics. These are used for spot-check validation (TC-DA-001 through TC-DA-010).

| # | Criteria | Count |
|---|---|---|
| 1 | Active, individual physician, primary specialty = Internal Medicine | 10 |
| 2 | Active, individual physician, primary specialty = Orthopedic Surgery | 10 |
| 3 | Active, individual physician, Part D prescriber of Eliquis (≥ 25 claims) | 5 |
| 4 | Active, individual physician, Open Payments recipient (> $10,000 general) | 5 |
| 5 | Active, individual physician, Part B: CPT 27447 billed ≥ 50 services | 5 |
| 6 | Active, individual physician, multi-site practice (≥ 2 address rows in NDF) | 5 |
| 7 | Inactive / deactivated NPI (status = Deactivated in NPPES) | 5 |
| 8 | NPI with mismatched address between NPPES and Provider Data Catalog | 5 |

Document each NPI, expected field values (pulled directly from raw CSV), and expected system display. This is your ground truth.

### 2.3 Reference Geospatial Points

Define 5 reference geolocation points for radius search testing:

| ID | Location | Lat | Lng | Known nearby physicians (within 5 miles) |
|---|---|---|---|---|
| GEO-01 | Mayo Clinic, Rochester MN | 44.0225 | -92.4669 | ≥ 50 (high density urban medical campus) |
| GEO-02 | Rural Nebraska (open farmland) | 41.5868 | -99.6384 | < 5 (low density, validates sparse results) |
| GEO-03 | Miami Beach, FL | 25.7907 | -80.1300 | ≥ 30 (urban coastal, boundary test) |
| GEO-04 | US-Canada border, northern MN | 48.9990 | -95.0000 | Tests that results don't cross international border |
| GEO-05 | Downtown Chicago, IL | 41.8827 | -87.6233 | ≥ 200 (very high density, stress test for pin rendering) |

Pre-count actual physicians within each radius band (1, 5, 10, 25, 50 miles) using a direct PostGIS query against the database. These counts become your expected results for geospatial test cases.

### 2.4 Specialty Normalization Reference Set

Compile a list of 200 NUCC taxonomy codes and their expected plain-English mappings for TC-SN-001 through TC-SN-005. Use the NUCC Health Care Provider Taxonomy Code Set (published at nucc.org) as the authoritative reference.

Include edge cases:
- Taxonomy codes that exist in NPPES but are not in the NUCC published set (obsolete codes)
- Codes that map to multiple possible display names (e.g., "Internal Medicine" vs "General Internal Medicine")
- Pediatric sub-specialties
- Non-physician providers (NP, PA, CRNA) — verify they appear correctly in credential/specialty display

---

## 3. Test Cases — Data Accuracy

### TC-DA-001: NPI Format Validation on Import

**Objective:** Verify all imported NPIs are 10-digit strings passing Luhn checksum (NFR-031).

**Setup:**
1. Prepare a synthetic test CSV with the following NPI variants:
   - 5 valid NPIs (format + Luhn pass)
   - 3 NPIs with fewer than 10 digits (9-digit, 7-digit, 1-digit)
   - 3 NPIs with more than 10 digits (11-digit)
   - 3 NPIs with non-numeric characters ("ABC1234567", "123456789X")
   - 2 NPIs that are exactly 10 digits but fail Luhn checksum (crafted to fail)
   - 1 NPI that is all zeros: "0000000000"
   - 1 blank/null NPI

**Steps:**
1. Trigger the NPPES import pipeline with this synthetic CSV.
2. Query the database for each test NPI.
3. Query the quarantine/reject log.

**Expected:**
- 5 valid NPIs → ingested, present in Physician table.
- All 13 invalid NPIs → quarantined, NOT in Physician table.
- Quarantine log contains one entry per invalid NPI with reason code (format vs. Luhn).
- Import process does NOT abort on invalid NPIs — it logs and continues.
- System alert/notification is generated for the quarantined records (import report).

**Pass Criteria:** All 5 valid NPIs ingested. All 13 invalid NPIs quarantined with correct reason codes. Zero invalid NPIs in production Physician table.

---

### TC-DA-002: NPPES Field Accuracy — Identity Fields

**Objective:** Verify physician identity fields on 50 reference NPIs match raw NPPES CSV exactly (FR-002).

**Steps:**
1. For each of the 50 reference NPIs, retrieve the physician profile via API.
2. For each NPI, compare the following API response fields against raw NPPES CSV values:
   - `npi`
   - `last_name`, `first_name`, `middle_name`, `name_suffix`
   - `credential` (MD, DO, NP, PA, etc.)
   - `gender`
   - `enumeration_date`
   - `last_updated_nppes`
   - `sole_proprietor_indicator`
   - Primary practice address: street1, street2, city, state, zip, phone, fax

**Expected:** Each field value matches the raw CSV value exactly, including capitalization and punctuation (system may normalize casing — document any normalization rules applied).

**Pass Criteria:** ≥ 99% field-level accuracy across all 50 NPIs × all identity fields. Zero NPIs displaying a different NPI number than requested.

---

### TC-DA-003: Provider Data Catalog Field Accuracy

**Objective:** Verify CMS Provider Data Catalog fields on 50 reference NPIs match raw NDF CSV (FR-003).

**Fields to verify:**
- `group_name`, `org_pac_id`
- `num_org_members`
- `medical_school`, `grad_year`
- `primary_specialty_cms`, `secondary_specialty_cms`
- `accepts_telehealth` (Y/N)
- `ind_assignment` (Y/N/M)
- `grp_assignment` (Y/N/M)

**Steps:**
1. Retrieve each of the 50 reference NPIs from the API (physician profile).
2. Look up each NPI in the raw NDF CSV.
3. Compare field-by-field.

**Special cases:**
- For the 5 multi-site NPIs: verify the NDF has multiple rows and all group associations are displayed.
- For physicians not present in NDF: verify profile displays gracefully (nulls/blanks, not errors).

**Pass Criteria:** ≥ 99% field-level accuracy. Multi-site NPIs display all group associations. Missing NDF records show blank fields, not 500 errors.

---

### TC-DA-004: Part B Data Accuracy

**Objective:** Verify Part B procedure data for 20 reference NPIs matches raw CMS Part B CSV (FR-006).

**Steps:**
1. Pick 20 NPIs from reference set that have Part B data.
2. Retrieve Part B section from physician profile API.
3. Compare for each HCPCS line:
   - `hcpcs_code`, `hcpcs_desc`
   - `total_beneficiaries`
   - `total_services`
   - `total_submitted_charges`
   - `total_medicare_payment`

**Expected:** All numeric values match raw CSV. Data year displayed matches source file year. No rounding errors beyond 2 decimal places on currency.

**Special case:** NPI billed under both individual and group — verify individual-level data is shown (system must not double-count group billing).

**Pass Criteria:** ≥ 99% accuracy on all numeric fields. Data year label correct on all 20 NPIs.

---

### TC-DA-005: Part D Data Accuracy

**Objective:** Verify Part D drug prescribing data for 20 reference NPIs matches raw CMS Part D CSV (FR-007).

**Steps:**
1. Pick 20 NPIs from reference set that have Part D data.
2. Retrieve Part D section from physician profile API.
3. Compare for each drug line:
   - `brand_name`, `generic_name`
   - `total_claims`
   - `total_30day_fills`
   - `total_drug_cost`
   - `total_beneficiaries`
   - `opioid_flag`

**Pass Criteria:** ≥ 99% accuracy. Opioid flag correct on all records. Data year label present and correct.

---

### TC-DA-006: Open Payments Data Accuracy

**Objective:** Verify Open Payments data for 20 reference NPIs matches raw Open Payments General CSV (FR-008, FR-009).

**Steps:**
1. Pick 20 NPIs from reference set that have Open Payments records.
2. Retrieve Open Payments section from physician profile API.
3. For each transaction line compare:
   - `manufacturer_name`
   - `payment_amount`
   - `payment_date`
   - `nature_of_payment`
   - `product_name`
4. Verify the rolled-up annual total matches the sum of individual transactions.
5. Verify lifetime total (FR-009) matches sum across all program years.

**Pass Criteria:** Individual transaction fields match. Rolled-up totals are arithmetically correct (no off-by-penny rounding). Lifetime total covers all years present in source files.

---

### TC-DA-007: NPI Deduplication Across Datasets

**Objective:** Verify no duplicate physician records exist in the master Physician table after ingestion from multiple datasets (NFR-030).

**Steps:**
1. Run SQL query: `SELECT npi, COUNT(*) as cnt FROM physician GROUP BY npi HAVING COUNT(*) > 1;`
2. For 10 NPIs that appear in multiple source files (NPPES + NDF + Part B + Open Payments), verify only one master record exists.
3. Verify all child records (PhysicianTaxonomy, PartBService, PartDDrug, etc.) correctly reference the single master NPI.

**Pass Criteria:** Zero duplicate NPIs in Physician table. All cross-dataset fields visible on single profile. No "phantom" profiles created from secondary dataset joins.

---

### TC-DA-008: Inactive NPI Handling

**Objective:** Verify deactivated NPIs are correctly excluded from search results but accessible directly (if applicable per product decision).

**Steps:**
1. Use the 5 inactive reference NPIs.
2. Search by name, specialty, and location — verify inactive NPIs do NOT appear in search results.
3. Attempt direct profile access by NPI number — document behavior (404 vs. profile with "Inactive" banner — get product decision before running).
4. Verify inactive NPIs are not counted in search result totals.

**Pass Criteria:** Inactive NPIs excluded from all search result sets. Behavior on direct access matches product specification (document if spec is not yet defined — flag as open issue).

---

### TC-DA-009: NPPES vs. Provider Data Catalog Address Discrepancy Detection

**Objective:** Verify system flags records where NPPES and Provider Data Catalog addresses differ (RISK-002).

**Steps:**
1. Identify 10 NPIs in reference set where NPPES address != NDF address.
2. Retrieve each profile from API.
3. Verify discrepancy flag is present in API response.
4. Verify both addresses are displayed (NPPES address + NDF address).

**Pass Criteria:** All 10 discrepancy records display both addresses with source labels. Discrepancy flag present in API response field. No NPIs silently suppressing one address.

---

### TC-DA-010: Data Freshness Indicator Accuracy

**Objective:** Verify FR-010 — the data freshness indicator shows correct dataset names and last ingestion timestamps.

**Steps:**
1. After a fresh ingestion run, retrieve freshness indicators for 10 NPIs.
2. Verify each section shows:
   - Source dataset name (e.g., "NPPES NPI Registry")
   - Last update date (matches ingestion log timestamp)
3. Trigger a re-ingestion of one dataset (e.g., Part D). Wait for completion.
4. Verify the Part D freshness indicator updates. Verify other sections are unchanged.

**Pass Criteria:** All 6 data sections display correct source names. Timestamps match ingestion log. Re-ingestion updates only the affected section's timestamp.

---

## 4. Test Cases — Specialty Normalization

### TC-SN-001: NUCC Taxonomy Code Decoding

**Objective:** Verify 200 NUCC taxonomy codes are decoded correctly to plain-English specialty names (RISK-005, FR-002).

**Steps:**
1. Submit each of the 200 taxonomy codes from the reference set via an internal taxonomy lookup endpoint (or direct DB query).
2. Compare decoded name against NUCC reference document.

**Pass Criteria:** ≥ 99.5% accuracy (≥ 199 of 200 codes decode correctly).

---

### TC-SN-002: Multi-Specialty Display

**Objective:** Verify all secondary specialties (up to 14) display correctly for physicians with multiple taxonomy codes (FR-002).

**Steps:**
1. Find 5 physicians with 3+ taxonomy codes in NPPES.
2. Retrieve profile, verify all taxonomy entries are present in the secondary specialties list.
3. Verify `is_primary = true` is only set on one taxonomy per NPI.

**Pass Criteria:** All secondary specialties present. Exactly one primary specialty per physician.

---

### TC-SN-003: CMS vs. NPPES Specialty Consistency

**Objective:** Verify that specialty filter returns consistent results whether using NPPES taxonomy or CMS NDF specialty name (RISK-005).

**Steps:**
1. Search for "Internal Medicine" — count results using NPPES taxonomy decode.
2. Search for "Internal Medicine" — count results using CMS NDF specialty.
3. Verify difference is ≤ 2% (acceptable due to providers in one dataset but not the other).
4. Verify no physician appears as "Internal Medicine" under NPPES and "Cardiology" under CMS (data integrity check for 10 sampled records).

**Pass Criteria:** Result count difference ≤ 2%. No systematic specialty mismatches on sampled records.

---

### TC-SN-004: Open Payments Specialty Category Mapping

**Objective:** Verify Open Payments "Category|Specialty" pipe-delimited format is parsed and normalized correctly (RISK-005).

**Steps:**
1. Pull 50 Open Payments records with various category|specialty combinations.
2. Verify the Category and Specialty components are parsed and stored separately.
3. Verify physicians who have Open Payments specialty but no NPPES specialty display gracefully.

**Pass Criteria:** Pipe-delimited format parsed correctly on all 50 records. No parse errors in logs.

---

### TC-SN-005: Obsolete Taxonomy Code Handling

**Objective:** Verify obsolete NUCC taxonomy codes (present in old NPPES data but removed from current NUCC set) are handled without crashing.

**Steps:**
1. Identify 5 NPIs with taxonomy codes not in the current NUCC set.
2. Retrieve profiles — verify they display "Specialty: Unclassified (code: XXXXXXXXX)" or equivalent, not a blank or error.

**Pass Criteria:** Obsolete codes display a graceful fallback label. No 500 errors. No null specialty on profile page.

---

## 5. Test Cases — Geospatial Search

### TC-GEO-001: Radius Search Accuracy — Known Dense Urban Area

**Objective:** Verify radius search returns the correct set of physicians within a given radius of GEO-01 (Mayo Clinic, Rochester MN).

**Steps:**
1. Execute search: `GET /api/physicians/search?lat=44.0225&lng=-92.4669&radius=5&unit=miles`
2. Compare result count against pre-computed PostGIS reference count.
3. For each result, verify `distance_miles` field is ≤ 5.0.
4. Verify no physicians outside 5 miles are included.
5. Repeat with radius = 1, 10, 25, 50 miles.

**Expected:** Result counts match reference counts ± 0 (exact match). All returned physicians have `distance_miles ≤ radius`.

**Pass Criteria:** 100% of returned physicians within specified radius. Zero false positives (physicians outside radius). Count matches reference within ± 1% (accounting for geocoding rounding).

---

### TC-GEO-002: Radius Search — Rural Sparse Area

**Objective:** Verify radius search handles sparse results correctly (GEO-02, rural Nebraska).

**Steps:**
1. Execute search: `GET /api/physicians/search?lat=41.5868&lng=-99.6384&radius=5&unit=miles`
2. Verify result count matches reference (expected < 5).
3. Execute with radius = 50 miles — verify reasonable result count.
4. Verify response time remains within NFR-002 (≤ 3 seconds) even for sparse results.

**Pass Criteria:** Correct result count. No phantom results. Response time ≤ 3 seconds.

---

### TC-GEO-003: Boundary Edge Cases — Exact Radius Boundary

**Objective:** Verify physicians exactly AT the radius boundary are included/excluded consistently.

**Steps:**
1. Identify 5 physicians whose geocoded location is within ±0.1 miles of a 10-mile radius from GEO-05 (Chicago).
2. Query radius = 10 miles. Document which boundary physicians are included.
3. Query radius = 9.9 miles. Verify boundary physicians are excluded.
4. Query radius = 10.1 miles. Verify boundary physicians are included.

**Pass Criteria:** Boundary behavior is consistent and predictable. Inclusion threshold is clearly defined and matches specification (≤ radius, not < radius).

---

### TC-GEO-004: International Border Non-Crossing

**Objective:** Verify radius search at GEO-04 (US-Canada border) does not return Canadian physicians.

**Steps:**
1. Execute search at GEO-04 with radius = 50 miles.
2. Verify all returned physicians have US addresses (state field contains valid US state abbreviation).
3. Verify no addresses in Canadian provinces (ON, MB, QC, etc.).

**Pass Criteria:** Zero non-US physician records returned. All state values are valid US states.

---

### TC-GEO-005: Lat/Lng Precision Validation

**Objective:** Verify geocoded lat/lng values are stored at sufficient precision to ensure < 10-meter radius error.

**Steps:**
1. Pull `geo_lat` and `geo_lng` for 20 reference physicians from the database.
2. Verify decimal precision is ≥ 6 places (e.g., 44.022523, -92.466914).
3. Calculate theoretical radius error for 6-decimal-place precision at mid-latitude (~44°N): should be < 0.2 meters. This is acceptable.
4. Verify 4-decimal-place values (theoretical ~11-meter error) are NOT present — flag any.

**Pass Criteria:** All geocoded coordinates stored at ≥ 6 decimal places. Zero 4-decimal precision values in Physician or Hospital address tables.

---

### TC-GEO-006: Null Geocode Handling

**Objective:** Verify physicians with failed geocoding (null lat/lng) are excluded from radius search without crashing.

**Steps:**
1. Artificially null the `geo_lat`/`geo_lng` for 5 test NPIs in a test environment.
2. Execute radius search that would otherwise include those 5 physicians.
3. Verify those 5 NPIs are not in results (expected — they can't be located).
4. Verify no 500 error, no query crash.
5. Verify a count of un-geocoded records is accessible via admin dashboard.

**Pass Criteria:** Null geocode records excluded from radius search. No crashes. Admin visibility into un-geocoded count.

---

### TC-GEO-007: Distance Calculation Accuracy

**Objective:** Verify the distance calculation (ST_DWithin / Haversine) is accurate within 0.1 miles.

**Steps:**
1. Pick 10 physicians from the reference set with known addresses.
2. Calculate expected distance from GEO-05 (Chicago) using Google Maps API (reference truth).
3. Compare Google Maps distance against `distance_miles` in API response.

**Pass Criteria:** All 10 distances within ±0.1 miles of Google Maps reference. No distance errors > 0.5 miles.

---

## 6. Test Cases — Search Filters (FR-015)

The system supports 13 distinct filter types per FR-015. Each filter must be tested individually and in combination with geography.

### TC-FILT-001: Geography Filter — State

**Steps:**
1. Search with `state=MN`. Verify all results have `state = MN` in primary address.
2. Search with `state=CA`. Verify all results have `state = CA`.
3. Search with invalid state code `state=XX`. Verify 400 error with clear message.

**Pass Criteria:** All results match requested state. Invalid state returns 400, not empty results.

---

### TC-FILT-002: Geography Filter — ZIP Code

**Steps:**
1. Search with `zip=90210`. Verify all results have `zip5 = 90210` in primary address.
2. Search with 9-digit ZIP `zip=90210-1234`. Verify system normalizes to 5-digit match.
3. Search with non-existent ZIP `zip=00001`. Verify zero results, not an error.

**Pass Criteria:** Results match ZIP. 9-digit input normalizes. Non-existent ZIP returns empty, not crash.

---

### TC-FILT-003: Geography Filter — City

**Steps:**
1. Search with `city=Minneapolis&state=MN`. Verify all results have city = Minneapolis, MN.
2. Search with `city=springfield` (no state). Verify results from ALL Springfields in all states.
3. Search with `city=CHICAGO` (all caps). Verify case-insensitive match.

**Pass Criteria:** City+state combination filters correctly. City-only returns multi-state results. Case-insensitive.

---

### TC-FILT-004: Geography Filter — Radius

*(See TC-GEO-001 through TC-GEO-007 for full geospatial coverage. This test focuses on the filter parameter itself.)*

**Steps:**
1. Submit radius < 1 mile (e.g., 0). Verify 400 error (minimum 1 mile per FR-015).
2. Submit radius > 100 miles. Verify 400 error (maximum 100 miles per FR-015).
3. Submit radius = 1 mile. Verify tight results.
4. Submit radius = 100 miles. Verify wide results.
5. Submit radius without lat/lng. Verify 400 error with guidance message.

**Pass Criteria:** Range enforcement (1–100 miles). Missing lat/lng returns actionable error.

---

### TC-FILT-005: Specialty Filter — Single Select

**Steps:**
1. Search `specialty=Cardiology`. Count results.
2. Pull 10 random results — verify each has primary specialty = Cardiology.
3. Verify results don't include physicians whose secondary specialty is Cardiology but primary is something else (unless the spec says secondary matches are included — get spec clarification).

**Pass Criteria:** All returned physicians match requested specialty. Behavior on primary-only vs. any-specialty match is consistent with spec.

---

### TC-FILT-006: Specialty Filter — Multi-Select

**Steps:**
1. Search `specialty=Cardiology,Pulmonology`. Verify results include physicians of either specialty.
2. Verify no results from unrelated specialties (e.g., Dermatology) appear.
3. Search with 10 specialties simultaneously. Verify union behavior.

**Pass Criteria:** Multi-select returns union (OR), not intersection. No out-of-set results.

---

### TC-FILT-007: Credential Filter

**Steps:**
1. Search `credential=MD`. Verify all results have `credential = MD`.
2. Search `credential=NP`. Verify all results have `credential = NP` or `credential = NP-C` (etc. — define accepted variants).
3. Search `credential=MD,DO`. Verify union of MDs and DOs.
4. Search `credential=WIZARD`. Verify 400 error (invalid credential).

**Pass Criteria:** Credential filter returns matching credentials only. Invalid credential returns 400.

---

### TC-FILT-008: Gender Filter

**Steps:**
1. Search `gender=M`. Verify all results have `gender = M`.
2. Search `gender=F`. Verify all results have `gender = F`.
3. Verify physicians with null/unknown gender are not returned under either filter (or are returned under both — document behavior per spec).

**Pass Criteria:** Gender filter returns correct gender only. Null-gender handling is consistent with spec.

---

### TC-FILT-009: Medicare Assignment Filter

**Steps:**
1. Search `assignment=Y`. Verify all results have `ind_assignment = Y`.
2. Search `assignment=N`. Verify all results have `ind_assignment = N`.
3. Search `assignment=M` (Modified). Verify correct results.
4. Combine with specialty filter: `specialty=Cardiology&assignment=Y`.

**Pass Criteria:** Assignment filter correct in isolation and combined. No mixed assignment values in results.

---

### TC-FILT-010: Telehealth Filter

**Steps:**
1. Search `telehealth=true`. Verify all results have `accepts_telehealth = true`.
2. Search `telehealth=false`. Verify all results have `accepts_telehealth = false`.
3. Combine with geography: `state=TX&telehealth=true`.

**Pass Criteria:** Telehealth filter correct. Combined filter returns intersection (AND behavior).

---

### TC-FILT-011: Group Practice Filter

**Steps:**
1. Search by group name: `group_name=Mayo+Clinic`. Verify all results belong to a practice with a matching group name (partial match or exact — document).
2. Search by org PAC ID: `org_pac_id=0042100570`. Verify results all share that PAC ID.
3. Search with non-existent PAC ID. Verify empty results, not error.

**Pass Criteria:** Group name returns matching practices. PAC ID returns exact match. Non-existent returns empty.

---

### TC-FILT-012: Hospital Affiliation Filter

**Steps:**
1. Search `hospital_name=Cleveland+Clinic`. Verify all results have an affiliation with Cleveland Clinic.
2. Search by CCN `hospital_ccn=360180`. Verify all results have that CCN affiliation.
3. Verify that physicians with NO hospital affiliations are excluded from results when hospital filter is applied.

**Pass Criteria:** All results have matching hospital affiliation. Physicians without affiliations correctly excluded.

---

### TC-FILT-013: Part B Procedure Volume Filter

**Objective:** Verify filter for HCPCS code + minimum services threshold (FR-015).

**Steps:**
1. Search `hcpcs=27447&min_services=50` (total knee replacement, ≥ 50 services).
2. Retrieve raw Part B data for 10 random results. Verify each has HCPCS 27447 with total_services ≥ 50.
3. Search with `min_services=0`. Verify all physicians who billed 27447 appear (including 1-service billers).
4. Search `hcpcs=ZZZZZ&min_services=1`. Verify 400 or empty results (invalid code).
5. Search `hcpcs=27447&min_services=10000`. Verify zero results (threshold too high to match anyone).

**Pass Criteria:** All results have the specified HCPCS with ≥ min_services. Min of 0 returns all billers. Invalid code returns 400. Impossible threshold returns empty, not error.

---

### TC-FILT-014: Part D Drug Prescribing Filter

**Objective:** Verify filter for drug name + minimum claim count (FR-015).

**Steps:**
1. Search `drug=Eliquis&min_claims=25`. Verify all results prescribed Eliquis with ≥ 25 claims.
2. Search by generic name: `drug=apixaban&min_claims=25`. Verify same/similar results as brand name.
3. Search `drug=Eliquis&min_claims=0`. Verify all Eliquis prescribers included.
4. Combine with specialty: `drug=Eliquis&min_claims=25&specialty=Cardiology`.

**Pass Criteria:** Brand and generic name lookup both work. Min claims threshold enforced. Combined filter works (AND behavior).

---

### TC-FILT-015: Open Payments — Total Amount Filter

**Steps:**
1. Search `min_payments=10000`. Verify all results have lifetime Open Payments total ≥ $10,000.
2. Pull 10 results, verify amounts from raw Open Payments CSV sum to ≥ $10,000.
3. Search `min_payments=0`. Verify all physicians with any Open Payments record appear.
4. Search `min_payments=1000000`. Verify small result set (only very high payment recipients).

**Pass Criteria:** All results meet minimum total. Summation is accurate within $0.01.

---

### TC-FILT-016: Open Payments — By Company Filter

**Steps:**
1. Search `payment_company=Pfizer`. Verify all results have at least one Open Payments record from Pfizer.
2. Verify physicians with ONLY non-Pfizer payments are NOT in results.
3. Search with exact vs. partial company name — document behavior (full-text match or exact).

**Pass Criteria:** Filter returns only physicians with matching company payments. No false positives.

---

### TC-FILT-017: Open Payments — Nature of Payment Filter

**Steps:**
1. Search `payment_nature=Consulting+Fee`. Verify all results have at least one consulting fee payment.
2. Search `payment_nature=Speaker+Fee`. Verify speaker fee recipients only.
3. Combine: `payment_nature=Speaker+Fee&min_payments=5000`.

**Pass Criteria:** Nature filter returns correct subset. Combined with amount filter, intersection is correct (AND).

---

### TC-FILT-018: Combined Filter Stress Test

**Objective:** Verify multiple filters combine correctly with AND logic.

**Steps:**
1. Execute: `specialty=Cardiology&state=NY&gender=M&assignment=Y&min_payments=10000`
2. Retrieve 10 results. Manually verify EVERY condition is true for each result.
3. Execute with 7 filters simultaneously (max combination).
4. Verify response time ≤ NFR-002 (3 seconds for geography queries; use 5 seconds as acceptable limit for heavily filtered non-geo queries).

**Pass Criteria:** All conditions satisfied in all results. No filter is silently ignored. Response time acceptable.

---

### TC-FILT-019: Sort Order Validation (FR-016)

**Steps:**
1. Search `specialty=Cardiology&state=MN&sort=name_asc`. Verify alphabetical ascending A→Z.
2. Sort `name_desc`. Verify Z→A.
3. Sort `distance_asc` (with lat/lng in request). Verify nearest first.
4. Sort `partb_volume_desc`. Verify highest total_services first; pull raw Part B data for top 5 to confirm.
5. Sort `partd_claims_desc`. Verify highest total_claims first.
6. Sort `payments_desc`. Verify highest lifetime Open Payments first.
7. Sort by field without providing required parameter (e.g., `sort=distance_asc` with no lat/lng). Verify 400 error.

**Pass Criteria:** All 6 sort options produce correctly ordered results. Invalid sort combination returns 400.

---

## 7. Test Cases — Performance

### TC-PERF-001: Physician Profile Load Time (NFR-001)

**Objective:** Profile pages load within 2 seconds on simulated 4G connection.

**Setup:** Use Apache JMeter or k6. Configure network throttle to simulate 4G (≈ 15 Mbps down, 8 Mbps up, 40ms latency).

**Steps:**
1. Send 100 sequential profile requests for the 50 reference NPIs (alternating).
2. Record p50, p95, p99 response times.
3. Load test: 500 concurrent users each requesting random physician profiles.

**Pass Criteria:**
- p95 response time ≤ 2,000ms
- p99 response time ≤ 4,000ms (1 standard deviation buffer)
- Zero 5xx errors under 500 concurrent users

---

### TC-PERF-002: Radius Search Response Time (NFR-002)

**Objective:** Radius search returns within 3 seconds for radii up to 50 miles.

**Steps:**
1. Execute radius searches at GEO-01, GEO-03, GEO-05 (high density) for each radius band: 1, 5, 10, 25, 50 miles.
2. Record response time for each.
3. Load test: 200 concurrent radius search requests at 50-mile radius from GEO-05 (worst case: maximum radius, maximum density).

**Pass Criteria:**
- All individual search responses ≤ 3,000ms
- Under 200 concurrent: p95 ≤ 3,000ms, p99 ≤ 5,000ms
- Zero 5xx errors

---

### TC-PERF-003: Heavily Filtered Search Performance

**Objective:** Multi-filter queries perform within acceptable bounds.

**Steps:**
1. Execute a 5-filter query (specialty + state + credential + assignment + min_payments).
2. Execute a 7-filter query (max tested combination).
3. Load test: 100 concurrent 5-filter queries.

**Pass Criteria:**
- Single 5-filter query ≤ 5,000ms
- Single 7-filter query ≤ 8,000ms
- Under 100 concurrent: p95 ≤ 5,000ms

---

### TC-PERF-004: 10,000 Concurrent User Load (NFR-005)

**Objective:** System sustains 10,000 concurrent active users without degradation.

**Setup:** Requires load testing infrastructure (k6, Gatling, or AWS load testing). Coordinate with DevOps. Run against staging environment with production data volume.

**Steps:**
1. Ramp from 0 → 10,000 concurrent users over 10 minutes.
2. Hold at 10,000 for 30 minutes.
3. Mix of operations: 40% profile views, 40% radius searches, 20% filtered searches.
4. Monitor: response times, error rates, database connection pool saturation, CPU/memory.

**Pass Criteria:**
- Profile pages: p95 ≤ 2,000ms at full load
- Radius search: p95 ≤ 3,000ms at full load
- Error rate < 0.1%
- No memory leaks over 30-minute hold
- Database CPU < 80% at peak

---

### TC-PERF-005: NPPES Full File Ingestion Time (NFR-006)

**Objective:** 8GB NPPES full replacement file ingests within 4 hours.

**Steps:**
1. Trigger NPPES full file ingestion with a production-sized test file (~8GB, ~8M records).
2. Monitor start → completion timestamp.
3. Verify record count matches expected count ± 0.01%.
4. Verify no records lost during ingestion.

**Pass Criteria:** Ingestion completes within 4 hours. Record count accurate. Post-ingestion spot-check: 20 random NPIs query successfully.

---

## 8. Test Cases — Data Freshness (NPPES Delta Updates)

### TC-FRESH-001: Weekly Delta File Processing

**Objective:** Verify NPPES weekly incremental/delta file correctly updates changed records and adds new records without overwriting unchanged records.

**Setup:** Use two consecutive NPPES weekly delta files. Identify the delta (added, changed, deactivated records) between them.

**Steps:**
1. Ingest Delta Week 1. Record field values for 20 NPIs that are modified in Delta Week 2.
2. Ingest Delta Week 2.
3. For the 20 modified NPIs: verify changed fields are updated. Verify unchanged fields are preserved.
4. For any new NPIs in Delta Week 2: verify they are now present in Physician table.
5. For any deactivated NPIs in Delta Week 2: verify their status is updated to "Inactive."

**Pass Criteria:** Changed fields updated. Unchanged fields preserved. New NPIs added. Deactivated NPIs marked inactive. No records lost from previous ingestion.

---

### TC-FRESH-002: Delta Update Idempotency

**Objective:** Verify processing the same delta file twice does not duplicate records or corrupt data.

**Steps:**
1. Ingest Delta Week 1.
2. Ingest Delta Week 1 again (duplicate run).
3. Verify record counts unchanged.
4. Verify no duplicate records in Physician table.
5. Verify no duplicate child records in PhysicianAddress, PhysicianTaxonomy.

**Pass Criteria:** Duplicate ingestion produces no data changes. Record counts identical before and after second run. Zero new duplicates.

---

### TC-FRESH-003: Address Change Triggers Geocode Re-Run

**Objective:** Verify that when a physician's address changes in a delta update, geocoding is triggered automatically.

**Steps:**
1. Capture `geo_lat`, `geo_lng`, `geo_last_updated` for 5 test NPIs.
2. Introduce address changes for those 5 NPIs via delta file.
3. Process delta. Wait for geocoding job.
4. Verify new `geo_lat`, `geo_lng` values reflect the new address.
5. Verify `geo_last_updated` timestamp is newer.

**Pass Criteria:** Geocoding triggered on address change. New coordinates reflect new address. Timestamp updated.

---

### TC-FRESH-004: Unchanged Address Does Not Re-Geocode

**Objective:** Verify geocoding is NOT triggered when address is unchanged (cost control, RISK-003).

**Steps:**
1. Process a delta file where 20 NPIs have non-address field changes only (e.g., name update, enrollment date).
2. Verify `geo_lat`, `geo_lng`, `geo_last_updated` are unchanged for those 20 NPIs.
3. Verify no geocoding API calls are logged for those NPIs.

**Pass Criteria:** No unnecessary geocoding calls. Geocode timestamps unchanged.

---

### TC-FRESH-005: Ingestion Pipeline Schema Validation Alert (RISK-007)

**Objective:** Verify the ingestion pipeline detects unexpected schema changes in CMS source files and raises an alert before committing data.

**Steps:**
1. Create a modified test NPPES CSV with one column renamed and one column removed.
2. Trigger ingestion.
3. Verify: ingestion does NOT commit to production tables.
4. Verify: a schema mismatch alert is generated (email, Slack, or system notification — per DevOps spec).
5. Verify: the previous data remains intact in the database.

**Pass Criteria:** Schema mismatch stops ingestion before commit. Alert generated. Existing data preserved.

---

## 9. Test Cases — Open Payments NPI Match Rate

### TC-OPNPI-001: NPI Match Rate Baseline Measurement

**Objective:** Measure the percentage of Open Payments records that successfully join to a physician NPI (RISK-006). Document as baseline metric.

**Steps:**
1. Count total Open Payments General records in source file.
2. Count records with non-null NPI that matches a Physician.npi in the database.
3. Count records with null NPI.
4. Count records with NPI that does not match any Physician.npi.

**Expected:** Per PRD RISK-006, approximately 3–5% of records will not join cleanly.

**Reporting:**
```
Total Open Payments records: [N]
Successfully joined to NPI: [N] ([X]%)
Unmatched (NPI present but not found): [N] ([X]%)
Null NPI: [N] ([X]%)
Match rate: [X]%
```

**Pass Criteria:** Match rate is documented and reported. If match rate < 95%, escalate — something may be wrong with ingestion. If match rate is 97–100%, this is better than expected — document.

---

### TC-OPNPI-002: Name+State Fuzzy Match Fallback (RISK-006)

**Objective:** Verify the fuzzy name+state fallback matches unmatched Open Payments records to the correct physician.

**Steps:**
1. Identify 20 Open Payments records with blank or invalid NPI.
2. Manually look up the correct physician (using name + state from the Open Payments record).
3. Verify the system matched each record to the same NPI via fuzzy matching.
4. Verify fuzzy-matched records are flagged as "low-confidence match" in the database.

**Pass Criteria:** ≥ 80% of the 20 fuzzy-matchable records correctly matched. All fuzzy matches flagged as low-confidence. Zero high-confidence flags on fuzzy matches.

---

## 10. Pass/Fail Criteria Summary

| Test Area | Pass Threshold | Escalation Trigger |
|---|---|---|
| NPI format validation | 100% invalid NPIs quarantined | Any invalid NPI in Physician table |
| Data accuracy (identity fields) | ≥ 99% field-level accuracy | < 98% accuracy across reference set |
| Data accuracy (Part B/D/Open Payments) | ≥ 99% field-level accuracy | Any numeric field off by > 1% |
| Specialty normalization (200 codes) | ≥ 199/200 correct (99.5%) | < 195/200 correct |
| Geospatial radius accuracy | 100% results within radius | Any result outside radius |
| Radius search response time (50 miles) | p95 ≤ 3,000ms | p95 > 5,000ms |
| Profile load time | p95 ≤ 2,000ms | p95 > 3,000ms |
| All 13 filter types | Each filter returns correct subset | Any filter returns wrong results |
| Combined filter (AND logic) | All conditions satisfied in results | Any condition silently ignored |
| Delta update accuracy | Changed fields updated, unchanged preserved | Any unchanged field overwritten |
| Delta idempotency | Zero duplicates after 2x ingestion | Any duplicate created |
| Open Payments NPI match rate | ≥ 95% match rate | < 90% match rate |
| Full file ingestion time | ≤ 4 hours | > 6 hours |
| 10,000 concurrent user load | p95 ≤ acceptable thresholds, error rate < 0.1% | Error rate > 1% |

---

## 11. Defect Severity Definitions

| Severity | Definition | Examples from This Test Plan |
|---|---|---|
| **Critical** | Data corruption, patient safety, or complete feature failure | Invalid NPIs in DB; radius search returns wrong physicians; data from wrong NPI displayed |
| **High** | Major feature broken, significant data inaccuracy | Filter returns physicians outside criteria; delta update overwrites correct data; match rate < 90% |
| **Medium** | Feature partially broken, workaround exists | Fuzzy match rate < 80%; obsolete taxonomy code crashes UI; freshness timestamp incorrect |
| **Low** | Cosmetic, minor UX | Specialty name capitalization inconsistency; minor address field truncation |

All **Critical** defects → immediately flagged to K2S0. No release until resolved.  
All **High** defects → blocker for feature area sign-off.  
**Medium/Low** → tracked in backlog, don't block release unless they cluster.

---

## 12. Test Environment Requirements

- PostgreSQL database with PostGIS extension installed
- Full production-scale dataset (≥ 4M physician records)
- All source files loaded as specified in Section 2.1
- API server running (Spring Boot application)
- JMeter or k6 installed for performance tests
- Access to ingestion pipeline logs
- Access to quarantine/error log tables
- Admin endpoint for triggering manual ingestion runs
- Network throttling capability for NFR-001 profile load test

---

## 13. Test Execution Order

1. TC-DA-001 (NPI validation) — foundational, run first
2. TC-FRESH-001 through TC-FRESH-005 (delta processing) — must run before spot checks
3. TC-DA-002 through TC-DA-010 (data accuracy) — run after data is confirmed to be loaded
4. TC-SN-001 through TC-SN-005 (specialty normalization)
5. TC-GEO-001 through TC-GEO-007 (geospatial)
6. TC-FILT-001 through TC-FILT-019 (filter + sort)
7. TC-OPNPI-001 through TC-OPNPI-002 (Open Payments match rate)
8. TC-PERF-001 through TC-PERF-005 (performance — run last, requires full data volume and load testing infra)

---

## 14. Exit Criteria

Testing is complete when:
- All test cases executed (or explicitly waived with documented justification)
- Zero open Critical defects
- Zero open High defects in data accuracy or radius search areas
- Open Payments NPI match rate documented
- Performance benchmarks documented for p50, p95, p99 at each load level
- Test results report filed in `/workspace-qa/reports/medsales-physician-data-results.md`

---

*Mac McDonald, QA Engineer*  
*I find the bugs. I destroy the bugs. That's the job.*  
*God willing, none of these tests fail. But they will. They always do.*
