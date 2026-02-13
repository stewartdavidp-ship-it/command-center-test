# ODRC Updates — Landing page to track what work is in progress and be launch pad for EtE flow
# Date: 2026-02-13
# Session: S-2026-02-13-013 (Exploration: fleshed out developer landing page concept, card model, product philosophy, and page architecture)
# Idea: landing-page-ta-track-what-work-is-in-progress-and-be-launch
# IdeaId: -OlMyYDc5kwPsLBhGRoB
# App: command-center
# Context: Defined the landing page as a thin work board that replaces Home, established CC's role as invisible infrastructure, and designed a polymorphic card model with context-aware launch sequences

## ODRC Updates

- RESOLVE OPEN: "CC landing experience should be developer-work-centric, not data-taxonomy-centric. Developer opens CC and sees: what am I working on, what stage is each thing in, what's my next action. Not: here are all your concepts organized by type." → Confirmed. The landing page is a work board of cards sorted by recency/relevance. Each card represents active work (Idea, App, or Project) and the primary action is launching the developer into a session. The page replaces Home and is designed for daily use — everything else is "dig deeper" territory on other pages.

- RESOLVE OPEN: "Landing card design — show active Ideas with phase, last session date, unresolved OPEN count, and recommended next action. One-click to continue any Idea into a Chat session with full context." → Partially resolved. Cards are polymorphic — not just Ideas but also Apps and Projects. Each card knows its type, its destination workspace (Chat, Claude Code, CC views), and the artifacts needed for launch. Card content should be lean and recognizable, not rich/detailed. One-click triggers a launch sequence (brief generation, clipboard prep, file download). Detailed ODRC state and session history stay on deeper pages, not on the card itself. Exact card fields still need design work.

- RESOLVE OPEN: "Smart session recommendation — CC analyzes ODRC gaps per Idea and suggests which session type to run next. 'You have gaps in voice of customer — recommend a VoC session.' Developer clicks, CC generates the brief, they're in Chat." → Deferred. Not addressed this session. The card model supports this concept — a recommended action could surface on the card — but the intelligence behind gap analysis and session type recommendation needs its own exploration.

- RESOLVE OPEN: "Recent activity feed — surface recent deploys, ODRC imports, session completions. Shows the developer what happened since they were last here without navigating to separate views." → Reconsidered. A separate activity feed adds noise and conflicts with the "keep it invisible" philosophy. Activity context is better served through card-level signals (last session date, phase changes) and the summary stats. The landing page should stay lean — the developer wants to launch, not review history.

- RESOLVE OPEN: "Relationship between this developer dashboard and the existing Dashboard view — does this replace it, sit above it, or become an alternative landing page?" → Resolved. The landing page replaces Home entirely. The existing Home content (project/app/idea hierarchy) moves to the Projects page, which already captures most of it. The only Home element retained on the landing page is the drop box (inbound data intake). The only gap is making app version labels on the Projects page launchable (clickable links to deployed apps).

- NEW DECISION: "Landing page replaces Home. Top of page is the drop box (inbound — external data entering CC). Rest of page is work cards (outbound — developer launching into sessions). This creates a clean inbound/outbound architecture for the developer's daily touchpoint."

- NEW DECISION: "Cards are polymorphic work items, not just Ideas. An Idea card launches you into Chat with a session brief. An App card can launch you into Claude Code with repo/issue context. A Project card provides a container view of related Ideas and Apps. The card type determines the destination and the artifacts assembled."

- NEW DECISION: "Cards are context-aware launch packages. Each card knows: what am I (type), where does work happen for me (destination workspace), and what artifacts do I need to bring (session brief, ODRC state, repo info, etc.). The Go action triggers a prep sequence — CC assembles everything, developer does a minimal manual handoff to Claude Chat or Code."

- NEW RULE: "CC is invisible infrastructure. The developer doesn't want to be in CC — they want to be in Claude. CC adds value by making AI sessions dramatically better through context assembly and continuity tracking. The less time spent in CC, the better the product is performing. Design every interaction to minimize clicks and get the developer out fast."

- NEW RULE: "CC is a prep engine, not an integration point. CC cannot programmatically inject context into Claude Chat or Claude Code. It maximizes handoff quality — generating briefs, building prompts, downloading files, copying to clipboard — while the developer completes the last-mile manual steps. Design within this constraint."

- NEW RULE: "Don't guess what developers want on the landing page — ship the minimum and let usage tell you what's missing. Aggregate stats like total OPENs across all Ideas were cut because they raise questions without providing actionable answers."

- NEW CONSTRAINT: "No programmatic integration with Claude Chat or Claude Code exists today. Launch sequences must use manual handoff patterns: file download, clipboard copy, and developer navigation. Design should not depend on future integration capabilities but should not block them either."

