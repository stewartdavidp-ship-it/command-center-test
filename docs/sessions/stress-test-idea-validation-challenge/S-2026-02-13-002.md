# ODRC Updates — Stress Test: Idea Validation Challenge
# Date: 2026-02-13
# Session: S-2026-02-13-003 (Session continuity architecture, Skills API discovery, and CC pipeline redesign)
# Idea: stress-test-idea-validation-challenge
# IdeaId: -OlMeQdbaPRy9nPFu6I0
# App: command-center
# Context: Discovered that Anthropic's Skills API enables CC to programmatically deploy and manage session behavior as versioned skills. This led to a fundamental architectural separation: skills define HOW sessions work (stable, versioned infrastructure), briefs define WHAT sessions are about (dynamic, per-session context). Established the session package as the standard output of every session and identified the session manifest as a mid-session durability mechanism.

## ODRC Updates

- RESOLVE OPEN: "Session lifecycle framework — define the full menu of session types (technical, competitive, VoC, economics, stress test, others?), what each one targets, and how CC recommends which session to run based on the Idea's current ODRC state and phase" → Architecture resolved: session types are skill mappings. Each session type has a defined set of skills that CC loads via the Skills API container.skills parameter. Adding a new session type means creating the skills and registering the mapping in CC. The specific session type menu and per-type skill definitions remain to be built, but the architectural approach is decided.

- RESOLVE OPEN: "Session handoff block — how many prior handoffs should the brief generator include? Most recent only, last 2-3, or adaptive based on brief size budget? Need to test with real data to find the right balance" → Superseded by brief reimagining. The brief is now a pure context document carrying ODRC state, historical narrative arc (not just last handoff), session goal, and skill manifest. The handoff remains part of the session package but the brief's historical context section replaces the question of "how many handoffs to include" with a richer approach: full recent handoff plus compressed summaries of earlier sessions plus a one-paragraph origin story for older history. Exact depth is still an OPEN.

- NEW DECISION: "Session protocols are delivered via the Anthropic Skills API, not embedded in briefs or manually loaded. CC creates and versions skills programmatically through the /v1/skills endpoint and attaches them to sessions via the container.skills parameter. This is native to the Anthropic platform and eliminates manual steps for loading session behavior."

- NEW DECISION: "Two-layer architecture separating skills from briefs. Skills define HOW sessions work — operational discipline, protocols, recurring patterns. Briefs define WHAT this session is about — ODRC state, historical context, session goal. Skills are stable and versioned like deployed software. Briefs are dynamic and regenerated fresh each session. These two concerns are never mixed."

- NEW DECISION: "Every session produces a session package. Minimum package contains three components: ODRC updates, session brief (final state), and session handoff. Optional artifacts (code, designs, analyses) are included as produced. Chat produces the package at session close. Developer downloads and uploads to CC. CC handles all downstream processing — GitHub push, metadata extraction, ODRC ingestion, session object updates."

- NEW DECISION: "Chat maintains a session manifest file on disk throughout the session. Created at session start, appended whenever an artifact is produced, read back after context loss. The manifest serves as both an artifact tracker and a recovery index. This protocol lives in a skill, not in the brief — the filesystem is the durable layer that compensates for Chat's unreliable context window."

- NEW DECISION: "CC's session object stores metadata pointers for all session artifacts. ODRC updates are ingested and applied. Session brief and handoff are stored. Additional artifacts go to GitHub with metadata in the session object providing bidirectional traceability — what the file is, where it lives, which session produced it. The session object becomes a complete receipt."

- NEW DECISION: "The session skill package has a base layer plus session-type-specific layers. Base skills load every session: ODRC model and extraction rules, session manifest protocol, session open procedures, session close procedures. Session-type skills layer on top based on CC's selection. CC manages the mapping between session types and their required skills."

- NEW DECISION: "The brief is reimagined as a context-only document. It carries: current ODRC state, session handoff from prior session, historical narrative arc across multiple sessions, session goal, and skill manifest showing which skills are active and why. No behavioral instructions — those live in skills. This dramatically reduces brief size while increasing context richness."

- NEW DECISION: "Skills are treated as deployed infrastructure — versioned and updated only when session behavior needs to change. This is equivalent to deploying a new version of an application. The dynamic, session-specific information is delivered through the brief. Skills define how we sculpt Chat's behavior; briefs provide the data Chat operates on."

- NEW DECISION: "Session close is triggered by a developer phrase (e.g., 'close session' or 'session package'). Chat re-reads the session close skill before producing any output, even if it believes it remembers the contents. The skill itself includes this self-reinforcing instruction. This compensates for context loss in long sessions."

