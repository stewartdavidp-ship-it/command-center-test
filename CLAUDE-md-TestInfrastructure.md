# CLAUDE.md — CC Test Infrastructure & Playwright Suite
# cc-spec-id: sp_cc_tests
# App: Command Center (index.html)
# Base version: 8.60.0 (or current after Phase 3)
# Target version: same (no app version bump — test infrastructure only)
# Depends on: Phase 1 (sp_ingest_p1), Phase 2 (sp_ingest_p2), Phase 3 (sp_ingest_p3)

---

## Task Summary

Build a Playwright test suite for Command Center with mock service layers so tests run without hitting live GitHub, Firebase, or Anthropic APIs. Establish the test infrastructure in `cc/tests/`, create comprehensive UI/interaction tests, and produce a structured results report. This becomes part of the standard post-task workflow for all future Code sessions.

---

## What to Build

### 1. Mock Service Layer

CC's architecture makes mocking clean — all external access goes through singleton service objects and the GitHubAPI class. Create mock versions that use in-memory state.

**Create `cc/tests/mocks.js`:**

```javascript
// Mock Firebase Database
// Replaces firebaseDb with an in-memory store that supports
// the same .ref().on('value'), .ref().push(), .ref().set(), .ref().update() patterns
const MockFirebaseDb = {
    _store: {},

    ref(path) {
        return {
            _path: path,
            _listeners: [],

            on(event, callback) {
                // Store listener, immediately fire with current data
                this._listeners.push({ event, callback });
                const data = MockFirebaseDb._getPath(path);
                callback({ val: () => data });
                return callback;
            },

            off(event, callback) {
                this._listeners = this._listeners.filter(l => l.callback !== callback);
            },

            once(event) {
                const data = MockFirebaseDb._getPath(path);
                return Promise.resolve({ val: () => data });
            },

            push() {
                const key = 'mock_' + Date.now() + '_' + Math.random().toString(36).substr(2, 5);
                const childPath = `${path}/${key}`;
                return {
                    key,
                    set(value) {
                        MockFirebaseDb._setPath(childPath, value);
                        return Promise.resolve();
                    }
                };
            },

            child(key) {
                return MockFirebaseDb.ref(`${path}/${key}`);
            },

            set(value) {
                MockFirebaseDb._setPath(path, value);
                return Promise.resolve();
            },

            update(value) {
                const current = MockFirebaseDb._getPath(path) || {};
                MockFirebaseDb._setPath(path, { ...current, ...value });
                return Promise.resolve();
            },

            remove() {
                MockFirebaseDb._deletePath(path);
                return Promise.resolve();
            },

            orderByChild(key) { return this; },
            limitToLast(n) { return this; }
        };
    },

    _getPath(path) { /* traverse _store by path segments */ },
    _setPath(path, value) { /* set value at path in _store */ },
    _deletePath(path) { /* delete value at path in _store */ },

    // Reset all data between tests
    _reset() { this._store = {}; },

    // Seed with fixture data
    _seed(data) { this._store = JSON.parse(JSON.stringify(data)); }
};

// Mock Firebase Auth
const MockFirebaseAuth = {
    _currentUser: { uid: 'test-user-123', email: 'test@example.com' },
    _listeners: [],

    onAuthStateChanged(callback) {
        this._listeners.push(callback);
        callback(this._currentUser);
        return () => { this._listeners = this._listeners.filter(l => l !== callback); };
    },

    get currentUser() { return this._currentUser; }
};

// Mock GitHub API
// Returns fixture data for repo operations without network calls
const MockGitHubAPI = {
    _fixtures: {},

    // Load fixtures: { 'repo/path': { content, textContent, sha } }
    loadFixtures(fixtures) { this._fixtures = fixtures; },

    async listRepoContents(repo, path) {
        const prefix = `${repo}/${path}/`;
        return Object.keys(this._fixtures)
            .filter(k => k.startsWith(prefix) && !k.slice(prefix.length).includes('/'))
            .map(k => ({
                name: k.split('/').pop(),
                path: k.replace(`${repo}/`, ''),
                type: 'file',
                sha: this._fixtures[k]?.sha || 'mock-sha'
            }));
    },

    async getFileContent(repo, path) {
        const key = `${repo}/${path}`;
        return this._fixtures[key] || null;
    },

    async getFile(repo, path) {
        const key = `${repo}/${path}`;
        return this._fixtures[key] || null;
    },

    async listRecentCommits(repo, perPage) {
        return this._fixtures[`${repo}/_commits`] || [];
    },

    async getCommitDetail(repo, sha) {
        return this._fixtures[`${repo}/_commit_${sha}`] || null;
    },

    async request(endpoint, options) {
        // Catch-all for unmocked endpoints
        console.warn(`[MockGitHubAPI] Unmocked request: ${endpoint}`);
        return {};
    },

    getRateLimit() {
        return { remaining: 4999, limit: 5000, resetTime: Date.now() + 3600000 };
    },

    _reset() { this._fixtures = {}; }
};

// Mock Claude API Service
const MockClaudeAPIService = {
    _responses: {},
    _callLog: [],

    // Pre-configure responses by matching system prompt keywords
    setResponse(keyword, response) {
        this._responses[keyword] = response;
    },

    async call({ model, system, userMessage, maxTokens }) {
        this._callLog.push({ model, system: system?.substring(0, 100), userMessage: userMessage?.substring(0, 100) });

        // Match by keyword in system prompt
        for (const [keyword, response] of Object.entries(this._responses)) {
            if (system?.includes(keyword) || userMessage?.includes(keyword)) {
                return response;
            }
        }

        // Default response
        return '# Review Prompt\n\nPlease review the attached completion file for correctness and alignment.';
    },

    isConfigured() { return true; },
    getApiKey() { return 'mock-api-key'; },

    _reset() { this._responses = {}; this._callLog = []; }
};

// Mock localStorage (for environments where it may not be available)
const MockLocalStorage = {
    _store: {},
    getItem(key) { return this._store[key] || null; },
    setItem(key, value) { this._store[key] = String(value); },
    removeItem(key) { delete this._store[key]; },
    clear() { this._store = {}; },
    _seed(data) { this._store = { ...data }; }
};
```

