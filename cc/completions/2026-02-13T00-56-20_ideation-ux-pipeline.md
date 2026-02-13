---
task: "Ideation Pipeline UX fixes and pipeline completeness — session types, notification banners, Idea name separation"
status: complete
cc-spec-id: sp_ideation_ux_pipeline
files:
  - path: "cc/index.html"
    action: modified
commits:
  - sha: "29237b5"
    message: "Ideation Pipeline UX fixes + session type tracking (v8.62.0) — Session 1"
  - sha: "5241832"
    message: "Ideation Pipeline: session types, dynamic labels, inbound routing (v8.62.0) — Session 2"
odrc:
  new_decisions:
    - "Use IIFE pattern for app grid filtering to keep computed variables scoped"
    - "Notification banners render as fixed position stack in top-right corner (top-16 right-4)"
    - "Inline idea creation in import modal uses 1.5s setTimeout to wait for Firebase listener"
    - "Session type determined by phase + session log: exploration → spec (when spec-ready) → claude-md (when spec session exists)"
    - "detectInboundArtifactType checks ODRC first, then CLAUDE.md (needs cc-spec-id header), then spec pattern"
  resolved_opens: []
  new_opens:
    - "Should notification banners stack with offset so multiple don't overlap? Currently they flex-column with gap-2"
    - "Should the inline Create Idea in import modal return the new idea ID directly instead of polling by name?"
    - "Should spec artifacts be auto-pushed to a specific repo path (docs/sessions/{slug}/specs/) or let the user choose?"
unexpected_findings:
  - "The RESOLVE format string 'matched to concept_id {id}' appeared in 4 separate locations — updated all to 'resolution explanation'"
  - "Mode 1 app grid was rendering all configuredApps including ones with zero content — some users may have 20+ apps configured"
  - "The locked CLAUDE.md template in the system prompt is substantial (~50 lines) — may need to be externalized if it changes frequently"
unresolved: []
---

## Approach

Split the spec into two sessions as recommended during review.

**Session 1** (A1-A7 + B2): All 7 UX friction fixes plus session type tracking. 316 lines added, 81 removed.

**Session 2** (B1, B3, B4): Pipeline completeness — three-type session system, dynamic button labels, three-path inbound routing. 260 lines added, 81 removed.

Total: ~576 lines added, ~162 removed across both sessions. Version 8.61.1 → 8.62.0. All 34 Playwright tests pass.

## Implementation Notes — Session 1

- **A1 (Explore on Mode 2):** Wrapped each idea chip in a flex container to add the purple icon button alongside. Added `hover:brightness-110` and `title` attribute for discoverability. Truncated idea names at 40 chars in chips.
- **A2 (Name/Description separation):** Replaced two sequential `showPrompt()` calls with a `CreateIdeaModal` component featuring separate Name (input, 80 char soft limit with counter) and Description (textarea) fields. Character counter turns amber at 60, red at 80. Shows "Consider moving details to Description" hint when name exceeds 60 chars.
- **A3 (Brief template fixes):** Updated RESOLVE format in `buildTemplateBrief()` and assembly fallback. Made Supporting Documents section conditional on `sessionCount > 0 || brief.length >= 8000`. Added `odrcState.replace()` to strip generic scope header.
- **A4 (Filter empty apps):** Split `configuredApps` into `appsWithContent` and `emptyApps` using IIFE. Active apps render as full cards, empties collapse into `<details>` element showing count.
- **A5 (Truncate title):** One-line change — `idea.name.substring(0, 50)` with ellipsis.
- **A6 (Notification banners):** Added `notifications` state array + `addNotification`/`dismissNotification` callbacks with `useCallback`. 30-second auto-dismiss via `setTimeout` in `addNotification`. Rendered as fixed stack below header. Replaced both `showConfirm()` calls in polling useEffect.
- **A7 (Auto-create idea):** Added `__create__` option to linked idea dropdown. Inline form pre-fills from metadata slug (kebab→Title Case) and appId. Creates idea via `IdeaManager.create()`, then polls `globalIdeas` after 1.5s to find and link the new idea.
- **B2 (Session type tracking):** Added `type` field to `addSessionLogEntry()` (defaults to 'exploration'). Added `detectSessionType()` function using content pattern matching. Updated `executeODRCImport()` to detect and pass session type from full file content.

## Implementation Notes — Session 2

- **B1 (Phase-aware session types):** Added `getSessionType(idea, concepts)` method that determines session type from phase + session log history. Three-type system: exploration (default), spec (spec-ready but no spec session logged), claude-md (spec-ready and spec session exists). Replaced `getSystemPrompt(phase)` with `getSystemPrompt(sessionType)` using switch/case. The claude-md prompt includes the full locked CLAUDE.md template (~50 lines). Updated `generate()` to compute session type and pass it. Updated `buildTemplateBrief()` to handle all three types with distinct headers and goal sections. Updated `getHandshakePrompt()` to accept and display session type label.
- **B3 (Dynamic button labels):** Mode 3 button uses IIFE to compute session type and display "📋 Generate Spec", "📄 Generate CLAUDE.md in Chat", or "🗣️ Explore in Chat". Mode 2 chips use same pattern for icon and tooltip. ExploreInChatModal title also reflects session type.
- **B4 (Inbound artifact detection):** Added `detectInboundArtifactType()` function — checks ODRC first (uses existing `detectODRCContent`), then CLAUDE.md (requires both `# CLAUDE.md` header and `# cc-spec-id:` marker), then spec pattern. Updated both ZIP and single-file handleFileDrop paths to use three-path routing. Spec/CLAUDE.md artifacts with embedded ODRC sections get dual handling — ODRC checklist modal plus the `artifactType` flag passed through for doc push. Added `addNotification` as DashboardView prop for artifact detection alerts.
