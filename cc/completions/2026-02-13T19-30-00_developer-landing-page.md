---
task: "Replace DashboardView (Home tab) with developer-centric landing page showing work cards and session launcher"
status: complete
cc-spec-id: feature_developer_landing_page
files:
  - path: "index.html"
    action: modified
commits: []
odrc:
  new_decisions: []
  resolved_opens: []
  new_opens:
    - "Should DashboardView legacy deploy state vars be fully pruned from the function signature, or kept for future staged-file quick-deploy from Home?"
    - "Should IdeaWorkCard show more metadata (concept counts, last session summary) or stay minimal?"
unexpected_findings:
  - "Legacy DashboardView contained ~1,840 lines of deploy state vars, effects, useMemo hooks, and JSX for staged files, deploy controls, app cards, pipeline summary, and product metrics — all removed"
  - "Initial approach of wrapping legacy JSX in {false && <React.Fragment>...} caused bracket mismatch due to embedded IIFEs in the JSX — had to fully remove instead of preserving"
  - "Duplicate `configuredApps` declaration (new code + legacy code) caused Babel 'already declared' error — caught by headless browser error capture"
unresolved: []
---

## Approach

Replaced DashboardView's deploy-centric layout with a developer work board. The page now has two zones:
1. **Inbound (top):** Drop box (unchanged) for session artifact intake
2. **Outbound (below):** Idea work cards showing top 5 active ideas by recency, each with phase stripe, app tag, and Continue button

## Implementation

### New components and helpers:
- `getRecentIdeas(globalIdeas, globalConcepts, apps, maxCount)` — enriches active ideas with concept counts, phase, app name, session info
- `IdeaWorkCard({ idea, onNavigateIdea, onNavigateApp, onContinue })` — card with phase-colored left stripe, idea name, app tag, phase badge, session label, date, OPEN count, and Continue button
- `PHASE_COLORS` constant — color definitions for exploring/building/converging/spec-ready phases

### DashboardView changes:
- Added `setViewPayload` prop for cross-view deep-linking
- New state: `continueIdea`, `configuredApps`, `recentIdeas`, `activeIdeaCount`, `activeAppCount`
- New handlers: `handleNavigateIdea` (deep-links to IdeasView), `handleNavigateApp`, `handleContinue` (opens ExploreInChatModal)
- Removed ~1,840 lines of legacy deploy state vars, effects, useMemo hooks, and JSX
- New return: drop box → ODRC import banner → stat chips → "Recent Work" header → IdeaWorkCard list → empty state → ExploreInChatModal → ODRC modal

### Parent wiring:
- Added `viewPayload`/`setViewPayload` state to parent App component
- Passed `setViewPayload` to DashboardView and IdeasView
- IdeasView deep-link effect: consumes `viewPayload.ideaId` to navigate to Idea Detail mode, `viewPayload.createIdea` to open create modal

## Verification

- 34/34 Playwright tests pass
- Version bumped to 8.65.0
- App loads successfully in headless browser
- No Babel compilation errors