### 2. Test Fixtures

**Create `cc/tests/fixtures/`** with seed data files:

**`cc/tests/fixtures/completion-files.js`:**
```javascript
// Sample completion files as they would appear in cc/completions/
const FIXTURE_COMPLETION_FILES = {
    planned: {
        fileName: '2026-02-12T14-30-00_fix-shared-deploy.md',
        content: `---
task: "Fix shared file deployment — cc-shared.css and cc-shared.js were not being copied to satellite repos"
status: complete
files:
  - path: "src/deploy.js"
    action: modified
  - path: "shared/cc-shared.css"
    action: modified
commits:
  - sha: "a1b2c3d"
    message: "Add shared directory to satellite deploy manifest"
odrc:
  resolved_opens:
    - "Shared files not deploying to satellite repos"
  applied_decisions:
    - "CC validates repo state before placing artifacts"
unexpected_findings:
  - "Deploy script had no error handling for missing source files"
unresolved:
  - item: "Deploy script lacks error handling"
    reason: "Separate concern, flagged for follow-up"
---

## Approach
Fixed the deploy manifest to include shared directory.
`,
        specId: 'sp_deploy_fix'
    },

    unplanned: {
        fileName: '2026-02-12T16-00-00_css-cleanup.md',
        content: `---
task: "Clean up stale CSS references across satellite apps"
status: complete
files:
  - path: "shared/cc-shared.css"
    action: modified
commits:
  - sha: "e4f5g6h"
    message: "Remove stale font references from shared CSS"
---

## Approach
Found and removed references to fonts removed in v8.47.
`,
        specId: null
    },

    partial: {
        fileName: '2026-02-12T18-00-00_api-migration.md',
        content: `---
task: "Migrate API calls from v1 to v2 endpoints"
status: partial
files:
  - path: "src/api.js"
    action: modified
commits:
  - sha: "i7j8k9l"
    message: "Migrate auth endpoints to v2"
unresolved:
  - item: "Data endpoints still on v1"
    reason: "Requires schema changes not yet approved"
---

## Approach
Started with auth endpoints as they have no schema dependencies.
`
    },

    invalidYaml: {
        fileName: '2026-02-12T19-00-00_bad-format.md',
        content: `---
task: Missing closing quote
status: complete
---
This file has invalid YAML.
`
    }
};
```

**`cc/tests/fixtures/odrc-concepts.js`:**
```javascript
// Sample ODRC concepts for Firebase seeding
const FIXTURE_CONCEPTS = [
    { id: 'c1', type: 'OPEN', content: 'Shared files not deploying to satellite repos', status: 'active', ideaOrigin: 'idea1', scopeTags: [], specTags: ['sp_deploy_fix'] },
    { id: 'c2', type: 'RULE', content: 'All shared Firebase-backed data lives as top-level state', status: 'active', ideaOrigin: 'idea1', scopeTags: [] },
    { id: 'c3', type: 'DECISION', content: 'CC validates repo state before placing artifacts', status: 'active', ideaOrigin: 'idea1', scopeTags: [] },
    { id: 'c4', type: 'OPEN', content: 'Deploy manifest does not account for nested shared directories', status: 'active', ideaOrigin: 'idea2', scopeTags: [] },
    { id: 'c5', type: 'CONSTRAINT', content: 'Single-file HTML app architecture', status: 'active', ideaOrigin: 'idea1', scopeTags: [] }
];

const FIXTURE_IDEAS = [
    { id: 'idea1', name: 'Deploy Pipeline Fix', description: 'Fix satellite deployment', type: 'base', status: 'active', appId: 'command-center', sequence: 1 },
    { id: 'idea2', name: 'Deploy Improvements', description: 'Enhance deploy capabilities', type: 'addon', status: 'active', appId: 'command-center', sequence: 2 }
];
```

