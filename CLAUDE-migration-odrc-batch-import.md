# CLAUDE.md — ODRC Batch Import Migration + projectId Field
# cc-spec-id: migration_odrc_batch_import
# App: Command Center (index.html)
# Base version: 8.64.0 (post-backfill build)
# Target version: +0.0.1
# Depends on: sp_idea_backfill_odrc_routing must be deployed (slugs, IdeaId, auto-link cascade)

---

## Task Summary

Two changes in one build:

1. **Data model**: Add `projectId` field to Ideas — a lightweight grouping that persists through graduation. Update `IdeaManager.create()`, `IdeaManager.update()`, and `backfillMissingFields()`.

2. **Migration script**: A temporary console-callable function that creates 11 Ideas under the `command-center` project and imports 125 ODRC concepts mapped to the correct Ideas. Self-destructs after use.

---

## Part 1: Add projectId to Idea Data Model

### P1: Update IdeaManager.create()

**Location:** `IdeaManager.create()` (~line 5491)

Add `projectId` as a parameter with default `null`:

```javascript
// BEFORE:
async create(uid, { name, description, type = 'base', appId = null, parentIdeaId = null }) {

// AFTER:
async create(uid, { name, description, type = 'base', appId = null, parentIdeaId = null, projectId = null }) {
```

Add `projectId` to the idea object being written:

```javascript
const idea = {
    id: ref.key,
    name,
    description,
    slug,
    type,
    appId,
    parentIdeaId,
    projectId,    // ← ADD THIS
    sequence,
    status: 'active',
    sessionLog: [],
    lastSessionDate: null,
    phase: null,
    createdAt: now,
    updatedAt: now
};
```

### P2: Update IdeaManager.update() allowed fields

**Location:** `IdeaManager.update()` (~line 5557)

Add `'projectId'` to the allowed fields array:

```javascript
// BEFORE:
const allowed = ['name', 'description', 'status', 'phase'];

// AFTER:
const allowed = ['name', 'description', 'status', 'phase', 'projectId'];
```

### P3: Update backfillMissingFields()

**Location:** `IdeaManager.backfillMissingFields()` (added by sp_idea_backfill_odrc_routing)

Add a backfill for `projectId`. For existing Ideas that have an `appId`, infer the projectId from the app's config project. For Ideas without an appId, leave null.

Add this block after the existing `parentIdeaId` backfill:

```javascript
// Backfill projectId — infer from appId's project if possible
if (idea.projectId === undefined) {
    // Try to infer from the app's project config
    // Apps in CC config have a .project field (e.g., 'command-center', 'gameshelf')
    patch.projectId = idea.appId || null;
    needsUpdate = true;
}
```

Note: For CC, the appId IS the projectId (`command-center`). For Game Shelf apps, the appId is different from the project (e.g., appId `quotle` but project `gameshelf`). The simple `idea.appId || null` works for now since all existing Ideas are command-center. A smarter inference (looking up app config) can come later.

---

## Part 2: Migration Script

### M1: The Function

Add a temporary function `runODRCBatchImport()` after the IdeaManager object definition. The function:

1. Verifies authentication
2. Creates 11 Ideas (if they don't already exist) with name, description, appId, and projectId
3. Imports concepts mapped to the correct Idea
4. Adds session log entries
5. Logs results

**Place after the IdeaManager closing `};` (~line 5663)**

```javascript
// ============================================================
// TEMPORARY: One-off ODRC batch import migration
// Remove this entire block after successful execution
// ============================================================
async function runODRCBatchImport() {
    const user = firebase.auth().currentUser;
    if (!user) { console.error('[Migration] Not authenticated.'); return; }
    const uid = user.uid;
    console.log('[Migration] Starting ODRC batch import...');

    const CC_APP_ID = 'command-center';
    const CC_PROJECT_ID = 'command-center';

    // ================================================================
    // IDEA DEFINITIONS
    // ================================================================
    const ideaDefs = [
        {
            key: 'workflow-model',
            name: 'CC Workflow Model',
            description: 'Code/Chat separation, two workflow speeds, journal-of-record philosophy, maker/checker model, and feedback loop. The foundational philosophy governing all CC development workflows.'
        },
        {
            key: 'completion-files',
            name: 'Completion Files & Code Contract',
            description: 'Code\'s obligation to CC — structured completion file format, ODRC mapping, post-task obligations. The contract that keeps CC current without being in Code\'s critical path.'
        },
        {
            key: 'ingestion-pipeline',
            name: 'Ingestion Pipeline',
            description: 'How CC discovers and tracks Code\'s output — GitHub polling, completion file lifecycle states, Job History view, Firebase caching, schema validation, unplanned work classification.'
        },
        {
            key: 'validation-bundle',
            name: 'Validation Bundle & Chat Check',
            description: 'Bundle assembly for Chat validation — zip packaging, review prompt generation, spec tags, ODRC state scoping, code file inclusion, and structured ODRC recommendations from Chat.'
        },
        {
            key: 'orphan-detection',
            name: 'Orphan Detection & Reconstruction',
            description: 'Detecting commits without completion files, reconstruction workflows via Code, and auto-matching orphans when completion files are ingested.'
        },
        {
            key: 'spec-packaging',
            name: 'Multi-Phase Spec Packaging',
            description: 'Spec packages for work exceeding one Code session — manifests, job queues, one-active-job-per-repo constraint, dependency tracking.'
        },
        {
            key: 'test-infrastructure',
            name: 'Test Infrastructure',
            description: 'Playwright testing with mock service layers, test result integration into completion files, test-as-informational philosophy.'
        },
        {
            key: 'task-tracking',
            name: 'Task Tracking & Classification',
            description: 'Task type taxonomy, tracking metadata, bug fix spec patterns, external system integration bridge.'
        },
        {
            key: 'ideation-platform',
            name: 'Ideation Platform',
            description: 'ConceptManager, IdeaManager, IdeasView, session brief generation, ODRC import pipeline. The system that manages Ideas and their ODRC concepts.'
        },
        {
            key: 'architecture-rules',
            name: 'CC Architecture Rules',
            description: 'Universal state management, Firebase listener patterns, prop flow, service object patterns, and concurrency rules that apply across all CC development.'
        },
        {
            key: 'stress-test',
            name: 'Stress Test: Idea Validation Challenge',
            description: 'A 5-minute conversational challenge where a persona probes your idea for weaknesses. Five persona lenses, two-phase feedback, constructive pressure to surface gaps via ODRC extraction.'
        }
    ];

    // ================================================================
    // CONCEPT DATA — mapped to idea keys
    // ================================================================
    const concepts = [
        // ---- CC Workflow Model (9) ----
        { idea: 'workflow-model', type: 'DECISION', content: 'Code owns mutations, Chat owns reasoning. CC bridges them by making Chat\'s reasoning available to Code (via CLAUDE.md) and Code\'s outcomes available to Chat (via completion files + validation bundle). The developer can go directly to Code for tactical fixes — CC captures what happened after the fact.' },
        { idea: 'workflow-model', type: 'DECISION', content: 'Two workflow speeds: Full lifecycle (shape → challenge → capture → refine → CLAUDE.md → Code) for planned features. Fast path (developer goes directly to Code, CC discovers after the fact) for bugs and tactical fixes. The line: if it changes product direction (new RULE, new DECISION, resolved OPEN), full lifecycle. If it fixes within current direction, fast path.' },
        { idea: 'workflow-model', type: 'DECISION', content: 'CC is the journal of record, not the gatekeeper. Any workflow that requires CC in the critical path for every change will be abandoned under time pressure. CC must reconstruct context after the fact, not just capture it in real time.' },
        { idea: 'workflow-model', type: 'DECISION', content: 'CC validates repo state before placing Chat artifacts. CC checks target file(s) against current repo state and blocks placement if Code has modified those files since the artifact was generated. Hard gate — the artifact lands cleanly or CC rejects it and says why.' },
        { idea: 'workflow-model', type: 'DECISION', content: 'Chat is the validation layer. Code makes, Chat checks. Completion files are review requests, not just records. CC routes Code\'s output to Chat for validation and records the outcome.' },
        { idea: 'workflow-model', type: 'DECISION', content: 'Feedback loop: Chat → CC → Code (ideation, direction, CLAUDE.md). Code → CC → Chat (completion files, review requests). Chat validates → CC records outcome (confirmed, challenged, escalated).' },
        { idea: 'workflow-model', type: 'RULE', content: 'CC is the journal of record, not the gatekeeper.' },
        { idea: 'workflow-model', type: 'OPEN', content: 'How does CC handle the complexity line between full lifecycle and fast path? Where exactly does "changes product direction" start? Is there a heuristic CC can apply, or is it always a developer judgment call?' },
        { idea: 'workflow-model', type: 'OPEN', content: 'Should Chat ever produce code changes, or is that a hard boundary? Middle ground: Chat produces code suggestions that only become real when committed through Code?' },

        // ---- Completion Files & Code Contract (7) ----
        { idea: 'completion-files', type: 'DECISION', content: 'Code outputs a structured completion file after every task. This is Code\'s obligation to CC — the contract that keeps CC current without being in Code\'s critical path. Completion files replace git-based discovery as the primary sync mechanism. Git discovery is fallback.' },
        { idea: 'completion-files', type: 'DECISION', content: 'Completion file format: Markdown with YAML frontmatter. Lands in cc/completions/ repo directory. One file per task. Named YYYY-MM-DDTHH-MM-SS_task-slug.md. Committed separately after code commits. Triggered by CLAUDE.md RULE.' },
        { idea: 'completion-files', type: 'DECISION', content: 'ODRC mapping on completion files is optional. Unplanned work that doesn\'t map to an existing OPEN is logged as unclassified activity and routed to Chat for classification during validation.' },
        { idea: 'completion-files', type: 'DECISION', content: 'Pre-completion checklist framing, not post-completion afterthought. CLAUDE.md RULE instructs Code: before reporting task complete, execute checklist — (1) commit code, (2) run tests if test suite exists, (3) archive CLAUDE.md, (4) generate completion file, (5) commit archive + completion file + test results together.' },
        { idea: 'completion-files', type: 'DECISION', content: 'Spec archival in repo. Code archives the CLAUDE.md it was working from to cc/specs/{spec-id}.md as part of the post-task commit. Only applies to planned work with a CC-generated spec-id.' },
        { idea: 'completion-files', type: 'RULE', content: 'Code must output a completion file after every task. This is enforced via CLAUDE.md RULE with pre-completion checklist framing.' },
        { idea: 'completion-files', type: 'RULE', content: 'One active Code job per repo at a time. CC enforces sequential execution to prevent merge conflicts.' },

        // ---- Ingestion Pipeline (14) ----
        { idea: 'ingestion-pipeline', type: 'DECISION', content: 'CC polls cc/completions/ via GitHub API on user-active moments. No timer, no webhook. Triggers: app open, Dashboard navigation, Job History navigation. One extra API call to list directory contents per configured repo. 60-second cooldown prevents re-polling on rapid navigation.' },
        { idea: 'ingestion-pipeline', type: 'DECISION', content: 'Completion file lifecycle states: New → Acknowledged → Reviewed → Checked. Non-linear transitions allowed (New can go straight to Checked). State stored in Firebase at command-center/{uid}/completionJobs. Completion file content lives in repo, lifecycle tracking lives in Firebase.' },
        { idea: 'ingestion-pipeline', type: 'DECISION', content: 'Parsed completion file frontmatter is cached in Firebase on detection. CC reads the file once from repo, parses YAML, stores structured data alongside lifecycle state. Job History reads from Firebase — no GitHub API calls per view. Repo remains source of truth.' },
        { idea: 'ingestion-pipeline', type: 'DECISION', content: 'Lightweight schema validation on ingestion. CC checks required fields exist (task, status, files, commits) and are correct types. Failures flag a warning state in Job History, never reject the file. Developer and Chat can still proceed with flagged files.' },
        { idea: 'ingestion-pipeline', type: 'DECISION', content: 'Job History is a dedicated view under the Plan nav dropdown. Shows all completion files with current lifecycle state, context-appropriate actions, filter controls (by state, by repo), and summary stats.' },
        { idea: 'ingestion-pipeline', type: 'DECISION', content: 'CC surfaces a dialog on new completion file detection — three options: Dismiss (acknowledge), Review Output (inspect in CC), Package for Chat Check (assemble validation bundle). Acknowledge suppresses future dialogs for that file.' },
        { idea: 'ingestion-pipeline', type: 'DECISION', content: 'Unplanned jobs are never forced into classification at detection time. CC tracks unclassified count and nudges at a configurable threshold (default: 5). Nudge offers to batch-package unclassified jobs for a single Chat classification session.' },
        { idea: 'ingestion-pipeline', type: 'DECISION', content: 'Chat classifies unplanned work through two paths: (a) match to existing ODRC items, or (b) create new categories if nothing fits (UX fix, performance, integration, bug fix, tech debt, etc.).' },
        { idea: 'ingestion-pipeline', type: 'OPEN', content: 'What happens to completion files in the repo over time? Archive strategy? Pruning?' },
        { idea: 'ingestion-pipeline', type: 'OPEN', content: 'How does CC handle completion files from multiple repos simultaneously?' },
        { idea: 'ingestion-pipeline', type: 'OPEN', content: 'GitHub API rate limiting: polling multiple repos on every dashboard visit could hit rate limits. Should we add a cooldown or last-polled timestamp beyond the current 60-second cooldown?' },
        { idea: 'ingestion-pipeline', type: 'OPEN', content: 'Should the detection dialog suppress if user has already dismissed all new jobs in the current session?' },
        { idea: 'ingestion-pipeline', type: 'OPEN', content: 'ODRC parser error tolerance: Chat output may have diverse formats. Should we add a manual entry fallback for unparseable lines?' },
        { idea: 'ingestion-pipeline', type: 'OPEN', content: 'Batch classification size: no limit on number of jobs in a batch. For large batches, the Claude API call may exceed token limits. Should we add batching within batches?' },

        // ---- Validation Bundle & Chat Check (12) ----
        { idea: 'validation-bundle', type: 'DECISION', content: 'CC assembles a validation bundle as a zip containing: completion file, the CLAUDE.md Code was working from, ODRC state summary, a dynamically generated review prompt, and the changed code files. Developer opens a new Chat session, uploads the zip, and Chat performs validation.' },
        { idea: 'validation-bundle', type: 'DECISION', content: 'CLAUDE.md gets a unique spec tag (cc-spec-id: sp_xxxxxxxx). Completion files reference this tag. CC uses it to match completion files back to their originating spec.' },
        { idea: 'validation-bundle', type: 'DECISION', content: 'OPENs are tagged with spec-ids at CLAUDE.md generation time. When CC generates a CLAUDE.md, it tags every active OPEN included in the spec with that spec-id. OPENs accumulate tags across multiple specs — their tag history IS their lifecycle across build cycles. Stored on the concept object as specTags[] (parallel to scopeTags[]).' },
        { idea: 'validation-bundle', type: 'DECISION', content: 'Review prompt is dynamically generated via Claude API (Haiku). Input is completion file metadata + ODRC snapshot. Output is a tailored review prompt. Uses dedicated Anthropic API key (cc_anthropic_api_key in localStorage). Two prompt modes: review (planned work, correctness/alignment) and classification (unplanned work, ODRC mapping).' },
        { idea: 'validation-bundle', type: 'DECISION', content: 'CC passes raw ODRC text from completion files through to the bundle. Chat proposes matches to existing OPENs during review. No matching logic built into CC. Chat does the reasoning, developer confirms.' },
        { idea: 'validation-bundle', type: 'DECISION', content: 'Chat outputs structured ODRC recommendations in a defined format. CC parses the output and presents a confirmation checklist. Developer confirms once, CC executes all Firebase writes. Manual creation remains as fallback. Maintains maker/checker — Chat recommends, developer approves, CC executes.' },
        { idea: 'validation-bundle', type: 'DECISION', content: 'ODRC state summary format: Scoped for planned work (tagged OPENs, RULEs, DECISIONs for that spec-id). Full active ODRC landscape for unplanned/classification work. Markdown format, generated dynamically from Firebase concepts at bundle time.' },
        { idea: 'validation-bundle', type: 'DECISION', content: 'Bundle includes changed code files. CC pulls all files listed in the completion file\'s files frontmatter from repo via GitHub API. Files placed in a files/ subdirectory preserving repo paths. Keeps bundle self-contained.' },
        { idea: 'validation-bundle', type: 'DECISION', content: 'Configurable bundle size limit (default 5MB). Under limit: single zip, all files. Over limit: auto-exclude largest files with manifest notes documenting exclusions.' },
        { idea: 'validation-bundle', type: 'OPEN', content: 'What does Chat\'s validation process look like in practice? Which checks run every time vs. triaged by complexity?' },
        { idea: 'validation-bundle', type: 'OPEN', content: 'When Chat challenges Code\'s work, what\'s the resolution path? CC creates a rework task for Code? CC creates a new OPEN? CC flags for developer decision?' },
        { idea: 'validation-bundle', type: 'OPEN', content: 'Does Chat need actual diffs attached to the validation bundle, or is the completion file + code sufficient?' },
        { idea: 'validation-bundle', type: 'OPEN', content: 'What is the exact structured output format Chat should use for ODRC recommendations? Needs a formal schema for reliable CC parsing.' },
        { idea: 'validation-bundle', type: 'OPEN', content: 'How does the developer get Chat\'s structured output back into CC? Phase 3 implements copy-paste into the Import modal. File upload? Direct API integration?' },
        { idea: 'validation-bundle', type: 'OPEN', content: 'Bundle size interactive selector — current implementation auto-excludes files over limit with manifest notes. Should there be an interactive file selector before assembly starts?' },
        { idea: 'validation-bundle', type: 'OPEN', content: 'Review prompt quality: using Haiku for cost efficiency. Should there be a model selector in settings for users who want higher quality prompts?' },

        // ---- Orphan Detection & Reconstruction (4) ----
        { idea: 'orphan-detection', type: 'DECISION', content: 'CC checks recent commits against completion file SHAs during existing polls. One extra API call fetching 30 recent commits. Filters: ignore commits only touching cc/ directory, ignore merge commits, configurable lookback window (default 14 days). Same dialog pattern: Dismiss, Reconstruct via Code, Ignore Permanently.' },
        { idea: 'orphan-detection', type: 'DECISION', content: 'Reconstruction via Code. CC generates a markdown task document listing orphaned commit SHAs, dates, messages, and files changed. Code uses git show, file inspection, and CLAUDE.md context to produce proper completion files.' },
        { idea: 'orphan-detection', type: 'DECISION', content: 'Orphan auto-matching on completion file ingestion. When a new completion file is ingested and its commit SHAs match existing orphan records, those orphans are automatically updated to reconstructed state.' },
        { idea: 'orphan-detection', type: 'OPEN', content: 'Orphan commit rate limiting: polling 30 commits + detail fetch per commit could hit GitHub API rate limits. Should we add commit-level caching?' },
        { idea: 'orphan-detection', type: 'OPEN', content: 'Reconstruction task delivery: currently clipboard-only. Should we add repo push to cc/tasks/ so Code can pick it up automatically?' },

        // ---- Multi-Phase Spec Packaging (5) ----
        { idea: 'spec-packaging', type: 'DECISION', content: 'Chat creates spec packages when work exceeds one Code session. Package contains a manifest (JSON) + ordered CLAUDE.md files. Manifest defines job sequence and dependencies. CC ingests package, creates job queue.' },
        { idea: 'spec-packaging', type: 'DECISION', content: 'One active Code job per repo at a time. Prevents merge conflicts from concurrent Code sessions. Job queue is sequential per repo. Separate repos have independent queues. CC informs developer of dependency status, never blocks.' },
        { idea: 'spec-packaging', type: 'OPEN', content: 'When should the multi-phase spec packaging and job queue feature be built?' },
        { idea: 'spec-packaging', type: 'OPEN', content: 'Manifest format — JSON or markdown? Current lean: JSON (easier to parse).' },
        { idea: 'spec-packaging', type: 'OPEN', content: 'How does CC ingest the spec package? Developer uploads zip to CC? CC detects it in a repo directory? Chat pushes via some mechanism?' },

        // ---- Test Infrastructure (8) ----
        { idea: 'test-infrastructure', type: 'DECISION', content: 'cc/tests/ is the third repo directory. Repo structure: cc/completions/ (Code output), cc/specs/ (archived CLAUDE.md specs), cc/tests/ (test infrastructure, fixtures, results).' },
        { idea: 'test-infrastructure', type: 'DECISION', content: 'Code runs tests as part of post-task obligations. Test execution happens between code commit and completion file generation. Results committed to cc/tests/results/ as structured JSON. Completion files include optional tests frontmatter section.' },
        { idea: 'test-infrastructure', type: 'DECISION', content: 'Test results are informational, not a gate. Consistent with journal-of-record philosophy. CC displays test summary badges on Job History cards. Chat validation bundles include test results. CC does not block based on test failures.' },
        { idea: 'test-infrastructure', type: 'DECISION', content: 'Playwright with mock service layers for CC testing. MockFirebaseDb, MockGitHubAPI, MockClaudeAPIService replace real services with in-memory equivalents. Test server injects mocks before app code runs.' },
        { idea: 'test-infrastructure', type: 'OPEN', content: 'How should CC handle test result files in cc/tests/results/ that don\'t match any completion file?' },
        { idea: 'test-infrastructure', type: 'OPEN', content: 'Should test failure change the urgency of Chat validation? Auto-suggest Package for Check when a completion file reports failed tests?' },
        { idea: 'test-infrastructure', type: 'OPEN', content: 'Test trend analytics — pass rate over time, test count growth, flaky test detection. When does this become valuable enough to build?' },
        { idea: 'test-infrastructure', type: 'OPEN', content: 'Should CC validate that tests were actually run? Completion file claims passed but no results file exists.' },

        // ---- Task Tracking & Classification (6) ----
        { idea: 'task-tracking', type: 'DECISION', content: 'Completion files support optional task_type field. Values: bug_fix, feature, refactor, chore, tech_debt, hotfix. Enables CC to filter, aggregate, and export by type.' },
        { idea: 'task-tracking', type: 'DECISION', content: 'Completion files support optional tracking section. Contains: type, source_spec, external_ref, priority. This is the bridge to external systems.' },
        { idea: 'task-tracking', type: 'DECISION', content: 'Bug fix specs are lightweight CLAUDE.md files. No phased approach, no architecture rules section. Just: what\'s broken, why, what to change, where to look, and post-task obligations. Uses cc-spec-id: bug_{short_id} naming.' },
        { idea: 'task-tracking', type: 'OPEN', content: 'Should task_type and tracking be formally added to the completion file spec?' },
        { idea: 'task-tracking', type: 'OPEN', content: 'How does CC\'s Job History display and filter by task type?' },
        { idea: 'task-tracking', type: 'OPEN', content: 'External system integration (Jira, GitHub Issues) — what does export/sync look like?' },
        { idea: 'task-tracking', type: 'OPEN', content: 'How does CC generate and dispatch bug fix specs? Quick-action from Dashboard? Button in Job History?' },

        // ---- Ideation Platform (8) ----
        { idea: 'ideation-platform', type: 'DECISION', content: 'ConceptManager and IdeaManager are standalone JS objects (not React components) placed between GitHub API class and Main App function. They are the data/service layer.' },
        { idea: 'ideation-platform', type: 'DECISION', content: 'appIdeas/{appId} is a denormalized Firebase index. Maps apps to idea IDs for fast lookup. Source of truth is ideas/{ideaId}.' },
        { idea: 'ideation-platform', type: 'DECISION', content: 'supersede(), resolve(), and transition() are distinct operations on concepts. Supersede replaces content (same type). Resolve marks an OPEN as done. Transition changes type following the state machine.' },
        { idea: 'ideation-platform', type: 'DECISION', content: 'IdeasView is its own top-level nav entry under the Plan dropdown, with three modes (All Concepts, App Aggregate, Idea Detail) and relationship links from Dashboard and Backlog.' },
        { idea: 'ideation-platform', type: 'DECISION', content: 'Top-level listeners for globalConcepts and globalIdeas in App component, following existing pattern for all shared data.' },
        { idea: 'ideation-platform', type: 'OPEN', content: 'Prop drilling is at its practical limit — DashboardView now has 30+ props. At what point does CC need React Context or another pattern?' },
        { idea: 'ideation-platform', type: 'OPEN', content: 'appIdeas index consistency — could get out of sync if writes fail partway. Does it need a sync check?' },
        { idea: 'ideation-platform', type: 'OPEN', content: 'Concept conflict detection in aggregate view — how to handle a RULE from Idea 1 contradicting a DECISION from Idea 3.' },

        // ---- CC Architecture Rules (14) ----
        { idea: 'architecture-rules', type: 'DECISION', content: 'Anthropic API key is stored separately from CC internal token. cc_anthropic_api_key in localStorage holds the real Anthropic API key for ClaudeAPIService. cc_api_key remains the CC-generated internal token for deploy queue authentication.' },
        { idea: 'architecture-rules', type: 'RULE', content: 'All shared Firebase-backed data lives as top-level state in the App component with global prefix. No view component owns shared data.' },
        { idea: 'architecture-rules', type: 'RULE', content: 'Firebase listeners are set up once in the App component\'s auth useEffect. Views never create their own listeners for shared data.' },
        { idea: 'architecture-rules', type: 'RULE', content: 'Views own local UI state only — filters, modal open/close, form inputs, selected items. Never data another view needs.' },
        { idea: 'architecture-rules', type: 'RULE', content: 'Write to Firebase via service methods, let listener update state. No optimistic UI updates. This prevents local state and Firebase from diverging.' },
        { idea: 'architecture-rules', type: 'RULE', content: 'Data flows down via props, events flow up via callbacks. No component reaches up or sideways.' },
        { idea: 'architecture-rules', type: 'RULE', content: 'Service objects are global singletons callable from any component. They are the write path to Firebase.' },
        { idea: 'architecture-rules', type: 'RULE', content: 'One listener per collection per user. Never two listeners on the same Firebase path.' },
        { idea: 'architecture-rules', type: 'RULE', content: 'Listener callbacks only call the state setter. No side effects, no cascading writes.' },
        { idea: 'architecture-rules', type: 'RULE', content: 'All listener useEffect blocks must return a cleanup function. No orphaned listeners.' },
        { idea: 'architecture-rules', type: 'RULE', content: 'Serialize by design, not by code. If two operations could modify the same Firebase path, the UI must prevent concurrent access.' },
        { idea: 'architecture-rules', type: 'RULE', content: 'Use Firebase multi-path updates when multiple writes must be atomic.' },
        { idea: 'architecture-rules', type: 'OPEN', content: 'The codebase is now ~24K+ lines and growing. Should satellite extraction be prioritized?' },
        { idea: 'architecture-rules', type: 'OPEN', content: 'ClaudeAPIService uses anthropic-dangerous-direct-browser-access header for browser-based API calls. Is this the long-term pattern or should calls go through Firebase Functions?' },
        { idea: 'architecture-rules', type: 'OPEN', content: 'The js-yaml CDN adds a new external dependency. Should we vendor it or keep CDN?' },
        { idea: 'architecture-rules', type: 'OPEN', content: 'CC should auto-populate base RULEs and CONSTRAINTs when an app is created, based on stack tags. Three tiers: Universal, Stack-specific, App-specific. When should this be built?' },

        // ---- Repo-Aware Artifact Placement OPENs → Workflow Model ----
        { idea: 'workflow-model', type: 'OPEN', content: 'When CC blocks a Chat artifact placement, what\'s the recovery path? CC shows the diff and developer reconciles? CC regenerates the request with current state? CC routes to Code for integration?' },
        { idea: 'workflow-model', type: 'OPEN', content: 'What metadata does CC track for placement validation? Commit SHA at artifact generation time? File-level hashes? Something simpler?' },

        // ---- Stress Test: Idea Validation Challenge (22) ----
        { idea: 'stress-test', type: 'DECISION', content: 'Concept renamed from "Pitch Challenge" to "Stress Test" — the word "pitch" pulls both the user and the AI into sales presentation dynamics, which undermines the real goal of testing whether the idea holds up under scrutiny' },
        { idea: 'stress-test', type: 'DECISION', content: 'Fixed 5-minute session length for all stress tests — provides consistency for evaluation and keeps sessions focused. Eliminates the 2/5/10/15/20 minute selection that added complexity without value' },
        { idea: 'stress-test', type: 'DECISION', content: 'Conversational format, not presentational — the persona opens with a reaction and question after receiving the idea summary. The user isn\'t delivering a monologue, they\'re defending their thinking in a dynamic back-and-forth' },
        { idea: 'stress-test', type: 'DECISION', content: 'Two-phase feedback at session end: Phase 1 is an in-character closing reaction that conveys the persona\'s assessment without a visible score. Phase 2 is an out-of-character ODRC extraction that converts the session into structured updates' },
        { idea: 'stress-test', type: 'DECISION', content: 'No visible rubric or scoring — the evaluation is embedded in the persona\'s in-character response and the ODRC extraction. Users should feel challenged, not tested' },
        { idea: 'stress-test', type: 'DECISION', content: 'Five persona lenses defined by challenge focus, not job title: The Strategist (prioritization/opportunity cost), The Builder (feasibility/efficiency), The User (practical value/friction), The Skeptic (risk/failure modes), The Investor (differentiation/scale at volume)' },
        { idea: 'stress-test', type: 'DECISION', content: 'Persona titles are lens-based not role-based — "The Builder" not "Senior Developer" — avoids defaulting into sales pitch dynamics and keeps focus on what dimension of the idea is being challenged' },
        { idea: 'stress-test', type: 'DECISION', content: 'Persona tone is challenging but constructive — open-minded, not adversarial. The persona wants to believe the idea has merit but needs the user to demonstrate they\'ve done the work' },
        { idea: 'stress-test', type: 'DECISION', content: 'ODRC state from prior sessions is passed into stress test sessions so the persona can track progress on previously identified OPENs — enables continuity between early and late stress tests' },
        { idea: 'stress-test', type: 'DECISION', content: 'Session intake requires three inputs: 2-3 sentence idea summary, target industry, and persona selection. Minimal context by design — the gap between what the AI knows and what the user knows is the playing field' },
        { idea: 'stress-test', type: 'DECISION', content: 'Stress test sessions use standard session ID and idea tags to write back ODRC updates like any other session type — no special handling needed in CC\'s inbound flow' },
        { idea: 'stress-test', type: 'DECISION', content: 'Phase 2 ODRC extraction distinguishes four output types: new OPENs from questions the user couldn\'t answer, validated DECISIONs from points defended with confidence, CONSTRAINTs from hard boundaries the user stated, and RULEs from patterns and guidelines the user articulated' },
        { idea: 'stress-test', type: 'RULE', content: 'Stress test recommended at two lifecycle points: early (session 2-3) to test "is this worth pursuing" and pre-spec to test "is this ready to build" — but never required, surfaced as suggestions by CC' },
        { idea: 'stress-test', type: 'RULE', content: 'Each persona lens should produce a meaningfully different set of ODRC outputs from the same idea — if two personas surface the same issues, the lenses aren\'t differentiated enough' },
        { idea: 'stress-test', type: 'CONSTRAINT', content: 'No role-playing as named real people — personas are archetypes only. Prevents gaming caricatures and keeps the exercise focused on transferable skills' },
        { idea: 'stress-test', type: 'CONSTRAINT', content: 'Maximum five personas to start — depth and true differentiation over breadth' },
        { idea: 'stress-test', type: 'OPEN', content: 'How does industry context get incorporated into the persona prompt — free text field or structured selection? Industry shapes what credible answers sound like' },
        { idea: 'stress-test', type: 'OPEN', content: 'What does the system prompt structure look like to keep Claude in challenge mode without drifting to helpful coaching? Live test showed this is a real risk' },
        { idea: 'stress-test', type: 'OPEN', content: 'How does CC surface stress test suggestions at the right lifecycle moments without mandating them? Needs UX design for the nudge' },
        { idea: 'stress-test', type: 'OPEN', content: 'How does persistent OPEN tracking work in Phase 2 — distinguishing new OPENs vs unresolved OPENs carried forward vs OPENs promoted to DECISIONs since last stress test?' },
        { idea: 'stress-test', type: 'OPEN', content: 'Can persona questions be designed to intentionally surface all four ODRC categories, not just OPENs? Live test showed the persona mostly probed for weaknesses and missed opportunities to draw out constraints and rules' },
        { idea: 'stress-test', type: 'OPEN', content: 'Brief output format needs enrichment — current expected output template is too minimal to produce the quality of session documents needed for CC ingestion' },
    ];

    // ================================================================
    // STEP 1: Create or find Ideas
    // ================================================================
    console.log('[Migration] Creating/finding Ideas...');
    let allIdeas = await IdeaManager.getAll(uid);
    const ideaIdMap = {}; // key → Firebase ID

    for (const def of ideaDefs) {
        // Try to find existing idea by slug or name
        let existing = allIdeas.find(i =>
            i.slug === def.key ||
            IdeaManager.generateSlug(i.name) === def.key ||
            (i.name || '').toLowerCase() === def.name.toLowerCase()
        );

        if (existing) {
            ideaIdMap[def.key] = existing.id;
            console.log(`[Migration] Found existing idea: "${existing.name}" (${existing.id})`);
            // Backfill projectId if missing
            if (!existing.projectId) {
                await IdeaManager._ref(uid).child(existing.id).update({ projectId: CC_PROJECT_ID });
                console.log(`[Migration]   → backfilled projectId: ${CC_PROJECT_ID}`);
            }
        } else {
            const newIdea = await IdeaManager.create(uid, {
                name: def.name,
                description: def.description,
                type: 'base',
                appId: CC_APP_ID,
                projectId: CC_PROJECT_ID
            });
            ideaIdMap[def.key] = newIdea.id;
            console.log(`[Migration] Created idea: "${def.name}" (${newIdea.id}, slug: ${newIdea.slug})`);
        }
    }

    // ================================================================
    // STEP 2: Check for existing concepts
    // ================================================================
    const existingConcepts = await ConceptManager.getAll(uid);
    const existingCount = existingConcepts.filter(c => c.status === 'active').length;
    if (existingCount > 0) {
        console.warn(`[Migration] WARNING: ${existingCount} active concepts already exist. This will ADD, not replace.`);
        console.warn('[Migration] Continuing in 3 seconds... (close console to abort)');
        await new Promise(r => setTimeout(r, 3000));
    }

    // ================================================================
    // STEP 3: Import concepts
    // ================================================================
    console.log(`[Migration] Importing ${concepts.length} concepts across ${ideaDefs.length} ideas...`);
    const results = { created: 0, errors: 0, byIdea: {}, byType: {} };

    for (const item of concepts) {
        const ideaId = ideaIdMap[item.idea];
        if (!ideaId) {
            console.error(`[Migration] No idea found for key: ${item.idea}`);
            results.errors++;
            continue;
        }
        try {
            await ConceptManager.create(uid, {
                type: item.type,
                content: item.content,
                ideaOrigin: ideaId,
                scopeTags: []
            });
            results.created++;
            results.byIdea[item.idea] = (results.byIdea[item.idea] || 0) + 1;
            results.byType[item.type] = (results.byType[item.type] || 0) + 1;
        } catch (e) {
            results.errors++;
            console.error(`[Migration] Error: ${item.type} on ${item.idea}:`, e.message);
        }
    }

    // ================================================================
    // STEP 4: Add session log entries
    // ================================================================
    const now = new Date().toISOString();
    for (const def of ideaDefs) {
        const count = results.byIdea[def.key] || 0;
        if (count === 0) continue;
        try {
            await IdeaManager.addSessionLogEntry(uid, ideaIdMap[def.key], {
                sessionId: `MIGRATION-2026-02-13-${def.key}`,
                date: now,
                docPath: null,
                summary: `Batch import: ${count} concepts from consolidated ODRC sessions (2026-02-12)`,
                conceptsCreated: count,
                conceptsResolved: 0,
                type: 'migration'
            });
        } catch (e) {
            console.error(`[Migration] Session log error for ${def.key}:`, e.message);
        }
    }

    // ================================================================
    // STEP 5: Summary
    // ================================================================
    console.log('');
    console.log('========================================');
    console.log('[Migration] ODRC Batch Import Complete');
    console.log('========================================');
    console.log(`Total: ${results.created} created, ${results.errors} errors`);
    console.log('By type:', JSON.stringify(results.byType));
    console.log('By idea:');
    for (const def of ideaDefs) {
        const count = results.byIdea[def.key] || 0;
        if (count > 0) console.log(`  ${def.name}: ${count}`);
    }
    console.log('========================================');
    console.log('✅ Done. Verify in IdeasView, then REMOVE this migration function.');
}

window.runODRCBatchImport = runODRCBatchImport;
// ============================================================
// END TEMPORARY MIGRATION
// ============================================================
```

---

## Architecture Rules

### What This Does
- Adds `projectId` field to Idea data model (create, update, backfill)
- Creates 11 Ideas via `IdeaManager.create()` with full data model fields
- Creates concepts via `ConceptManager.create()` — same write path as UI
- Adds session log entries recording the migration
- All writes use existing service methods — no direct Firebase manipulation

### What NOT To Do
- Do NOT make the migration script permanent — it's one-time, remove after use
- Do NOT add a button or UI trigger — console-only
- Do NOT modify ConceptManager or IdeaManager beyond adding projectId
- Do NOT run twice without checking for duplicates

---

## Testing

### Test 1: projectId on New Ideas
1. After deploy, create a new Idea through CC UI
2. Check Firebase — verify `projectId` field exists (will be null unless appId is set)
3. Run `runODRCBatchImport()` — verify created Ideas have `projectId: 'command-center'`

### Test 2: Migration Execution
1. Open console, run `runODRCBatchImport()`
2. Verify 11 Ideas created (or found if existing)
3. Verify concept counts match:
   - Workflow Model: 9, Completion Files: 7, Ingestion Pipeline: 14
   - Validation Bundle: 16, Orphan Detection: 5, Spec Packaging: 5
   - Test Infrastructure: 8, Task Tracking: 7, Ideation Platform: 8
   - Architecture Rules: 16, Stress Test: 22
   - **Total: 117** (some cross-cutting OPENs shift the per-idea counts vs original 125)
4. Verify session log entries on each Idea

### Test 3: Verify in IdeasView
1. Navigate to IdeasView
2. Verify all 11 Ideas appear under Command Center app
3. Open each Idea — verify concepts by type match expected counts
4. Verify phase computation reflects ODRC ratios

### Test 4: Backfill projectId
1. Reload CC after deploy
2. Check Firebase — existing Ideas should have `projectId` backfilled
3. Console should show backfill ran

---

## Post-Task Obligations

RULE: Before reporting this task as complete:

1. Verify `projectId` field added to create, update, and backfill
2. Verify migration function runs without errors
3. Verify all 11 Ideas created with correct names, descriptions, and projectId
4. Verify concepts appear in IdeasView mapped to correct Ideas
5. **Do NOT remove migration function yet** — developer will run, verify, then request cleanup
6. Commit all code changes
7. Archive this CLAUDE.md to `cc/specs/migration_odrc_batch_import.md`
8. Generate completion file to `.cc/completions/`
9. Commit spec archive and completion file separately

**Completion file naming:** `YYYY-MM-DDTHH-MM-SS_odrc-batch-import-migration.md`

**Completion file format:**

```yaml
---
task: "Add projectId to Idea data model + one-off migration to create 11 Ideas and import 125 ODRC concepts from consolidated session documents"
status: complete | partial
cc-spec-id: migration_odrc_batch_import
files:
  - path: "cc/index.html"
    action: modified
commits:
  - sha: "{sha}"
    message: "{message}"
odrc:
  new_decisions:
    - "projectId added to Idea schema — lightweight grouping that persists through graduation"
    - "11 Ideas established under command-center project covering workflow model through stress test"
  resolved_opens:
    - "Bulk import of ODRC concepts from existing documents — needed for seeding, deferred from Phase 1"
  new_opens:
    - "How does projectId relate to the existing config.projects structure? Should they merge?"
    - "Project as first-class object — UI for creating, viewing, and managing Projects"
unexpected_findings:
  - "{anything unexpected}"
unresolved:
  - "{anything not completed}"
---

## Approach

{Brief narrative}

## Implementation Notes

{Key details}
```

Do not wait for the developer to ask. Generate the completion file automatically after committing code.
