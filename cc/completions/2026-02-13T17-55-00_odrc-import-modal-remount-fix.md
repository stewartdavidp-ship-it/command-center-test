---
task: "Fix ODRC Import modal Apply button: extract IIFE component to stable top-level function to prevent remount"
status: complete
cc-spec-id: bugfix-odrc-import-modal-remount
files:
  - path: "index.html"
    action: modified
commits:
  - sha: "1afe574"
    message: "Fix ODRC Import modal: extract from IIFE to prevent remount, fix Apply button (v8.64.3)"
odrc:
  new_decisions: []
  resolved_opens: []
  new_opens:
    - "Should all remaining IIFE-defined modal components be audited and extracted to top-level?"
unexpected_findings:
  - "Same anti-pattern as ExploreInChatModal (fixed in v8.63.4) — IIFE creates new component type on every parent render, causing React to unmount/remount and reset all internal state"
  - "The remount cycle caused auto-link cascade to re-run on every parent render (visible as x2, x4 in console logs), and the Apply button's onClick handler was lost on each remount"
  - "Session doc pushes (S-005 through S-012) appearing in console were triggered by remount side effects, not by user clicks"
unresolved: []
---

## Approach

Extracted `ODRCImportChecklistModalInner` from the IIFE closure inside DashboardView's render to a top-level `ODRCImportChecklistModal` function component with explicit props. This is the same fix pattern used for `ExploreInChatModal` in v8.63.4.

## Root Cause

The component was defined inside an IIFE `(() => { const Inner = () => { ... }; return <Inner />; })()` which created a **new component type** on every parent render. React's reconciler saw a different component type each time, so it:
1. Unmounted the old instance (destroying all state including `checkedItems`, `linkedIdeaId`, `importing`, etc.)
2. Mounted a new instance (re-initializing all state, re-running the auto-link cascade)
3. The `handleApply` function was bound to the old instance's state and was garbage collected

The Apply button appeared functional (correct text, not disabled) but clicking it either fired on a stale closure or was interrupted by a remount between mousedown and click.

## Implementation Notes

- Extracted to `function ODRCImportChecklistModal({ pendingOdrcImport, setPendingOdrcImport, globalIdeas, globalConcepts, apps, configuredApps, firebaseUid, github, showAlert })`
- Placed between DashboardView and AppsView (line ~12030)
- Added diagnostic logging to `handleApply`: logs blocked reason or start of import
- Removed the temporary `console.log` in checkedItems useState initializer
- All 34 Playwright tests pass
- Version bumped to 8.64.3