**`cc/tests/fixtures/orphan-commits.js`:**
```javascript
// Sample commits for orphan detection testing
const FIXTURE_COMMITS = [
    {
        sha: 'aaa111',
        commit: { message: 'Fix typo in header', author: { date: '2026-02-10T10:00:00Z' } },
        parents: [{ sha: 'parent1' }],
        files: [{ filename: 'index.html', status: 'modified' }]
    },
    {
        sha: 'bbb222',
        commit: { message: 'Update shared CSS colors', author: { date: '2026-02-11T14:00:00Z' } },
        parents: [{ sha: 'parent2' }],
        files: [{ filename: 'shared/cc-shared.css', status: 'modified' }]
    },
    {
        sha: 'ccc333',
        commit: { message: 'Merge branch feature-x', author: { date: '2026-02-11T15:00:00Z' } },
        parents: [{ sha: 'parent3' }, { sha: 'parent4' }],  // Merge commit — should be filtered
        files: []
    },
    {
        sha: 'ddd444',
        commit: { message: 'Add completion file', author: { date: '2026-02-11T16:00:00Z' } },
        parents: [{ sha: 'parent5' }],
        files: [{ filename: 'cc/completions/test.md', status: 'added' }]  // cc/ only — should be filtered
    }
];
```

### 3. Test Server Setup

**Create `cc/tests/serve.js`:**

A minimal script that serves `index.html` with mocks injected. Uses Node's built-in `http` module (no external dependencies needed since Playwright provides Node):

```javascript
const http = require('http');
const fs = require('fs');
const path = require('path');

const PORT = 3333;
const ROOT = path.resolve(__dirname, '../..');

// Read the original index.html
let html = fs.readFileSync(path.join(ROOT, 'index.html'), 'utf8');

// Read mock and fixture files
const mocks = fs.readFileSync(path.join(__dirname, 'mocks.js'), 'utf8');
const fixtureCompletions = fs.readFileSync(path.join(__dirname, 'fixtures/completion-files.js'), 'utf8');
const fixtureConcepts = fs.readFileSync(path.join(__dirname, 'fixtures/odrc-concepts.js'), 'utf8');
const fixtureOrphans = fs.readFileSync(path.join(__dirname, 'fixtures/orphan-commits.js'), 'utf8');

// Inject mocks before the app code (after the CDN scripts, before <script type="text/babel">)
const mockScript = `
<script>
// === TEST MOCKS ===
window.__CC_TEST_MODE = true;
${mocks}
${fixtureCompletions}
${fixtureConcepts}
${fixtureOrphans}
</script>
`;

// Insert mock script before the babel script block
html = html.replace(
    '<script type="text/babel">',
    `${mockScript}\n<script type="text/babel">`
);

// Replace real Firebase/GitHub initialization with mocks
// The app code checks window.__CC_TEST_MODE and uses mock services if true
// (This requires a small code change in index.html — see Section 5)

const server = http.createServer((req, res) => {
    if (req.url === '/' || req.url === '/index.html') {
        res.writeHead(200, { 'Content-Type': 'text/html' });
        res.end(html);
    } else {
        // Serve static files if needed
        const filePath = path.join(ROOT, req.url);
        if (fs.existsSync(filePath)) {
            res.writeHead(200);
            res.end(fs.readFileSync(filePath));
        } else {
            res.writeHead(404);
            res.end('Not found');
        }
    }
});

server.listen(PORT, () => {
    console.log(`CC Test Server running at http://localhost:${PORT}`);
});
```

### 4. Playwright Configuration

**Create `cc/tests/playwright.config.js`:**

```javascript
const { defineConfig } = require('@playwright/test');

module.exports = defineConfig({
    testDir: './',
    testMatch: '*.spec.js',
    timeout: 30000,
    retries: 1,
    use: {
        baseURL: 'http://localhost:3333',
        headless: true,
        screenshot: 'only-on-failure',
        trace: 'on-first-retry'
    },
    webServer: {
        command: 'node serve.js',
        port: 3333,
        cwd: __dirname,
        reuseExistingServer: true
    },
    reporter: [
        ['list'],
        ['json', { outputFile: 'results/latest.json' }]
    ]
});
```

### 5. Test Mode Hook in index.html

Add a small test mode check near the top of the app code (inside the `<script type="text/babel">` block, after Firebase config but before initialization):

```javascript
// Test mode: swap real services with mocks when running under Playwright
const IS_TEST_MODE = window.__CC_TEST_MODE === true;

