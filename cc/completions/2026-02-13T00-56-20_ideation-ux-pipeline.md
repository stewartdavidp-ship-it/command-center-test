---
task: "Ideation Pipeline UX fixes and pipeline completeness — session types, notification banners, Idea name separation"
status: partial
cc-spec-id: sp_ideation_ux_pipeline
files:
  - path: "cc/index.html"
    action: modified
commits:
  - sha: "29237b5"
    message: "Ideation Pipeline UX fixes + session type tracking (v8.62.0) — Session 1"
odrc:
  new_decisions:
    - "Use IIFE pattern for app grid filtering to keep computed variables scoped"
    - "Notification banners render as fixed position stack in top-right corner (top-16 right-4)"
    - "Inline idea creation in import modal uses 1.5s setTimeout to wait for Firebase listener"
  resolved_opens: []
  new_opens:
    - "Should notification banners stack with offset so multiple don't overlap? Currently they flex-column with gap-2"
    - "Should the inline Create Idea in import modal return the new idea ID directly instead of polling by name?"
unexpected_findings:
  - "The RESOLVE format string 'matched to concept_id {id}' appeared in 4 separate locations — updated all to 'resolution explanation' (brief template, assembly fallback)"
  - "Mode 1 app grid was rendering all configuredApps including ones with zero content — some users may have 20+ apps configured"
unresolved:
  - "B1: Phase-aware session types with three system prompts + locked CLAUDE.md template"
  - "B3: Dynamic Explore in Chat button labels based on session type"
  - "B4: Spec and CLAUDE.md detection in inbound flow with three-path routing"
---

## Approach

Split the spec into two sessions as recommended during review. Session 1 covers all 7 UX friction fixes (Category A) plus B2 (session type tracking), totaling 8 items. Session 2 will cover B1, B3, B4 (pipeline completeness).

All changes in a single file (index.html), version bumped 8.61.1 → 8.62.0. 316 lines added, 81 removed.

## Implementation Notes

- **A1 (Explore on Mode 2):** Wrapped each idea chip in a flex container to add the purple icon button alongside. Added `hover:brightness-110` and `title` attribute for discoverability. Truncated idea names at 40 chars in chips.
- **A2 (Name/Description separation):** Replaced two sequential `showPrompt()` calls with a `CreateIdeaModal` component featuring separate Name (input, 80 char soft limit with counter) and Description (textarea) fields. Character counter turns amber at 60, red at 80. Shows "Consider moving details to Description" hint when name exceeds 60 chars.
- **A3 (Brief template fixes):** Updated RESOLVE format in `buildTemplateBrief()` and assembly fallback. Made Supporting Documents section conditional on `sessionCount > 0 || brief.length >= 8000`. Added `odrcState.replace()` to strip generic scope header.
- **A4 (Filter empty apps):** Split `configuredApps` into `appsWithContent` and `emptyApps` using IIFE. Active apps render as full cards, empties collapse into `<details>` element showing count.
- **A5 (Truncate title):** One-line change — `idea.name.substring(0, 50)` with ellipsis.
- **A6 (Notification banners):** Added `notifications` state array + `addNotification`/`dismissNotification` callbacks with `useCallback`. 30-second auto-dismiss via `setTimeout` in `addNotification`. Rendered as fixed stack below header. Replaced both `showConfirm()` calls in polling useEffect.
- **A7 (Auto-create idea):** Added `__create__` option to linked idea dropdown. Inline form pre-fills from metadata slug (kebab→Title Case) and appId. Creates idea via `IdeaManager.create()`, then polls `globalIdeas` after 1.5s to find and link the new idea.
- **B2 (Session type tracking):** Added `type` field to `addSessionLogEntry()` (defaults to 'exploration'). Added `detectSessionType()` function using content pattern matching. Updated `executeODRCImport()` to detect and pass session type from full file content.

All 34 Playwright tests pass.
