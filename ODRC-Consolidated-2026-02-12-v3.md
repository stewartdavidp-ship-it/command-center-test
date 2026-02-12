# ODRC — CC Workflow Model, Activity Discovery & Validation Pipeline
# Consolidated from: Full working sessions (2026-02-12)
# Updated: 2026-02-12 v3 — Phase 1+2 build findings, test infrastructure, tracking, bug fix pattern
# Origin: CC satellite deploy bug → workflow gaps → completion files → maker/checker → validation pipeline → ingestion spec → builds → testing

---

## DECISIONS (Ratified)

### Workflow Model

1. **Code owns mutations, Chat owns reasoning.** CC bridges them by making Chat's reasoning available to Code (via CLAUDE.md) and Code's outcomes available to Chat (via completion files + validation bundle). The developer can go directly to Code for tactical fixes — CC captures what happened after the fact.

2. **Two workflow speeds.** Full lifecycle (shape → challenge → capture → refine → CLAUDE.md → Code) for planned features. Fast path (developer goes directly to Code, CC discovers after the fact) for bugs and tactical fixes. The line: if it changes product direction (new RULE, new DECISION, resolved OPEN), full lifecycle. If it fixes within current direction, fast path.

3. **CC is the journal of record, not the gatekeeper.** Any workflow that requires CC in the critical path for every change will be abandoned under time pressure. CC must reconstruct context after the fact, not just capture it in real time.

### Repo-Aware Artifact Placement

4. **CC validates repo state before placing Chat artifacts.** CC checks target file(s) against current repo state and blocks placement if Code has modified those files since the artifact was generated. Hard gate — the artifact lands cleanly or CC rejects it and says why.

### Code Completion Files

5. **Code outputs a structured completion file after every task.** This is Code's obligation to CC — the contract that keeps CC current without being in Code's critical path. Completion files replace git-based discovery as the primary sync mechanism. Git discovery is fallback.

6. **Completion file format:** Markdown with YAML frontmatter. Lands in `cc/completions/` repo directory. One file per task. Named `YYYY-MM-DDTHH-MM-SS_task-slug.md`. Committed separately after code commits. Triggered by CLAUDE.md RULE.

7. **ODRC mapping on completion files is optional.** Unplanned work that doesn't map to an existing OPEN is logged as unclassified activity and routed to Chat for classification during validation.

### Maker/Checker Model

8. **Chat is the validation layer.** Code makes, Chat checks. Completion files are review requests, not just records. CC routes Code's output to Chat for validation and records the outcome.

9. **Feedback loop:** Chat → CC → Code (ideation, direction, CLAUDE.md). Code → CC → Chat (completion files, review requests). Chat validates → CC records outcome (confirmed, challenged, escalated).

### Validation Bundle

10. **CC assembles a validation bundle as a zip** containing: completion file, the CLAUDE.md Code was working from, ODRC state summary, a dynamically generated review prompt, and the changed code files. Developer opens a new Chat session, uploads the zip, and Chat performs validation.

11. **CLAUDE.md gets a unique spec tag** (`cc-spec-id: sp_xxxxxxxx`). Completion files reference this tag. CC uses it to match completion files back to their originating spec.

12. **OPENs are tagged with spec-ids at CLAUDE.md generation time.** When CC generates a CLAUDE.md, it tags every active OPEN included in the spec with that spec-id. OPENs accumulate tags across multiple specs — their tag history IS their lifecycle across build cycles. Stored on the concept object as `specTags[]` (parallel to `scopeTags[]`).

13. **Review prompt is dynamically generated via Claude API (Haiku).** Input is completion file metadata + ODRC snapshot. Output is a tailored review prompt. Uses dedicated Anthropic API key (`cc_anthropic_api_key` in localStorage). Two prompt modes: review (planned work, correctness/alignment) and classification (unplanned work, ODRC mapping).

14. **CC surfaces a dialog on new completion file detection** — three options: Dismiss (acknowledge), Review Output (inspect in CC), Package for Chat Check (assemble validation bundle). Acknowledge suppresses future dialogs for that file.

### Ingestion Pipeline

15. **CC polls `cc/completions/` via GitHub API on user-active moments.** No timer, no webhook. Triggers: app open, Dashboard navigation, Job History navigation. One extra API call to list directory contents per configured repo. 60-second cooldown prevents re-polling on rapid navigation.