if (IS_TEST_MODE) {
    console.log('[CC] Running in TEST MODE — using mock services');
    // Override globals that would normally be set by Firebase/GitHub init
    // firebaseDb = MockFirebaseDb;
    // firebaseAuth = MockFirebaseAuth;
    // The mock objects are injected via the test server
}
```

The actual swapping approach: rather than complex conditional logic throughout the app, inject the mocks as the global objects the app already references. The test server's injected script sets:

```javascript
window.firebaseDb = MockFirebaseDb;
window.firebaseAuth = MockFirebaseAuth;
```

And for GitHub, the app already creates `github = new GitHubAPI(token)` — in test mode, override the constructor or replace after creation:

```javascript
if (window.__CC_TEST_MODE) {
    // Override GitHubAPI to return mock
    window.__MockGitHubAPI = MockGitHubAPI;
}
```

Then in the app's GitHub initialization:
```javascript
const github = IS_TEST_MODE ? window.__MockGitHubAPI : new GitHubAPI(token);
```

This is a minimal code change — one conditional at GitHub init, and the Firebase mocks replace the globals before app code runs.

### 6. Test Suites

**Create `cc/tests/app-load.spec.js`:**
```javascript
const { test, expect } = require('@playwright/test');

test.describe('App Loading', () => {
    test('loads without console errors', async ({ page }) => {
        const errors = [];
        page.on('console', msg => { if (msg.type() === 'error') errors.push(msg.text()); });
        await page.goto('/');
        await page.waitForSelector('[data-testid="app-loaded"]', { timeout: 10000 });
        // Filter out expected warnings
        const realErrors = errors.filter(e => !e.includes('favicon'));
        expect(realErrors).toHaveLength(0);
    });

    test('displays correct version', async ({ page }) => {
        await page.goto('/');
        await page.waitForSelector('text=Command Center');
        const version = await page.textContent('text=/v\\d+\\.\\d+\\.\\d+/');
        expect(version).toBeTruthy();
    });

    test('shows navigation with all sections', async ({ page }) => {
        await page.goto('/');
        await page.waitForSelector('text=Deploy');
        await expect(page.locator('text=Plan')).toBeVisible();
        await expect(page.locator('text=Sessions')).toBeVisible();
        await expect(page.locator('text=Settings')).toBeVisible();
    });
});
```

**Create `cc/tests/job-history.spec.js`:**
```javascript
const { test, expect } = require('@playwright/test');

