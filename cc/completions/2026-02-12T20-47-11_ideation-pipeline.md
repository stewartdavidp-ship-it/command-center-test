---
task: "Implement Ideation Workflow: Idea-to-Chat Pipeline — full spec (Phases 1-4)"
task_type: feature
status: complete
tracking:
  type: feature
  source_spec: sp_ideation_pipeline
  priority: high
cc-spec-id: sp_ideation_pipeline
files:
  - path: "index.html"
    action: modified
commits:
  - sha: "9383d39"
    message: "Add Idea-to-Chat Pipeline — Phases 1+2+4p (v8.61.0)"
  - sha: "c94b177"
    message: "Add Phase 3: Inbound ODRC return path with import checklist (v8.61.0)"
tests:
  framework: playwright
  passed: 34
  failed: 0
  skipped: 0
  note: "All existing tests pass — no regressions from ~950 lines of new code"
odrc:
  new_decisions:
    - "IdeationBriefGenerator falls back to template-based brief when ClaudeAPIService is unavailable, ensuring the feature works without API keys"
    - "Session history in Idea Detail is collapsible to avoid cluttering the view for ideas with many sessions"
    - "Phase badge shows computed phase with mismatch indicator when manual override differs from computed value"
    - "Stale indicator (>14 days) only shows when the idea has active OPENs, not for converging/spec-ready ideas"
    - "Explore in Chat button only visible for active ideas (not archived/graduated)"
    - "ODRC detection wired into both ZIP and single-file deploy tab paths, before file action classification"
    - "ODRCImportChecklistModal defined as IIFE-wrapped inner component to get fresh useState on each open"
    - "Duplicate detection uses 50% word overlap on content words (common words excluded) — pragmatic heuristic per spec"
    - "executeODRCImport generates summary via Haiku with fallback to count-based default when API unavailable"
    - "Dual-track file handling: ODRC section goes to checklist, full doc pushes to repo automatically on apply"
  resolved_opens:
    - "Session ID format confirmed as S-YYYY-MM-DD-NNN via generateSessionId() utility function"
    - "Phase 3 inbound return path now fully implemented — deploy tab ODRC detection, editable confirmation checklist, dual-track file handling"
  new_opens:
    - "ExploreInChatModal 'include codebase' checkbox is present but zip packaging of index.html is not yet implemented"
    - "Slug uniqueness check queries all ideas for an app on every create — may need optimization for apps with many ideas"
    - "ODRC content detection relies on line-start patterns ('^- NEW/RESOLVE') — may need tuning for Chat outputs with different formatting"
unexpected_findings:
  - "IdeaManager.create() needed async slug generation (via _uniqueSlug) which queries Firebase for existing ideas — this makes create() slightly slower but ensures uniqueness"
  - "React hooks cannot be used inside IIFEs in JSX — session history expanded state had to be lifted to IdeasView component level"
  - "ODRCImportChecklistModal needed IIFE-wrapped inner component pattern to avoid hooks-in-callback issues while still allowing fresh state on each modal open"
  - "Deploy tab has two distinct file routing paths (ZIP at line ~7329, single at line ~7617) that both needed ODRC detection insertion"
unresolved:
  - item: "Codebase inclusion in zip"
    reason: "The checkbox is present in ExploreInChatModal but actual index.html packaging is deferred — need to handle the 24k+ line file size concern"
---

## Approach

Built the full spec across two commits in one session. Phases 1+2+4p first, then Phase 3:

**Phase 1 (Data Model):** Extended IdeaManager with slug auto-generation (kebab-case, unique within app scope), sessionLog array, lastSessionDate, and phase field. Added `generateSlug()`, `_uniqueSlug()`, and `addSessionLogEntry()` methods. Extended `update()` allowlist to include `slug` and `phase`. Added standalone `computeIdeaPhase()` utility and `generateSessionId()` utility.

**Phase 2 (Outbound Brief):** Built `IdeationBriefGenerator` service object with AI-powered brief generation via ClaudeAPIService (claude-sonnet-4-20250514) plus template fallback. Built `ExploreInChatModal` with brief preview, package contents display, Copy Prompt/Copy Brief/Download Zip/Push to Repo actions. Size-aware output: small briefs get Copy to Clipboard as primary, large briefs get zip download. Zip includes brief + previous session ODRC output when available.

**Phase 3 (Inbound Return Path):** Added ODRC content detection utilities (detectODRCContent, extractODRCMetadata, extractODRCSection) with content-based pattern matching. Wired detection into both ZIP and single-file deploy tab intake paths. Built ODRCImportChecklistModal with: idea picker (grouped by app), editable action/type/description per item, duplicate detection with word-overlap heuristic, auto-uncheck for already-resolved targets, and Apply/Cancel footer. Built executeODRCImport function that writes concepts via ODRCUpdateIngestionService, generates summary via ClaudeAPIService (Haiku), adds session log entry, and pushes full doc to repo (dual-track). Added ODRC import banner in deploy tab with dismiss/review buttons.

**Phase 4 partial (UI):** Added phase badges (exploring=blue, converging=amber, spec-ready=green) to Idea Detail header and Idea History chain in Mode 2. Added session history collapsible section with session ID, date, summary, and concept counts. Added stale session indicator (>14 days with active OPENs). Added slug display in idea detail header.

## Implementation Notes

- `IdeationBriefGenerator` placed in END DATA SERVICE LAYER section alongside other service objects
- ODRC detection functions placed after ODRCUpdateIngestionService (~line 2734)
- `ExploreInChatModal` defined as const arrow function inside IdeasView
- `ODRCImportChecklistModal` uses IIFE-wrapped inner component pattern for clean state management
- Template fallback brief includes session history table, ODRC state, and expected output format
- Phase computation: spec-ready when no OPENs or (≤2 OPENs + ≥10 DECISIONs), converging when decisions ≥ opens with rules/constraints present, exploring otherwise
- ~950 total lines added across two commits
- `pendingOdrcImport` state lives in App component, passed through to DashboardView — follows the same pattern as `pendingSessionReturn`
- findDuplicateConcepts tokenizes descriptions, strips common words, computes overlap percentage against shorter description
