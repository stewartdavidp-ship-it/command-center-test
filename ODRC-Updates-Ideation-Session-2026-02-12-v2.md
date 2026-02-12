# ODRC Updates — Ideation Workflow & Smart Intake Session
# Date: 2026-02-12
# Session: Ideation Workflow Design (Session 4)
# Context: Designing the Idea → Chat → CC iterative flow
# Note: Session pivoted from design to code review — revealed significant existing infrastructure

---

## ODRC Updates

### Decisions

- NEW DECISION: "CC uses ClaudeAPIService to generate session briefs for ideation exploration — AI builds the prompt from Idea context, no template logic or variant selection in CC"
- NEW DECISION: "Ideation session output is dual-tracked: full session document saved to GitHub as immutable record, ODRC metadata (new/resolved OPENs, DECISIONs, RULEs, CONSTRAINTs) updates living concept objects in Firebase via ConceptManager"
- NEW DECISION: "Ideas are evolving containers with cumulative ODRC state and session history — each exploration session's output accumulates on the Idea, and the next session's prompt is built from the full accumulated state"
- NEW DECISION: "All AI-generated ideation prompts are framed in ODRC terms — the AI structures exploration around OPENs to surface, DECISIONs to make, RULEs to establish, and CONSTRAINTs to define, ensuring every session produces ingestible ODRC output"
- NEW DECISION: "Session prompt generation combines three inputs: current concept state from Firebase (what's open, decided, ruled), previous session documents from GitHub (reasoning and context history), and app-level context (version, description, existing specs)"
- NEW DECISION: "Ideation flow builds on existing CC infrastructure: SessionBriefGenerator patterns for structure, ODRCSummaryGenerator for ODRC state, ClaudeAPIService for AI calls, ODRCUpdateIngestionService for return path parsing — not a new system, an extension of what exists"
- NEW DECISION: "Every Chat session begins by asking the developer what they want to accomplish, then responds with what additional context is needed before proceeding — the developer's stated goal drives context requirements, not a predetermined maturity model"

### Rules

- NEW RULE: "ODRC is the thinking framework for all ideation conversations — every Chat session exploring an Idea must produce structured ODRC output, not just prose conclusions"
- NEW RULE: "Every ideation session is additive to the decision record — no session output is throwaway, each conversation advances the cumulative ODRC state of the Idea"
- NEW RULE: "Session context requirements are driven by developer intent, not session type — a spec-bound session requires code, an exploratory session may not, and the developer's stated goal determines what CC packages"

### OPENs

- NEW OPEN: "Where do ideation session documents live in the repo structure? Options include cc/sessions/{idea-slug}/session-NNN.md, docs/sessions/, or a new top-level directory"
- NEW OPEN: "Token budget for session prompts — how much previous session document content to include vs. relying on current ODRC concept state? Full history may exceed context window for long-running Ideas"
- NEW OPEN: "How does CC present the generated session brief and supporting docs? Modal with copy, download as markdown, auto-attach codebase for spec-level sessions?"
- NEW OPEN: "Should the ODRC Import modal be accessible from Idea Detail view for the return path, or is a broader drop zone solution needed first?"
- NEW OPEN: "Add 'ideation' to SESSION_TYPES or build a parallel path that uses the same SessionBriefGenerator patterns but pulls from Idea/Concept data instead of work items?"

### Resolved

- RESOLVE OPEN 13: "Bulk import of ODRC concepts from existing documents" → Subsumed by Smart Intake design — CC's file drop detects ODRC markdown patterns (## DECISIONS, ## OPENs, ## RULES, ## CONSTRAINTS headers) and routes to import flow with confirmation checklist. Immediate seeding of v4 ODRC is the first test case.

---

## Session Notes

### Key Finding: Existing Infrastructure Coverage
Code review revealed CC already has significant infrastructure applicable to the ideation flow:
- **SessionBriefGenerator** — generates structured briefs with app context, deploy history, session types, context strategies
- **SESSION_TYPES** — 7 types (build, design, fix, test, research, review, refactor) with role framing, scope rules, delivery requirements
- **ODRCSummaryGenerator** — renders ODRC state as markdown (scoped or full)
- **ODRCUpdateIngestionService** — parses and executes structured ODRC updates against Firebase
- **ClaudeAPIService** — working API calls with configurable model and token limits
- **GenerateCLAUDEModal** — generates CLAUDE.md from app concepts with copy/push actions

### What Needs Building (Narrower Than Initially Scoped)
1. Add ideation-aware session type or extend SessionBriefGenerator to pull from Idea/Concept state
2. "Explore in Chat" action on Idea Detail view triggering AI-powered brief generation
3. Extend Idea data model in Firebase to track session history and lifecycle
4. Make ODRC Import accessible from Idea Detail (return path)
5. Session document storage in GitHub repo

### Session Retrospective
Session started without code review, leading to redundant design discussion about infrastructure that already existed. Key learning: developer's stated goal ("spec the plumbing") should have triggered immediate code review. For future sessions, asking what the developer wants to accomplish and requesting appropriate context up front would have made this session significantly more productive.
