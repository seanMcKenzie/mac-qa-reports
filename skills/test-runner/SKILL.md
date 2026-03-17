# Skill: Test Runner

## Purpose
Execute test suites, parse results, and generate structured pass/fail reports.

## Triggers
- Admin delegates testing for a PR or branch
- Regression testing requested after a bug fix
- Pre-deployment verification requested

## Procedure

### 1. Environment Setup
1. Pull the target branch
2. Install dependencies (`npm install`, `pip install -r requirements.txt`, etc.)
3. Verify the test environment is functional (database, services, etc.)
4. If environment issues arise, escalate to Admin for DevOps assistance

### 2. Test Execution
1. Identify the project's test framework (jest, pytest, go test, etc.)
2. Run the full test suite with verbose output
3. Capture stdout, stderr, and exit code
4. If specific tests are requested, run only those

### 3. Result Parsing
1. Parse test output for:
   - Total tests run
   - Passed / Failed / Skipped / Errored counts
   - Names of failing tests
   - Error messages and stack traces
   - Test duration
2. Identify flaky tests (tests that fail intermittently) — rerun failures once to confirm

### 4. Report Generation
```
## Test Report — <branch/PR>

**Date:** <timestamp>
**Framework:** <name>
**Result:** ✅ ALL PASSED / ❌ FAILURES DETECTED

### Summary
- Total: <n>
- ✅ Passed: <n>
- ❌ Failed: <n>
- ⏭️ Skipped: <n>
- ⏱️ Duration: <time>

### Failures (if any)
| # | Test Name | Error | File:Line |
|---|-----------|-------|-----------|

### Coverage (if available)
- Statements: <n>%
- Branches: <n>%
- Functions: <n>%
- Lines: <n>%

### Verdict
<PASS: Ready for deployment / FAIL: Issues must be resolved>
```

### 5. Bug Filing
- For each confirmed failure, check if a GitHub Issue already exists
- If not, create one using the bug report template from TOOLS.md
- Link the Issue to the PR being tested

### 6. Reporting
- Send the test report to Admin
- If all pass: recommend proceeding to deployment
- If failures: list blocking Issues and recommend Developer review