test.describe('Job History View', () => {
    test.beforeEach(async ({ page }) => {
        await page.goto('/');
        await page.waitForSelector('text=Command Center');
    });

    test('navigates to Job History from Plan dropdown', async ({ page }) => {
        await page.hover('text=Plan');
        await page.click('text=Job History');
        await expect(page.locator('text=Completion files from Claude Code sessions')).toBeVisible();
    });

    test('shows empty state when no jobs exist', async ({ page }) => {
        await page.hover('text=Plan');
        await page.click('text=Job History');
        await expect(page.locator('text=No completion files detected yet')).toBeVisible();
    });

    test('displays job cards when jobs exist', async ({ page }) => {
        // Seed mock Firebase with completion job data
        await page.evaluate(() => {
            MockFirebaseDb._seed({
                'command-center': {
                    'test-user-123': {
                        completionJobs: {
                            'job1': {
                                id: 'job1',
                                repoFullName: 'test/repo',
                                fileName: '2026-02-12T14-30-00_test-task.md',
                                state: 'new',
                                task: 'Test task description',
                                status: 'complete',
                                files: [{ path: 'index.html', action: 'modified' }],
                                commits: [{ sha: 'abc123', message: 'Test commit' }],
                                validationStatus: 'pass',
                                classified: false,
                                detectedAt: new Date().toISOString()
                            }
                        }
                    }
                }
            });
        });
        await page.hover('text=Plan');
        await page.click('text=Job History');
        await expect(page.locator('text=Test task description')).toBeVisible();
    });

    test('filter by state works', async ({ page }) => {
        // Seed with jobs in different states
        await page.evaluate(() => {
            MockFirebaseDb._seed({
                'command-center': {
                    'test-user-123': {
                        completionJobs: {
                            'job1': { id: 'job1', state: 'new', task: 'New job', status: 'complete', repoFullName: 'test/repo', fileName: 'a.md', files: [], commits: [], validationStatus: 'pass', classified: false, detectedAt: new Date().toISOString() },
                            'job2': { id: 'job2', state: 'reviewed', task: 'Reviewed job', status: 'complete', repoFullName: 'test/repo', fileName: 'b.md', files: [], commits: [], validationStatus: 'pass', classified: true, detectedAt: new Date().toISOString() }
                        }
                    }
                }
            });
        });
        await page.hover('text=Plan');
        await page.click('text=Job History');
        // Select "New" filter
        await page.selectOption('select[data-testid="state-filter"]', 'new');
        await expect(page.locator('text=New job')).toBeVisible();
        await expect(page.locator('text=Reviewed job')).not.toBeVisible();
    });

    test('card expands to show details', async ({ page }) => {
        await page.evaluate(() => {
            MockFirebaseDb._seed({
                'command-center': {
                    'test-user-123': {
                        completionJobs: {
                            'job1': {
                                id: 'job1', state: 'new', task: 'Expandable job',
                                status: 'complete', repoFullName: 'test/repo',
                                fileName: 'test.md', validationStatus: 'pass', classified: false,
                                files: [{ path: 'src/app.js', action: 'modified' }],
                                commits: [{ sha: 'abc123', message: 'Fix the thing' }],
                                unexpectedFindings: ['Found a stale reference'],
                                detectedAt: new Date().toISOString()
                            }
                        }
                    }
                }
            });
        });
        await page.hover('text=Plan');
        await page.click('text=Job History');
        // Click to expand
        await page.click('text=Expandable job');
        await expect(page.locator('text=src/app.js')).toBeVisible();
        await expect(page.locator('text=abc123')).toBeVisible();
        await expect(page.locator('text=Found a stale reference')).toBeVisible();
    });

    test('dismiss action changes state to acknowledged', async ({ page }) => {
        await page.evaluate(() => {
            MockFirebaseDb._seed({
                'command-center': {
                    'test-user-123': {
                        completionJobs: {
                            'job1': { id: 'job1', state: 'new', task: 'Dismiss me', status: 'complete', repoFullName: 'test/repo', fileName: 'a.md', files: [], commits: [], validationStatus: 'pass', classified: false, detectedAt: new Date().toISOString() }
                        }
                    }
                }
            });
        });
        await page.hover('text=Plan');
        await page.click('text=Job History');
        await page.click('button:text("Dismiss")');
        // Verify state changed
        await expect(page.locator('text=acknowledged')).toBeVisible();
    });

    test('validation warning indicator shows for invalid files', async ({ page }) => {
        await page.evaluate(() => {
            MockFirebaseDb._seed({
                'command-center': {
                    'test-user-123': {
                        completionJobs: {
                            'job1': { id: 'job1', state: 'new', task: 'Warning job', status: 'complete', repoFullName: 'test/repo', fileName: 'a.md', files: [], commits: [], validationStatus: 'warning', validationErrors: ['Missing commits field'], classified: false, detectedAt: new Date().toISOString() }
                        }
                    }
                }
            });
        });
        await page.hover('text=Plan');
        await page.click('text=Job History');
        await expect(page.locator('text=⚠')).toBeVisible();
    });

    test('unclassified nudge shows when threshold met', async ({ page }) => {
        // Seed 5+ unclassified jobs and settings with threshold 5
        await page.evaluate(() => {
            const jobs = {};
            for (let i = 0; i < 6; i++) {
                jobs[`job${i}`] = { id: `job${i}`, state: 'acknowledged', task: `Unclassified ${i}`, status: 'complete', repoFullName: 'test/repo', fileName: `${i}.md`, files: [], commits: [], validationStatus: 'pass', classified: false, detectedAt: new Date().toISOString() };
            }
            MockFirebaseDb._seed({
                'command-center': {
                    'test-user-123': {
                        completionJobs: jobs,
                        settings: { completionFiles: { unclassifiedNudgeThreshold: 5 } }
                    }
                }
            });
        });
        await page.hover('text=Plan');
        await page.click('text=Job History');
        await expect(page.locator('text=/unclassified jobs/i')).toBeVisible();
    });
});
```

**Create `cc/tests/bundle-assembly.spec.js`:**
```javascript
const { test, expect } = require('@playwright/test');

test.describe('Bundle Assembly (Phase 2)', () => {
    test.beforeEach(async ({ page }) => {
        await page.goto('/');
        await page.waitForSelector('text=Command Center');
        // Seed a reviewed job ready for packaging
        await page.evaluate(() => {
            MockGitHubAPI.loadFixtures({
                'test/repo/cc/completions/test.md': {
                    textContent: '---\ntask: "Test task"\nstatus: complete\nfiles:\n  - path: "index.html"\n    action: modified\ncommits:\n  - sha: "abc"\n    message: "test"\n---\nBody text.'
                },
                'test/repo/CLAUDE.md': {
                    textContent: '# CLAUDE.md\n## Rules\n- Test rule'
                },
                'test/repo/index.html': {
                    textContent: '<html>test content</html>'
                }
            });
            MockClaudeAPIService.setResponse('review', '# Review Prompt\n\nCheck correctness of implementation.');
            MockFirebaseDb._seed({
                'command-center': {
                    'test-user-123': {
                        completionJobs: {
                            'job1': {
                                id: 'job1', state: 'reviewed', task: 'Test task',
                                status: 'complete', repoFullName: 'test/repo',
                                fileName: 'test.md', specId: null,
                                files: [{ path: 'index.html', action: 'modified' }],
                                commits: [{ sha: 'abc', message: 'test' }],
                                validationStatus: 'pass', classified: false,
                                detectedAt: new Date().toISOString()
                            }
                        }
                    }
                }
            });
        });
    });

    test('Package for Check button is enabled for reviewed jobs', async ({ page }) => {
        await page.hover('text=Plan');
        await page.click('text=Job History');
        const btn = page.locator('button:text("Package for Check")');
        await expect(btn).toBeVisible();
        await expect(btn).toBeEnabled();
    });

    test('Bundle assembly modal shows progress steps', async ({ page }) => {
        await page.hover('text=Plan');
        await page.click('text=Job History');
        await page.click('button:text("Package for Check")');
        // Modal should appear with progress
        await expect(page.locator('text=/fetching|generating|building/i')).toBeVisible({ timeout: 5000 });
    });

    test('API key missing shows alert', async ({ page }) => {
        await page.evaluate(() => {
            MockClaudeAPIService.isConfigured = () => false;
            MockClaudeAPIService.getApiKey = () => '';
        });
        await page.hover('text=Plan');
        await page.click('text=Job History');
        await page.click('button:text("Package for Check")');
        await expect(page.locator('text=/API key/i')).toBeVisible({ timeout: 5000 });
    });
});
```

**Create `cc/tests/odrc-ingestion.spec.js`:**
```javascript
const { test, expect } = require('@playwright/test');