16. **Completion file lifecycle states: New → Acknowledged → Reviewed → Checked.** Non-linear transitions allowed (New can go straight to Checked). State stored in Firebase at `command-center/{uid}/completionJobs`. Completion file content lives in repo, lifecycle tracking lives in Firebase.

17. **Parsed completion file frontmatter is cached in Firebase on detection.** CC reads the file once from repo, parses YAML, stores structured data alongside lifecycle state. Job History reads from Firebase — no GitHub API calls per view. Repo remains source of truth.

18. **Lightweight schema validation on ingestion.** CC checks required fields exist (task, status, files, commits) and are correct types. Failures flag a warning state in Job History, never reject the file. Developer and Chat can still proceed with flagged files.

19. **Job History is a dedicated view under the Plan nav dropdown.** Shows all completion files with current lifecycle state, context-appropriate actions, filter controls (by state, by repo), and summary stats. Cards show task description, repo, state badge, validation status, classification status, file/commit counts, and detection date.

### Unplanned Work Classification

20. **Unplanned jobs are never forced into classification at detection time.** CC tracks unclassified count and nudges at a configurable threshold (default: 5). Nudge uses the same dialog pattern as other CC notifications. Nudge offers to batch-package unclassified jobs for a single Chat classification session.

21. **Chat classifies unplanned work through two paths:** (a) match to existing ODRC items ("this resolves OPEN 4"), or (b) create new categories if nothing fits (UX fix, performance, integration, bug fix, tech debt, etc.).

### ODRC Updates from Validation

22. **CC passes raw ODRC text from completion files through to the bundle.** Chat proposes matches to existing OPENs during review. No matching logic built into CC. Chat does the reasoning, developer confirms.

23. **Chat outputs structured ODRC recommendations in a defined format.** CC parses the output and presents a confirmation checklist. Developer confirms once, CC executes all Firebase writes (resolve OPENs, create new OPENs, link concepts). Manual creation remains as fallback. Maintains maker/checker — Chat recommends, developer approves, CC executes.

### Validation Bundle Assembly

24. **ODRC state summary format:** Scoped for planned work (tagged OPENs, RULEs, DECISIONs for that spec-id). Full active ODRC landscape for unplanned/classification work. Markdown format, generated dynamically from Firebase concepts at bundle time.

25. **Spec archival in repo.** Code archives the CLAUDE.md it was working from to `cc/specs/{spec-id}.md` as part of the post-task commit. Only applies to planned work with a CC-generated spec-id. Unplanned work has no spec archive — bundle uses current repo root CLAUDE.md for context.

26. **Bundle includes changed code files.** CC pulls all files listed in the completion file's `files` frontmatter from repo via GitHub API. Files placed in a `files/` subdirectory preserving repo paths. Keeps bundle self-contained — Chat has everything, no assumptions.

27. **Configurable bundle size limit (default 5MB).** Under limit: single zip, all files. Over limit: auto-exclude largest files with manifest notes documenting exclusions. Chat knows what's missing. Interactive file selector deferred.

### Code Post-Task Obligations

28. **Pre-completion checklist framing, not post-completion afterthought.** CLAUDE.md RULE instructs Code: before reporting task complete, execute checklist — (1) commit code, (2) run tests if test suite exists, (3) archive CLAUDE.md to `cc/specs/{spec-id}.md` if spec-id present, (4) generate completion file to `cc/completions/`, (5) commit archive + completion file + test results together.

### Orphan Commit Detection

29. **CC checks recent commits against completion file SHAs during existing polls.** One extra API call. Filters: ignore commits only touching `cc/` directory, ignore merge commits, configurable lookback window (default 14 days). Same dialog pattern: Dismiss, Reconstruct via Code, Ignore Permanently.

30. **Reconstruction via Code.** CC packages orphaned commit SHAs and details as a task for Code. Code uses `git show`, file inspection, and CLAUDE.md context to produce proper completion files. Files land in `cc/completions/` and flow through normal ingestion. Reconstructed files are inherently lossy — narrative sections may be incomplete.

### Multi-Phase Spec Packaging

31. **Chat creates spec packages when work exceeds one Code session.** Package contains a manifest (JSON) + ordered CLAUDE.md files. Manifest defines job sequence and dependencies. CC ingests package, creates job queue.

32. **One active Code job per repo at a time.** Prevents merge conflicts from concurrent Code sessions. Job queue is sequential per repo — next job doesn't start until current completes. Validation before starting next job is recommended but not required. Separate repos have independent queues. CC informs developer of dependency status, never blocks.

