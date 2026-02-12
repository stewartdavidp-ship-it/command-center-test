---
task: "Build Playwright test infrastructure with mock service layers for Command Center"
task_type: feature
status: complete
tracking:
  type: feature
  source_spec: sp_cc_tests
  priority: medium
files:
  - path: "index.html"
    action: modified
  - path: "cc/tests/mocks.js"
    action: created
  - path: "cc/tests/serve.js"
    action: created
  - path: "cc/tests/playwright.config.js"
    action: created
  - path: "cc/tests/run.sh"
    action: created
  - path: "cc/tests/format-results.js"
    action: created
  - path: "cc/tests/fixtures/completion-files.js"
    action: created
  - path: "cc/tests/fixtures/odrc-concepts.js"
    action: created
  - path: "cc/tests/fixtures/orphan-commits.js"
    action: created
  - path: "cc/tests/app-load.spec.js"
    action: created
  - path: "cc/tests/job-history.spec.js"
    action: created
  - path: "cc/tests/bundle-assembly.spec.js"
    action: created
  - path: "cc/tests/odrc-ingestion.spec.js"
    action: created
  - path: "cc/tests/orphan-detection.spec.js"
    action: created
  - path: "cc/tests/detection-dialog.spec.js"
    action: created
  - path: "cc/tests/settings.spec.js"
    action: created
  - path: "cc/specs/sp_cc_tests.md"
    action: created
commits:
  - sha: "1a1f96f"
    message: "Add Playwright test infrastructure with mock service layer (34/34 pass)"
tests:
  framework: playwright
  passed: 34
  failed: 0
  skipped: 0
  report: cc/tests/results/sp_cc_tests-2026-02-12T18-55-08-131Z.json
odrc:
  resolved_opens: []
  applied_decisions:
    - "Mock service layer replaces all external dependencies (Firebase, GitHub API, Claude API, localStorage) with in-memory implementations"
    - "Test server injects mocks before Babel script block, overriding firebase global methods to return mocks"
unexpected_findings:
  - "MockFirebaseDb._seed() needed to notify all active listeners for React state to update mid-test — the original spec did not specify this behavior"
  - "firebase.database() is called directly (not through firebaseDb variable) in OrphanDetectionService — required overriding the firebase global in the test server"
  - "MockGitHubAPI needed 20+ method stubs beyond what the spec listed — the app calls listRepos(), batchCommit(), createOrUpdateFile(), etc. on startup"
  - "Action buttons (Dismiss, Package for Check, Mark as Checked) are inside expanded card detail, requiring card expansion before button interaction"
  - "Navigation dropdowns use CSS group-hover visibility — hover nav button first, then click dropdown item"
unresolved:
  - item: "Detection dialog end-to-end test"
    reason: "The dialog is triggered by pollCompletionFiles during navigation — requires complex fixture setup with GitHub repo contents; deferred to stub tests"
  - item: "Bundle assembly download verification"
    reason: "JSZip blob download can't be verified in headless Playwright without intercepting the download"
---

## Approach

Built the full Playwright test infrastructure per the sp_cc_tests spec:

1. **Mock Service Layer** (`cc/tests/mocks.js`) — Five mock objects replacing all external dependencies:
   - `MockFirebaseDb` — In-memory store with path traversal, listener management, and seed-with-notify for mid-test data injection
   - `MockFirebaseAuth` — Instant auth with test user (uid: test-user-123)
   - `MockGitHubAPI` — Fixture-based responses for all 26 GitHubAPI class methods
   - `MockClaudeAPIService` — Keyword-matched response routing with call logging
   - `MockLocalStorage` — Full Storage API implementation with seed support

2. **Test Server** (`cc/tests/serve.js`) — Node.js HTTP server that reads index.html, injects the mock script block before `<script type="text/babel">`, overrides `firebase.database()` and `firebase.auth()` to return mocks, and pre-seeds localStorage with tokens.

3. **index.html Changes** — Minimal, non-breaking additions:
   - `IS_TEST_MODE` flag at top of app code
   - Conditional Firebase init (skip real init in test mode)
   - Conditional GitHub init (`MockGitHubAPI` in test mode)
   - ClaudeAPIService.call() mock routing in test mode
   - 6 `data-testid` attributes on key elements

4. **7 Test Spec Files** — 34 tests across:
   - App Loading (5): Console errors, version, nav, test mode, auth
   - Job History (8): Navigation, empty state, cards, filters, expand, state change, warning, nudge
   - Bundle Assembly (4): Package button, modal, mark as checked, ODRC import
   - ODRC Ingestion (5): Parse standard format, arrow variations, empty input, resolve/create
   - Orphan Detection (4): Orphan identification, merge filter, cc-only filter, CRUD
   - Detection Dialog (3): Badge count, nav access, listener pattern
   - Settings (5): Completion files section, nudge threshold, API key, status, orphan detection

## Notes

- No app version bump — this is test infrastructure only
- Test results at `cc/tests/results/sp_cc_tests-2026-02-12T18-55-08-131Z.json`
- Run with `bash cc/tests/run.sh` from the repo root
- Tests run in ~41 seconds headless on Chromium
