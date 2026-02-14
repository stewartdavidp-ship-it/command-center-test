# ODRC Updates — Session Tab
# Date: 2026-02-14
# Session: S-2026-02-14-010 (Spec Unit 4 — landing page & session card UX)
# Idea: session-tab
# IdeaId: -OlO6yEDlG9FFkjXgAxl
# App: command-center
# Context: Produced spec Unit 4 covering the landing page session card redesign, session lifecycle integration on the home page, Chrome session package detection, staleness safeguards, and session history enrichment in IdeasView.

## ODRC Updates

- NEW DECISION: "The home page is the primary session workflow surface — the full cycle of continue → brief generation → external session → package ingestion is driven from home page session cards, not from the session tab. The session tab (IdeasView) is the detail/history view."
- NEW DECISION: "Session cards on the home page show an active session visual state — green left stripe and border glow replacing the phase color stripe, plus an 'active' badge alongside the phase badge — distinguishing ideas with in-flight work from idle ones."
- NEW DECISION: "The 'Continue' button on the home page card is the trigger for session brief generation. When a session is active, the button changes to 'Upload Results' to guide the user toward completing the cycle rather than accidentally re-dispatching."
- NEW DECISION: "Active session cards sort to the top of the home page work cards section, above pending cards sorted by recency. This ensures in-flight work is always the most visible."
- NEW DECISION: "Staleness indicators on active session cards escalate progressively — no warning under 7 days, amber warning indicator at 7+ days, escalated amber border at 14+ days. These thresholds are starting values to be refined from usage data."
- NEW DECISION: "Session history in IdeasView idea-detail mode renders enriched fields from session.json ingestion — chain metadata (link count, concept blocks, elapsed minutes), debrief summary (expandable), next session recommendation, and phase transitions."

## Session Notes

### What Was Accomplished
Produced the spec document for Unit 4 — Landing Page & Session Card UX. The spec covers six major areas: session card redesign (IdeaWorkCard evolving to dual-state display), session lifecycle integration on the home page (continue, upload results, abandon flows), home page section evolution (active-first sorting, section header), Chrome session package detection (filename pattern matching, ingestion banner), staleness safeguards (progressive visual indicators), and session history enrichment in IdeasView.

The spec maps closely to the existing v8.67.0 infrastructure — IdeaManager session lifecycle methods (activateSession, completeSession, abandonSession), the ExploreInChatModal active session guard, and the SessionPackageProcessor validation pipeline are all already implemented. The spec primarily defines the visual/UX layer that consumes this infrastructure.

### Key Design Principles Established
- The home page is the operational cockpit; the session tab is the archive. Full session lifecycle runs from home page cards.
- Session state and idea phase are visually distinct dimensions on the same card — phase badge (color-coded by maturity) and session state indicator (green active / absent for pending) are independent.
- Active sessions pin to the top of the card list, ensuring in-flight work is never buried below idle ideas.
- Staleness is progressive and advisory, not gating. The user decides when to abandon.

### Artifacts Produced
| # | Filename | Type | Status | Purpose |
|---|----------|------|--------|---------|
| 1 | spec-unit-4-landing-page-session-card-ux.md | spec | complete | Unit 4 spec document |
| 2 | ODRC-Updates-session-tab-S010.md | odrc | complete | This document |

### Session Status
- Concepts: 6 DECISIONs (new), 0 OPENs (new), 0 resolved, 0 RULEs, 0 CONSTRAINTs
- Phase: spec-ready (unchanged)
- Next session: Spec Unit 5 — Ingestion Pipeline. Covers how CC receives, validates, and processes session packages — package detection routing, session record creation, ODRC state update, debrief storage, and the processor rewrite alignment. May need to wait on processor rewrite settling.
