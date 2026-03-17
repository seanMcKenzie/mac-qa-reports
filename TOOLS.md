# QA — Tools & Conventions

## Available Tools
- **Code execution**: Run test suites (pytest, jest, go test, etc.)
- **Git**: Pull branches, check diffs
- **GitHub**: Create bug Issues, comment on PRs
- **Web search / fetch**: Look up testing docs, error messages
- **Agent-to-Agent**: Report results to Admin

## Conventions
- Bug report format:
  ```
  **Title:** [BUG] Short description
  **Severity:** Critical / High / Medium / Low
  **Steps to Reproduce:**
  1. ...
  2. ...
  **Expected:** ...
  **Actual:** ...
  **Environment:** OS, runtime version, branch
  **Evidence:** logs, screenshots, test output
  ```
- Severity levels:
  - **Critical**: App crashes, data loss, security vulnerability
  - **High**: Major feature broken, no workaround
  - **Medium**: Feature partially broken, workaround exists
  - **Low**: Cosmetic, minor UX issue
- Test results format: `✅ 42 passed, ❌ 3 failed, ⏭️ 1 skipped`

### Google Drive
- Credentials: ~/.openclaw/google-credentials.json
- My folder ID: 12LBh2CMt2IZZipFgwUZMLU51uWKamOq-
- Shared folder ID: 1549xZ2SeRSenizm6oqfmItaIGuyN2IOy
- Root project folder: 1Ba8k-urbZljNq6az_KPz1y_zBbYPUfvj
- Use google-auth + google-api-python-client (already installed) to read/write files
- OAuth2 token: ~/.openclaw/google-token.json (use this for read/write — service account is read-only)
- Use google.oauth2.credentials.Credentials + google-api-python-client to access Drive