test.describe('ODRC Update Ingestion (Phase 3)', () => {
    test('parses structured ODRC update text', async ({ page }) => {
        await page.goto('/');
        const result = await page.evaluate(() => {
            const text = `## ODRC Updates
- RESOLVE OPEN: "Shared files not deploying" → matched to concept_id c1
- NEW OPEN: "Deploy needs nested directory support" → tag to Idea Deploy Improvements
- NEW DECISION: "Use flat shared directory only" → untagged
- NEW RULE: "Always validate deploy manifest before push"`;
            return ODRCUpdateIngestionService.parse(text);
        });
        expect(result).toHaveLength(4);
        expect(result[0].action).toBe('resolve');
        expect(result[0].conceptId).toBe('c1');
        expect(result[1].action).toBe('create');
        expect(result[1].type).toBe('OPEN');
        expect(result[1].targetIdea).toBe('Deploy Improvements');
        expect(result[2].action).toBe('create');
        expect(result[2].type).toBe('DECISION');
        expect(result[3].type).toBe('RULE');
    });

    test('handles format variations gracefully', async ({ page }) => {
        await page.goto('/');
        const result = await page.evaluate(() => {
            const text = `## ODRC Updates
- RESOLVE OPEN: "With quotes" -> matched to concept_id c1
- NEW OPEN: No quotes here → untagged
- NEW DECISION: "Arrow variation" ==> untagged`;
            return ODRCUpdateIngestionService.parse(text);
        });
        // Should parse at least the standard format ones
        expect(result.length).toBeGreaterThan(0);
    });

    test('returns empty array for unparseable input', async ({ page }) => {
        await page.goto('/');
        const result = await page.evaluate(() => {
            return ODRCUpdateIngestionService.parse('This is not ODRC format at all');
        });
        expect(result).toHaveLength(0);
    });
});
```

**Create `cc/tests/orphan-detection.spec.js`:**
```javascript
const { test, expect } = require('@playwright/test');

test.describe('Orphan Commit Detection (Phase 3)', () => {
    test('identifies commits not in any completion file', async ({ page }) => {
        await page.goto('/');
        await page.evaluate(() => {
            // Commits fixture: 4 commits, one is merge (filtered), one is cc-only (filtered)
            MockGitHubAPI.loadFixtures({
                'test/repo/_commits': [
                    { sha: 'aaa111', commit: { message: 'Fix header', author: { date: '2026-02-10T10:00:00Z' } }, parents: [{ sha: 'p1' }] },
                    { sha: 'bbb222', commit: { message: 'Update CSS', author: { date: '2026-02-11T14:00:00Z' } }, parents: [{ sha: 'p2' }] },
                    { sha: 'ccc333', commit: { message: 'Merge feature-x', author: { date: '2026-02-11T15:00:00Z' } }, parents: [{ sha: 'p3' }, { sha: 'p4' }] },
                    { sha: 'ddd444', commit: { message: 'Add completion file', author: { date: '2026-02-11T16:00:00Z' } }, parents: [{ sha: 'p5' }] }
                ],
                'test/repo/_commit_aaa111': { sha: 'aaa111', files: [{ filename: 'index.html', status: 'modified' }] },
                'test/repo/_commit_bbb222': { sha: 'bbb222', files: [{ filename: 'shared/cc-shared.css', status: 'modified' }] },
                'test/repo/_commit_ddd444': { sha: 'ddd444', files: [{ filename: 'cc/completions/test.md', status: 'added' }] }
            });
        });
        // Existing jobs reference sha 'bbb222' — so only 'aaa111' should be orphaned
        // ccc333 filtered (merge), ddd444 filtered (cc/ only)
        const orphans = await page.evaluate(async () => {
            const existingJobs = [{ commits: [{ sha: 'bbb222' }] }];
            const existingOrphans = [];
            const settings = { orphanDetectionDays: 14 };
            return await pollOrphanCommits(MockGitHubAPI, 'test/repo', 'test-user-123', existingJobs, existingOrphans, settings);
        });
        expect(orphans).toHaveLength(1);
        expect(orphans[0].commitSha).toBe('aaa111');
    });

    test('filters merge commits', async ({ page }) => {
        await page.goto('/');
        const result = await page.evaluate(async () => {
            MockGitHubAPI.loadFixtures({
                'test/repo/_commits': [
                    { sha: 'merge1', commit: { message: 'Merge', author: { date: '2026-02-11T10:00:00Z' } }, parents: [{ sha: 'p1' }, { sha: 'p2' }] }
                ]
            });
            return await pollOrphanCommits(MockGitHubAPI, 'test/repo', 'test-user-123', [], [], { orphanDetectionDays: 14 });
        });
        expect(result).toHaveLength(0);
    });
});
```

**Create `cc/tests/detection-dialog.spec.js`:**
```javascript
const { test, expect } = require('@playwright/test');

