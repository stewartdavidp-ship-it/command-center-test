---
task: "Fix ExploreInChatModal remount flicker — extract to top-level component"
status: complete
cc-spec-id: bugfix-explore-modal-flicker
files:
  - path: "index.html"
    action: modified
commits:
  - sha: "ad34769"
    message: "Fix ExploreInChatModal remount flicker — extract to top-level component (v8.63.5)"
odrc:
  new_decisions:
    - "ExploreInChatModal extracted from IdeasView to a top-level function component — prevents React from recreating the component type on parent re-renders"
    - "Modal now receives configuredApps, apps, globalConcepts, showAlert, and github as explicit props instead of capturing them via closure"
    - "exploreIdea lookup stabilized with React.useMemo keyed on [showExploreModal, globalIdeas] — prevents unnecessary re-computation"
    - "IIFE render pattern replaced with standard conditional render using the memoized exploreIdea value"
  new_opens: []
unexpected_findings:
    - "The flicker was amplified by OrphanDetectionService running in the background — each orphan commit write to Firebase triggered state updates that propagated to IdeasView, causing re-renders that remounted the modal"
    - "The API key was invalid (401), so IdeationBriefGenerator.generate() failed on every mount attempt, fell back to template, set state, and the cycle repeated"
unresolved: []
---

## Approach

Extracted ExploreInChatModal from inside IdeasView (where it was a nested function component) to a top-level function component. This gives React a stable component reference that persists across parent re-renders.

## Implementation Notes

**Root cause chain:**
1. `ExploreInChatModal` defined as nested function inside `IdeasView` → new component type every render
2. Background Firebase writes (OrphanDetectionService, etc.) trigger IdeasView re-renders
3. React sees new component type → unmounts old modal, mounts new one
4. New mount fires `useEffect` → calls `IdeationBriefGenerator.generate()`
5. API call fails (401) → template fallback → state update → parent re-render → goto 2
6. Result: visible flicker loop with repeated API calls

**Fix:**
- Moved ~192 lines of component code from inside IdeasView to a standalone top-level function
- Added 5 explicit props to replace closure captures
- Added `React.useMemo` for `exploreIdea` lookup (was previously computed inside an IIFE in JSX)
- Changed render from IIFE pattern `{showExploreModal && (() => { ... })()}` to `{exploreIdea && (<ExploreInChatModal ... />)}`