### Test Infrastructure

33. **`cc/tests/` is the third repo directory.** Repo structure: `cc/completions/` (Code output), `cc/specs/` (archived CLAUDE.md specs), `cc/tests/` (test infrastructure, fixtures, results). Test fixtures and specs live alongside test results.

34. **Code runs tests as part of post-task obligations.** Test execution happens between code commit and completion file generation. Results committed to `cc/tests/results/` as structured JSON. Completion files include an optional `tests` frontmatter section with pass/fail/skip counts and path to results file.

35. **Test results are informational, not a gate.** Consistent with journal-of-record philosophy. CC displays test summary badges on Job History cards. Chat validation bundles include test results. CC does not block packaging or progression based on test failures.

36. **Playwright with mock service layers for CC testing.** MockFirebaseDb, MockGitHubAPI, MockClaudeAPIService replace real services with in-memory equivalents. Test server injects mocks before app code runs. Minimal app changes — one `IS_TEST_MODE` flag check and `data-testid` attributes.

### Task Tracking & Classification

37. **Completion files support optional `task_type` field.** Values: `bug_fix`, `feature`, `refactor`, `chore`, `tech_debt`, `hotfix`. Enables CC to filter, aggregate, and eventually export by type. Unplanned work without task_type gets classified after the fact through Chat.

38. **Completion files support optional `tracking` section.** Contains: `type` (same as task_type), `source_spec` (cc-spec-id), `external_ref` (Jira ticket, GitHub issue, etc.), `priority` (low/medium/high/critical). This is the bridge to external systems.

39. **Bug fix specs are lightweight CLAUDE.md files.** No phased approach, no architecture rules section, no "What NOT to Build." Just: what's broken, why, what to change, where to look, and post-task obligations. Uses `cc-spec-id: bug_{short_id}` naming. Same completion file output requirement — bugs produce completion files like everything else.

### API Key Separation

40. **Anthropic API key is stored separately from CC internal token.** `cc_anthropic_api_key` in localStorage holds the real Anthropic API key (`sk-ant-...`) for ClaudeAPIService. `cc_api_key` remains the CC-generated internal token for deploy queue authentication. Settings view has a dedicated input for the Anthropic key.

### Ideation Platform (Steps 1-4)

41. **ConceptManager and IdeaManager are standalone JS objects** (not React components) placed between GitHub API class and Main App function. They are the data/service layer.

42. **`appIdeas/{appId}` is a denormalized Firebase index.** Maps apps to idea IDs for fast lookup. Source of truth is `ideas/{ideaId}`.

43. **`supersede()`, `resolve()`, and `transition()` are distinct operations** on concepts. Supersede replaces content (same type). Resolve marks an OPEN as done. Transition changes type following the state machine.

44. **IdeasView is its own top-level nav entry** under the Plan dropdown, with three modes (All Concepts, App Aggregate, Idea Detail) and relationship links from Dashboard and Backlog.

45. **Top-level listeners for `globalConcepts` and `globalIdeas`** in App component, following existing pattern for all shared data.

---

## RULES (Ratified)

### Workflow Rules

1. **CC is the journal of record, not the gatekeeper.** (See Decision 3.)

2. **Code must output a completion file after every task.** This is enforced via CLAUDE.md RULE with pre-completion checklist framing. (See Decisions 5, 28.)

3. **One active Code job per repo at a time.** CC enforces sequential execution to prevent merge conflicts. (See Decision 32.)

### State Management Rules (Base — applies to all single-file React/Firebase apps)

4. **All shared Firebase-backed data lives as top-level state** in the App component with `global` prefix. No view component owns shared data.

5. **Firebase listeners are set up once** in the App component's auth useEffect. Views never create their own listeners for shared data.

6. **Views own local UI state only** — filters, modal open/close, form inputs, selected items. Never data another view needs.

7. **Write to Firebase via service methods, let listener update state.** No optimistic UI updates. This prevents local state and Firebase from diverging.

### Data Flow Rules

8. **Data flows down via props, events flow up via callbacks.** No component reaches up or sideways.

9. **Service objects are global singletons** callable from any component. They are the write path to Firebase.

10. **One listener per collection per user.** Never two listeners on the same Firebase path.

11. **Listener callbacks only call the state setter.** No side effects, no cascading writes.