test.describe('Detection Dialog', () => {
    test('shows dialog when new completion files detected', async ({ page }) => {
        await page.goto('/');
        await page.evaluate(() => {
            MockGitHubAPI.loadFixtures({
                'test/repo/cc/completions/': {},
                'test/repo/cc/completions/2026-02-12T14-30-00_new-task.md': {
                    textContent: '---\ntask: "New task"\nstatus: complete\nfiles:\n  - path: "index.html"\n    action: modified\ncommits:\n  - sha: "abc"\n    message: "test"\n---',
                    name: '2026-02-12T14-30-00_new-task.md'
                }
            });
        });
        // Navigate to dashboard to trigger poll
        await page.click('text=Dashboard');
        // Dialog should appear
        await expect(page.locator('text=/New Completed Work/i')).toBeVisible({ timeout: 10000 });
    });

    test('dismiss sets jobs to acknowledged', async ({ page }) => {
        // Similar setup, then click Dismiss, verify state
    });

    test('review navigates to Job History', async ({ page }) => {
        // Similar setup, then click Review, verify navigation
    });
});
```

**Create `cc/tests/settings.spec.js`:**
```javascript
const { test, expect } = require('@playwright/test');

test.describe('Completion File Settings', () => {
    test('settings view shows completion files section', async ({ page }) => {
        await page.goto('/');
        await page.hover('text=Settings');
        await page.click('text=Settings');
        await expect(page.locator('text=/Completion Files|Unclassified/i')).toBeVisible();
    });

    test('nudge threshold is configurable', async ({ page }) => {
        await page.goto('/');
        await page.hover('text=Settings');
        await page.click('text=Settings');
        // Find and change the threshold input
        const input = page.locator('input[data-testid="nudge-threshold"]');
        if (await input.isVisible()) {
            await input.fill('10');
            // Verify the value is updated
            await expect(input).toHaveValue('10');
        }
    });
});
```

### 7. Test Runner Script

**Create `cc/tests/run.sh`:**
```bash
#!/bin/bash
# Run CC Playwright test suite
# Usage: ./cc/tests/run.sh [--headed] [--filter pattern]

cd "$(dirname "$0")"

# Install Playwright if not present
if ! npx playwright --version > /dev/null 2>&1; then
    echo "Installing Playwright..."
    npm init -y > /dev/null 2>&1
    npm install @playwright/test > /dev/null 2>&1
    npx playwright install chromium > /dev/null 2>&1
fi

# Create results directory
mkdir -p results

# Run tests
if [ "$1" = "--headed" ]; then
    npx playwright test --headed "${@:2}"
elif [ "$1" = "--filter" ]; then
    npx playwright test --grep "$2"
else
    npx playwright test "$@"
fi

# Copy results to standardized location
if [ -f results/latest.json ]; then
    TIMESTAMP=$(date -u +"%Y-%m-%dT%H-%M-%S")
    cp results/latest.json "results/run-${TIMESTAMP}.json"
    echo "Results saved to results/run-${TIMESTAMP}.json"
fi
```

### 8. Test Results Format

The Playwright JSON reporter outputs results that need to be transformed into the CC-expected format for completion file integration.

**Create `cc/tests/format-results.js`:**
```javascript
// Transform Playwright JSON output into CC completion file test format
const fs = require('fs');

const raw = JSON.parse(fs.readFileSync('results/latest.json', 'utf8'));

