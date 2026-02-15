---
task: "Ingestion Pipeline — session package detection, validation, ODRC import with tangent routing, debrief storage, phase ratchet prompt, context package button"
status: complete
cc-spec-id: sp_session_tab_unit5
files:
  - path: "index.html"
    action: modified
commits:
  - sha: "87a7a03"
    message: "Add ingestion pipeline: session package detection, tangent routing, phase ratchet, context package (v8.69.0)"
odrc:
  new_decisions:
    - "processSessionPackage defined at App scope (near handleFileDrop) where setPendingOdrcImport and showAlert are accessible"
    - "Session package detection (isSessionPackageZip) routes BEFORE deploy routing — highest priority in handleFileDrop zip processing"
    - "Tangent routing is best-effort — unresolved slugs fall back to the primary idea with a console.warn"
    - "Debrief storage is fire-and-forget — failure logged but does not block import"
    - "Phase ratchet prompt is one-directional — only prompts when computed > stored in PHASE_ORDER"
    - "advancing state hoisted to component level to avoid React hooks-in-conditional violation (spec had useState inside if block)"
    - "Context Package button always visible regardless of brief generation state"
  resolved_opens: []
  new_opens:
    - "Should context package include debriefs from previous sessions?"
    - "Should tangent routing show a confirmation before routing items to different ideas?"
    - "Should artifact files from session packages be staged for deployment or just inventoried?"
unexpected_findings:
  - "Spec had useState(false) for advancing state inside conditional if(results) block — this violates React hooks rules. Hoisted to component top level."
  - "Spec referenced completion path as .cc/completions/ (with dot) — used cc/completions/ per repo convention"
  - "Spec referenced file path as cc/index.html — actual path is index.html in repo root"
unresolved: []
---

## Approach

Built all four phases (A-D) sequentially in one session as recommended. Phase A laid the foundation (detection + extraction), Phase B added the UI enhancements to the existing checklist modal, Phase C rewrote executeODRCImport with tangent routing and added debrief storage + phase ratchet, Phase D added the context package button.

## Implementation Notes

- `isSessionPackageZip()` added at line ~2932 after SessionPackageProcessor, checks for session.json at root
- `processSessionPackage()` added at line ~7560 in App scope, extracts session.json + debrief.md, validates, collects artifacts, routes to pendingOdrcImport
- Session package check inserted at line ~7670 in handleFileDrop zip processing, BEFORE isMultiAppZip
- `toIngestionUpdates()` enhanced with `targetIdeaSlug` field for tangent routing
- `extractDebrief()` added to SessionPackageProcessor
- ODRCImportChecklistModal: session package header, tangent badges, debrief preview (collapsible), artifact inventory
- `executeODRCImport()` now separates primary vs tangent items, routes tangents via slug resolution, stores debriefs, returns `tangentCreated` count
- `IdeaManager.storeDebrief()` writes to `ideas/{ideaId}/debriefs/{sessionId}` in Firebase
- `PHASE_ORDER` constant at line ~16113 alongside other phase constants
- Results screen: phase ratchet prompt with Keep/Advance buttons, debrief summary, next session hint, tangent count
- `downloadContextPackage()` in ExploreInChatModal fetches codebase from GitHub, prior specs, ODRC state snapshot, session history
- Total: 390 lines added/modified across 4 phases