12. **All listener useEffect blocks must return a cleanup function.** No orphaned listeners.

### Concurrency Rules

13. **Serialize by design, not by code.** If two operations could modify the same Firebase path, the UI must prevent concurrent access (e.g., modal open blocks list actions on the same item).

14. **Use Firebase multi-path updates** when multiple writes must be atomic.

---

## OPENs (Active)

### Workflow Model

1. How does CC handle the complexity line between full lifecycle and fast path? Where exactly does "changes product direction" start? Is there a heuristic CC can apply, or is it always a developer judgment call?

2. Should Chat ever produce code changes, or is that a hard boundary? Middle ground: Chat produces code suggestions that only become real when committed through Code?

### Repo-Aware Artifact Placement

3. When CC blocks a Chat artifact placement, what's the recovery path? CC shows the diff and developer reconciles? CC regenerates the request with current state? CC routes to Code for integration? Should CC recommend based on conflict type?

4. What metadata does CC track for placement validation? Commit SHA at artifact generation time? File-level hashes? Something simpler?

### Maker/Checker Model

5. What does Chat's validation process look like in practice? Which checks run every time vs. triaged by complexity? (Correctness, alignment, side effects, ODRC impact, completeness.)

6. When Chat challenges Code's work, what's the resolution path? CC creates a rework task for Code? CC creates a new OPEN? CC flags for developer decision? (Instinct: Option 3 — Chat challenges, developer decides, Code executes.)

7. Does Chat need actual diffs attached to the validation bundle, or is the completion file + code sufficient? Diffs would let Chat verify Code's self-report against reality.

### Validation Bundle

8. What is the exact structured output format Chat should use for ODRC recommendations? Needs a formal schema for reliable CC parsing.

9. How does the developer get Chat's structured output back into CC? Copy-paste? File upload? Direct API integration? (Phase 3 implements copy-paste as first pass.)

10. Bundle size interactive selector — current implementation auto-excludes files over limit with manifest notes. Should there be an interactive file selector before assembly starts?

### Ideation Platform

11. Prop drilling is at its practical limit — DashboardView now has 30+ props and growing with each phase. At what point does CC need React Context or another pattern? What's the trigger for that architectural change?

12. `appIdeas` index consistency — could get out of sync if writes fail partway. Does it need a sync check?

13. Bulk import of ODRC concepts from existing documents — needed for seeding, deferred from Phase 1.

14. Concept conflict detection in aggregate view — how to handle a RULE from Idea 1 contradicting a DECISION from Idea 3.

15. The codebase is now ~23K+ lines and growing. Should satellite extraction be prioritized?

### Rule Inheritance (Future Feature)

16. CC should auto-populate base RULEs and CONSTRAINTs when an app is created, based on stack tags (e.g., `single-file-html + react-cdn + firebase-rtdb`). Three tiers: Universal, Stack-specific, App-specific. Captured in CC-Rule-Inheritance-Model.md. When should this be built?

### Ingestion Pipeline

17. What happens to completion files in the repo over time? Archive strategy? Pruning?

18. How does CC handle completion files from multiple repos simultaneously?

19. GitHub API rate limiting: polling multiple repos on every dashboard visit could hit rate limits for users with many repos. Should we add a cooldown or last-polled timestamp beyond the current 60-second cooldown?

20. Should the detection dialog suppress if user has already dismissed all new jobs in the current session?

### Multi-Phase Spec Packaging

21. When should the multi-phase spec packaging and job queue feature be built? Not needed for current manual phasing but becomes important as workflow matures.

22. Manifest format — JSON or markdown? Current lean: JSON (easier to parse).

23. How does CC ingest the spec package? Developer uploads zip to CC? CC detects it in a repo directory? Chat pushes via some mechanism?

### API & Infrastructure

24. ClaudeAPIService uses `anthropic-dangerous-direct-browser-access` header for browser-based API calls. Is this the long-term pattern or should calls go through Firebase Functions?

25. Review prompt quality: using Haiku for cost efficiency. Should there be a model selector in settings for users who want higher quality prompts?

26. The js-yaml CDN adds a new external dependency. Should we vendor it or keep CDN? CDN is consistent with existing React/Firebase/JSZip pattern.

### Test Infrastructure

27. How should CC handle test result files in `cc/tests/results/` that don't match any completion file? Orphan test results could indicate an incomplete Code session.