- NEW OPEN: "How deep does the historical context in the brief go? Options: full handoff from every prior session (gets heavy fast), rolling summary that CC compresses over time (requires summarization logic), or tiered approach — full recent handoff, compressed summaries for 2-3 sessions before that, one-paragraph origin story for older history. Needs to balance context richness against brief size budget."

- NEW OPEN: "What specific skills comprise the base session package? Categories identified: ODRC model and extraction rules, session manifest protocol, session open procedures, session close procedures. Need to write the actual skill definitions — this is the next implementation step."

- NEW OPEN: "What session types exist beyond stress test and exploration, and what is each type's specific skill mapping? Previously named: technical deep-dive, competitive analysis, voice of customer, economics. Full menu and per-type skill requirements are undefined."

- NEW OPEN: "How does CC manage the skill registry — creation, versioning, and mapping of skills to session types? This is the CC engineering work to integrate with the Skills API. Includes initial skill creation, version management when protocols evolve, and the session-type-to-skill mapping logic."

- NEW OPEN: "Does the Skills API skill attachment work with Claude.ai manual chat sessions or only with direct API calls? CC makes API calls for brief generation, but when the developer starts a manual chat session in Claude.ai, the skill attachment mechanism may differ. Needs technical validation."

- NEW OPEN: "Session close skill template — what is the exact procedure Chat follows at session close? Needs to define: re-read the skill, read the session manifest, verify all artifacts are accounted for, produce ODRC updates in standard format, produce session handoff in standard format, assemble and present the complete package. This skill needs to be written and tested."

- NEW RULE: "Every session must produce a minimum session package: ODRC updates, session brief, and session handoff. No exceptions. CC flags missing components during ingestion."

- NEW RULE: "Skills are the single source of truth for session behavior. Briefs never contain behavioral instructions — only context. If a behavioral pattern needs to change, it's a skill version update, not a brief template change."

- NEW RULE: "Any protocol that needs to survive context loss must use the filesystem as its durable layer. Chat's memory is unreliable — the session manifest, transcripts, and skills all live on disk and can be re-read at any time."

- NEW RULE: "At session close, Chat must re-read the session close skill before producing output, regardless of session length or perceived context retention. The skill includes this self-reinforcing instruction."

- NEW CONSTRAINT: "The Anthropic Skills API is currently in beta (skills-2025-10-02). CC's integration depends on this API remaining available and stable. Breaking changes would require CC adaptation."

- NEW CONSTRAINT: "Skills operate in a sandboxed execution environment with no data persistence between sessions. Skill definitions persist at the workspace level, but runtime state (files created during a session) does not carry over."

- NEW CONSTRAINT: "Maximum 8 skills can be attached per API request. The base package plus session-type-specific skills must fit within this limit, constraining how granularly skills can be decomposed."

## Session Notes

### What Was Accomplished
Session 3 began as a test of session continuity — evaluating whether the Session 2 handoff and brief provided adequate context for a new session. The evaluation revealed real gaps: the ODRC updates from Session 2 hadn't been ingested into CC, demonstrating the exact manual-step failure mode that Session 2 predicted. This led to designing the session package concept (standardized output of every session) and the session manifest (mid-session artifact tracking on disk). The breakthrough discovery was that Anthropic's Skills API allows CC to programmatically create, version, and attach skills to sessions — eliminating the need to embed behavioral instructions in briefs or manually load skills. This produced a fundamental architectural insight: skills are deployed infrastructure (stable, versioned, defining HOW sessions work), while briefs are runtime data (dynamic, per-session, defining WHAT sessions are about). CC becomes an orchestrator that assembles the right skills and context for each session type.

### Key Design Principles Established
- Skills are deployed infrastructure, briefs are runtime data — never mix the two
- The filesystem is the durable layer that compensates for Chat's unreliable context window
- CC's value is imposing discipline on a general-purpose tool without reducing its capability
- Manual steps in the pipeline will be skipped under pressure — automate or accept the failure
- Session close requires re-reading the skill, not trusting memory — self-reinforcing instruction

### Session Status
- Concepts: 2 OPENs remaining (from original 6, carried from Session 2) + 6 new OPENs, 24 DECISIONs, 6 RULEs, 5 CONSTRAINTs
- Phase: converging (stress test mechanics complete, now defining CC pipeline architecture)
- Next session: Write the base session skills — ODRC model skill, session manifest skill, session open skill, session close skill. These are the foundation everything else builds on.