const formatted = {
    specId: process.argv[2] || 'unknown',
    runAt: new Date().toISOString(),
    framework: 'playwright',
    duration: Math.round((raw.stats?.duration || 0) / 1000),
    summary: {
        total: raw.stats?.expected + raw.stats?.unexpected + raw.stats?.skipped || 0,
        passed: raw.stats?.expected || 0,
        failed: raw.stats?.unexpected || 0,
        skipped: raw.stats?.skipped || 0
    },
    tests: (raw.suites || []).flatMap(suite =>
        (suite.specs || []).map(spec => ({
            name: `${suite.title} > ${spec.title}`,
            status: spec.ok ? 'passed' : 'failed',
            duration: spec.tests?.[0]?.results?.[0]?.duration || 0
        }))
    ),
    errors: (raw.suites || []).flatMap(suite =>
        (suite.specs || [])
            .filter(spec => !spec.ok)
            .map(spec => ({
                test: `${suite.title} > ${spec.title}`,
                message: spec.tests?.[0]?.results?.[0]?.error?.message || 'Unknown error'
            }))
    )
};

const outPath = `results/${formatted.specId}-${formatted.runAt.replace(/[:.]/g, '-')}.json`;
fs.writeFileSync(outPath, JSON.stringify(formatted, null, 2));
console.log(`Formatted results: ${outPath}`);
console.log(`Summary: ${formatted.summary.passed}/${formatted.summary.total} passed, ${formatted.summary.failed} failed, ${formatted.summary.skipped} skipped`);
```

---

## index.html Changes Required

Minimal changes to support test mode:

1. **Add test mode flag check** at top of app code:
```javascript
const IS_TEST_MODE = window.__CC_TEST_MODE === true;
if (IS_TEST_MODE) console.log('[CC] Running in TEST MODE');
```

2. **Conditional GitHub initialization:**
```javascript
const github = IS_TEST_MODE ? window.__MockGitHubAPI : (githubToken ? new GitHubAPI(githubToken) : null);
```

3. **Conditional ClaudeAPIService** (if Phase 2 is built):
```javascript
// In ClaudeAPIService.call(), first line:
if (IS_TEST_MODE && window.MockClaudeAPIService) return window.MockClaudeAPIService.call(arguments[0]);
```

4. **Add data-testid attributes** to key elements for reliable test selectors:
   - `data-testid="app-loaded"` on the main app container
   - `data-testid="state-filter"` on the Job History state filter dropdown
   - `data-testid="repo-filter"` on the Job History repo filter dropdown
   - `data-testid="nudge-threshold"` on the settings threshold input
   - `data-testid="job-card-{id}"` on each job card
   - `data-testid="bundle-progress"` on the bundle assembly modal

These are non-breaking additions — they don't affect the app's behavior or appearance.

---

## Architecture Rules

### State Management Rules
- All shared Firebase-backed data lives as top-level state in App component with `global` prefix
- Firebase listeners are set up once in the App component's auth useEffect
- Views own local UI state only
- Write to Firebase via service methods, let listener update state

### Data Flow Rules
- Data flows down via props, events flow up via callbacks
- Service objects are global singletons
- One listener per collection per user
- All listener useEffect blocks must return a cleanup function

---

## File Structure

All test files go in the `cc/tests/` directory:

```
cc/tests/
  mocks.js                      ← Mock service layer
  fixtures/
    completion-files.js          ← Sample completion file data
    odrc-concepts.js             ← Sample ODRC concepts and ideas
    orphan-commits.js            ← Sample commits for orphan testing
  serve.js                       ← Test server (injects mocks into index.html)
  playwright.config.js           ← Playwright configuration
  run.sh                         ← Test runner script
  format-results.js              ← Results formatter for completion file integration
  app-load.spec.js               ← App loading tests
  job-history.spec.js            ← Job History view tests
  bundle-assembly.spec.js        ← Bundle assembly tests (Phase 2)
  odrc-ingestion.spec.js         ← ODRC update parsing tests (Phase 3)
  orphan-detection.spec.js       ← Orphan commit detection tests (Phase 3)
  detection-dialog.spec.js       ← Detection dialog tests
  settings.spec.js               ← Settings view tests
  results/                       ← Test run outputs (gitignored except latest)
    latest.json
```

---

## Post-Task Obligations

RULE: Before reporting this task as complete, execute this checklist:

1. Commit all code changes to the repo (test files + index.html test mode hook)
2. **Run the test suite** — execute `bash cc/tests/run.sh` and verify results
3. Format test results — execute `node cc/tests/format-results.js sp_cc_tests`
4. Archive this CLAUDE.md to `cc/specs/sp_cc_tests.md`
5. Generate a completion file to `cc/completions/` per the CC Completion File Spec:
   - Include the `tests` frontmatter section with pass/fail counts and report path
   - Required fields: task, status, files, commits
   - Include contextual fields and narrative body
6. Commit the spec archive, completion file, and test results together

**Completion file tests section format:**
```yaml
tests:
  framework: playwright
  passed: {N}
  failed: {N}
  skipped: {N}
  report: cc/tests/results/sp_cc_tests-{timestamp}.json
```

Do not wait for the developer to ask. Run tests and produce the completion file automatically.