28. Should test failure change the urgency of Chat validation? (e.g., auto-suggest "Package for Check" when a completion file reports failed tests)

29. Test trend analytics — pass rate over time, test count growth, flaky test detection. When does this become valuable enough to build?

30. Should CC validate that tests were actually run? (completion file claims 12/12 passed but no results file exists in `cc/tests/results/`)

### Task Tracking

31. Should `task_type` and `tracking` be formally added to the completion file spec? (Currently optional/informal.)

32. How does CC's Job History display and filter by task type?

33. External system integration (Jira, GitHub Issues) — what does export/sync look like?

### Bug Fix Workflow

34. How does CC generate and dispatch bug fix specs? Quick-action from Dashboard? Button in Job History when a validation challenge is recorded? What's the UI flow?

---

## OPENs (Resolved)

*Resolved during ingestion pipeline spec session (2026-02-12):*

| Original # | Question | Resolution |
|------------|----------|------------|
| 3 (old) | Missing completion file fallback — alert on unmatched commits? | → Decision 29: Orphan detection with Dismiss/Reconstruct/Ignore dialog |
| 6 (old) | Schema validation — validate YAML or accept anything? | → Decision 18: Lightweight validation, warnings not rejections |
| 7 (old) | Naming convention — does CC care about filename? | CC cares about contents; naming is for human benefit and sort order |
| 11 (old) | ODRC summary format in zip? | → Decision 24: Scoped for planned, full for classification, markdown |
| 12 (old) | How does CC know which CLAUDE.md to include? | → Decision 25: Code archives to cc/specs/{spec-id}.md |
| 13 (old) | Model/token budget for review prompt? | → Decision 13 (updated): Haiku, uses dedicated Anthropic key |
| 14 (old) | Where does job completion page live? | → Decision 19: Job History, own view under Plan dropdown |
| 15 (old) | Listen mechanism for new completion files? | → Decision 15: Poll on user-active moments via GitHub API |
| 20 (old) | How to match resolved_opens to ODRC? | → Decision 22: Chat proposes, CC passes raw text through |
| 21 (old) | How to route new_opens from completion files? | → Decision 23: Chat recommends structured output, CC presents checklist |

*Resolved during Phase 2 build (2026-02-12):*

| Original # | Question | Resolution |
|------------|----------|------------|
| (P2 new) | What exactly goes into a 'check package'? | → Zip: completion file, spec, code files, ODRC summary, AI review prompt, manifest |
| (P1 new) | listRepoContents/getFileContent method names | → Verified: methods exist on GitHubAPI class at lines 4521 and 4448 |

---

## DECISION Candidates (For Discussion)

1. **Git-based discovery as fallback sync.** CC reads recent commits from a repo, uses AI to summarize what changed, presents to developer for ODRC classification. Only needed when completion files are missing. Requires just GitHub API access CC already has.

2. **CLAUDE.md generation should snapshot ODRC state** — store the full list of tagged OPENs, active RULEs, DECISIONs, and CONSTRAINTs as a frozen record alongside the spec-id. This enables historical comparison: what did Code know vs. what's current.

---

## Configurable Settings (Ingestion Pipeline)

| Setting | Default | Description |
|---------|---------|-------------|
| Unclassified nudge threshold | 5 | Number of unclassified jobs before CC nudges for batch classification |
| Bundle size limit | 5MB | Maximum zip size before auto-exclusion with manifest notes |
| Orphan detection window | 14 days | How far back CC looks for orphaned commits |
| Orphan nudge threshold | 5 | Number of orphaned commits before CC nudges for reconstruction |
| Completion poll cooldown | 60s | Minimum seconds between GitHub API polls for completion files |

---

## Repo Directory Structure

```
cc/
  completions/     ← Code's output (completion files, one per task)
  specs/           ← Archived CLAUDE.md specs (planned work)
  tests/           ← Test infrastructure
    fixtures/      ← Mock data, test scenarios
    results/       ← Test run outputs (JSON reports)
    mocks.js       ← Mock service layer
    *.spec.js      ← Playwright test files
    serve.js       ← Test server
    playwright.config.js
    run.sh
```

---

## Completion File Spec (v0.2)

See separate document: **CC-Completion-File-Spec.md**

Updates from v0.1:
- Directory: `cc/completions/` (not `.cc/completions/`)
- New optional field: `task_type` (bug_fix, feature, refactor, chore, tech_debt, hotfix)
- New optional section: `tracking` (type, source_spec, external_ref, priority)
- New optional section: `tests` (framework, passed, failed, skipped, report path)