- NEW OPEN: "Exact card layout and fields per card type — what minimal information makes each card type (Idea, App, Project) recognizable and launchable at a glance? Needs design session." → Resolved for Idea cards. MVP card fields: Idea name (linked to Idea page), app tag, phase badge with color-coded left stripe, last session number (linked to session detail), full date/time of last session, unresolved OPEN count. Single "Continue" button launches the session brief modal. No overflow menu — inline links handle navigation to deeper views. App and Project card designs deferred to future session.

- NEW DECISION: "Idea card 'Continue' button launches the session brief generation modal, not a page navigation. The card is a launchpad — clicking Continue starts the existing brief workflow without leaving the landing page."

- NEW DECISION: "Idea card has four click targets: card body clicks navigate to the Idea page, app tag clicks navigate to the App page, session number clicks navigate to the last session detail, and the Continue button launches the session brief/prompt builder modal. App tag, session link, and Continue button use stopPropagation to prevent triggering the card-level Idea navigation."

- NEW DECISION: "Cards show full date/time of last session (e.g. 'Feb 13, 2026 2:30 PM') rather than relative time ('today', '2 days ago'). Gives the developer precise context about when they last touched this work."

- RESOLVE OPEN: "Card sorting and prioritization logic — recency is the starting default, but how should urgency, staleness, or external signals affect card order? Is this purely recency-based or does CC apply intelligence to recommend what to work on next?" → Top 5 Ideas by recency of last session. No smart prioritization for MVP. Future iterations can layer in urgency signals or external inputs.

- NEW OPEN: "How will external inputs (Jira issues, bug reports, production alerts) eventually create or reprioritize cards on the landing page? This is a future capability but the card model should accommodate it."

- NEW OPEN: "App launch capability on Projects page — version labels exist today but need to be made clickable/launchable. What's the implementation scope?"

- RESOLVE OPEN: "Drop box positioning and design on the new landing page — does it stay the same as current Home, or does it need to be rethought as the inbound section of an inbound/outbound page layout?" → No change to drop box. Keep existing drop box component at top of page. Cards replace the project/app section below it. Drop box role has simplified — package deploy capability moved to Claude Code, so drop box is now primarily for session artifact intake.

- NEW DECISION: "Empty state — when the developer has no Ideas, the landing page shows the drop box and a prompt to create their first Idea via the New Idea button. The first action for a new user is always New Idea."

- NEW DECISION: "Landing page shows top 5 Ideas by recency. No 'view all' link needed for MVP — the Ideas tab in the nav handles that. Strictly 5 cards max on the landing page to keep it lean."

## Session Notes

### What Was Accomplished
Explored and defined the developer landing page concept for Command Center. Started from the five OPENs captured during an earlier insight and evolved the concept significantly through discussion. The key breakthrough was reframing the landing page not as a miniature version of existing CC pages, but as a thin work board designed around the developer's actual workflow: see what's on my plate, decide what to do, and get launched into a Claude session with full context. Established CC's core product philosophy as invisible infrastructure — the less time developers spend in CC, the better it's working. Defined a polymorphic card model where cards represent different work types (Ideas, Apps, Projects) and each card acts as a context-aware launch package that knows its destination and required artifacts. Resolved the Home page relationship by deciding the landing page replaces Home, with a clean inbound (drop box) / outbound (work cards) architecture. Existing Home content moves to the Projects page with the addition of launchable app version labels. Created an HTML mockup of the landing page using CC's design system, then refined the Idea card design to include linked Idea name, linked last session number, full date/time, phase badge, OPEN count, and a Continue button that launches the session brief modal. The concept is now defined enough to move to spec.

### Key Design Principles Established
- CC is invisible infrastructure — developers want to be in Claude, not in CC
- The landing page is a launchpad, not a dashboard — optimize for getting out fast
- Cards are polymorphic and context-aware — they know their type, destination, and required artifacts
- Keep the noise down — lean cards, minimal information, deeper views live elsewhere
- CC is a prep engine — maximize handoff quality within manual handoff constraints
- Inbound/outbound page architecture — drop box at top for intake, work cards below for launching
- External inputs (Jira, etc.) should eventually drive the flow but aren't in scope now

### Session Status
- Concepts: 2 OPENs remaining (external inputs, app launch on Projects page — both out of scope for spec), 8 DECISIONs, 3 RULEs, 1 CONSTRAINT
- Phase: ready for spec
- Next session: Spec session — write implementation spec for replacing Home with new landing page. Covers: drop box retention, Idea card component, summary stat chips, empty state, removal of project/app section from Home (moves to Projects page). Mockup available as reference.