---

## Ingestion Pipeline Spec (v0.1)

See separate document: **CC-Ingestion-Pipeline-Spec.md**

---

## Key Artifacts

| Artifact | Purpose |
|----------|---------|
| CC-Completion-File-Spec.md | Full spec for completion file format |
| CC-Ingestion-Pipeline-Spec.md | Full spec for ingestion pipeline |
| CC-Ingestion-Pipeline-Addendum.md | Multi-phase packaging and job queue decisions |
| CLAUDE-md-Phase1-Ingestion.md | Phase 1 spec: detection, Job History, Firebase |
| CLAUDE-md-Phase2-BundleAssembly.md | Phase 2 spec: bundle assembly, API prompts, zip |
| CLAUDE-md-Phase3-OrphanODRC.md | Phase 3 spec: orphan detection, ODRC ingestion, batch classification |
| CLAUDE-md-TestInfrastructure.md | Test infrastructure: Playwright, mocks, fixtures |
| CLAUDE-md-Bug-ApiKeyMismatch.md | Bug fix: ClaudeAPIService API key separation |
| CC-Rule-Inheritance-Model.md | Future: auto-populated rules by app type |
| ODRC-Additions-Workflow-Discovery_1.md | Original workflow gap analysis |

---

## Build Log

| Version | Spec ID | Type | Status | Summary |
|---------|---------|------|--------|---------|
| 8.57.0 | (Steps 3-4) | feature | ✅ Complete | IdeasView, ConceptManager, IdeaManager |
| 8.58.0 | sp_ingest_p1 | feature | ✅ Complete | Detection, Job History, polling, settings |
| 8.59.0 | sp_ingest_p2 | feature | ✅ Complete | Bundle assembly, ClaudeAPIService, review prompts |
| 8.60.0 | sp_ingest_p3 | feature | 🔨 Building | Orphan detection, ODRC ingestion, batch classification |
| — | sp_cc_tests | infrastructure | 📋 Queued | Playwright test suite with mock services |
| — | bug_api_key_mismatch | bug_fix | 📋 Queued | Anthropic API key separation from CC internal token |

---

## Session Log

### Session 1: Workflow Discovery & Ideation Platform

**Session origin:** Testing CC satellite deployment → discovered shared files never deployed → exposed workflow gaps.

**Key progression:** Workflow gaps → unplanned work model → completion files → Code's obligation to CC → Chat as validator (maker/checker) → validation bundle pipeline → spec tagging → OPEN lifecycle tracking → CLAUDE.md generated and pushed → Code built Steps 3-4 (v8.57.0) → first completion file produced → first validation performed.

**What was validated:** Code built IdeasView (672 lines), three modes, three modals, CLAUDE.md generation with completion file section, relationship links. All rules followed. Spec contradiction identified and resolved correctly by Code. Pre-existing bug (doc push routing) found and fixed.

### Session 2: Ingestion Pipeline Spec

**Session origin:** Picking up from Session 1's "what's next" — spec the completion file ingestion pipeline.

**Key progression:** Reviewed completion file spec and ODRC → systematic decision-making on 17 topics → detection, lifecycle, Job History, spec archival, classification, prompts, validation, bundle assembly, ODRC updates, orphan detection, post-task checklist, multi-phase packaging, job queue → generated Phase 1 CLAUDE.md.

**Decisions ratified:** 15 new decisions (Decisions 15-32), resolving 10 previously open items.

### Session 3: Pipeline Build & Test Infrastructure

**Session origin:** Executing the three-phase build and designing test infrastructure.

**Key progression:** Phase 1 built (8.58.0) → Phase 2 built (8.59.0) → Phase 3 dispatched → test infrastructure designed (Playwright + mocks) → test results integration into completion files → discovered API key mismatch bug → designed bug fix spec pattern → designed task_type and tracking fields for completion files → four test result OPENs added → ODRC update from accumulated findings.

**Decisions ratified:** 8 new decisions (Decisions 33-40). New OPENs: 13 added (OPENs 19-34 covering API, testing, tracking, bug workflow). Resolved: 2 OPENs from build findings.

**What's next:** Phase 3 build completes → bug fix for API key → test infrastructure build → first full end-to-end test run → ODRC ingestion from Chat becomes functional.
