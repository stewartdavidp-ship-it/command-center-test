# Spec Unit 4 — Landing Page & Session Card UX

**Idea:** Session Tab | **App:** Command Center | **Spec ID:** sp_session_tab_unit4
**Base version:** 8.67.0 | **Phase:** spec-ready
**Depends on:** Unit 1 (session.json schema), Unit 2 (session lifecycle & idea phase model)
**Date:** 2026-02-14

---

## Task Summary

Transform the home page work card section from a simple "recent ideas" list into a session-driven operational surface. The home page becomes the primary cockpit for the full session lifecycle: see all active ideas, understand both session state and idea phase at a glance, launch sessions via Continue, and ingest results — all without leaving the landing page. The session tab (IdeasView) remains the detail/history view.

---

## What to Build

### 1. Session Card Redesign (IdeaWorkCard → SessionCard)

The current `IdeaWorkCard` (line ~15875) displays idea metadata and a Continue button. It needs to evolve into a dual-state session card that shows both session state and idea phase as independent dimensions.

#### 1.1 Card Data Model

The card consumes an enriched idea object from `getRecentIdeas()` (line ~6107). Extend the enrichment to include session lifecycle state:

```javascript
// In getRecentIdeas(), add to the returned object:
{
    ...idea,
    // Existing fields
    openCount, totalConcepts, appName, lastSession, sessionCount, phase,
    // New session lifecycle fields
    sessionState: idea.activeSession ? 'active' : 'pending',
    activeSessionId: idea.activeSession?.sessionId || null,
    activeSessionCreatedAt: idea.activeSession?.createdAt || null,
    briefDownloaded: idea.activeSession?.briefDownloaded || false,
    staleDays: idea.activeSession
        ? Math.floor((Date.now() - new Date(idea.activeSession.lastActivityAt).getTime()) / 86400000)
        : null
}
```

Session state values and their meaning:
- **pending** — No active session. Ready for a new session cycle. Card shows "Continue" action.
- **active** — Brief has been dispatched (downloaded). Session is in-flight. Card shows active visual indicator and session context.

The `complete` state is transient — when a session package is ingested, `activeSession` is cleared and the card returns to `pending` with updated ODRC state. The completed session record lives in `sessionLog`.

#### 1.2 Card Visual States

**Pending state** (no activeSession):
- Left stripe: phase color (existing `PHASE_COLORS` — line ~15867)
- Content: idea name, app badge, phase badge, session count, date of last session, OPEN count
- Action: "Continue →" button (existing gradient style)

**Active state** (activeSession exists):
- Left stripe: replaced by green glow or green left border to distinguish from phase coloring
- Content: idea name, app badge, phase badge, active session ID, time since brief dispatched
- Session context line: "Session {id} active · Brief dispatched {timeago}"
- Action: button changes from "Continue →" to context-appropriate:
  - If brief downloaded but no package yet: "📦 Upload Results" or show file drop target
  - Staleness escalation: if `staleDays >= 7`, add amber warning indicator: "⚠️ {N} days since brief dispatched"

**Active session guard** (existing, line ~16008): When a user clicks Continue on an idea that already has an activeSession, the modal already warns and blocks. This is implemented in v8.67.0.

#### 1.3 Card Component Changes

Rename `IdeaWorkCard` to `SessionCard` (or keep internal name but evolve the rendering):

```jsx
function SessionCard({ idea, onNavigateIdea, onNavigateApp, onContinue, onUploadResults }) {
    const isActive = idea.sessionState === 'active';
    const isStale = idea.staleDays >= 7;
    const phase = PHASE_COLORS[idea.phase] || PHASE_COLORS.exploring;

    // Card border/glow for active sessions
    const cardBorder = isActive
        ? 'border-green-600/60 shadow-[0_0_12px_rgba(34,197,94,0.15)]'
        : 'border-slate-700/50';

    // Left stripe: phase color for pending, green for active
    const stripeColor = isActive ? '#22c55e' : phase.stripe;

    return (
        <div className={`flex items-center gap-4 p-4 rounded-xl border bg-slate-800/50 
            cursor-pointer transition-all hover:shadow-lg hover:-translate-y-0.5 ${cardBorder}`}
            onClick={() => onNavigateIdea(idea.id)}>

            <div className="w-1 h-12 rounded-full flex-shrink-0" 
                style={{ background: stripeColor }} />

            <div className="flex-1 min-w-0">
                {/* Row 1: Name + App badge */}
                <div className="flex items-center gap-2 mb-1">
                    <span className="text-sm font-semibold text-slate-100 truncate">
                        {idea.name}
                    </span>
                    <span className="text-xs font-mono px-2 py-0.5 rounded 
                        bg-indigo-500/15 text-indigo-400 flex-shrink-0"
                        onClick={e => { e.stopPropagation(); onNavigateApp(idea.appId); }}>
                        {idea.appName}
                    </span>
                </div>

                {/* Row 2: Phase + Session state + metadata */}
                <div className="flex items-center gap-2 text-xs text-slate-500">
                    {/* Phase badge (idea maturity) */}
                    <span className="px-2 py-0.5 rounded font-semibold uppercase tracking-wider"
                        style={{ background: phase.bg, color: phase.text, fontSize: '10px' }}>
                        {idea.phase}
                    </span>

                    {/* Session state indicator */}
                    {isActive ? (
                        <span className="px-2 py-0.5 rounded bg-green-900/50 text-green-400 
                            font-semibold uppercase tracking-wider" style={{ fontSize: '10px' }}>
                            active
                        </span>
                    ) : (
                        <span>Session {idea.sessionCount}</span>
                    )}

                    {/* Context line */}
                    {isActive ? (
                        <React.Fragment>
                            <span className="opacity-40">·</span>
                            <span>{idea.activeSessionId}</span>
                            <span className="opacity-40">·</span>
                            <span>dispatched {formatTimeAgo(idea.activeSessionCreatedAt)}</span>
                            {isStale && (
                                <span className="text-amber-400 font-medium">
                                    ⚠️ {idea.staleDays}d
                                </span>
                            )}
                        </React.Fragment>
                    ) : (
                        <React.Fragment>
                            {idea.lastSession && (
                                <React.Fragment>
                                    <span className="opacity-40">·</span>
                                    <span>{formatDate(idea.lastSession.date)}</span>
                                </React.Fragment>
                            )}
                            <span className="opacity-40">·</span>
                            <span className={`font-mono font-semibold 
                                ${idea.openCount > 0 ? 'text-amber-400' : 'text-slate-600'}`}>
                                {idea.openCount} OPENs
                            </span>
                        </React.Fragment>
                    )}
                </div>
            </div>

            {/* Action button */}
            <button
                onClick={e => { e.stopPropagation(); 
                    isActive ? onUploadResults(idea) : onContinue(idea); 
                }}
                className="flex items-center gap-1.5 px-4 py-2 rounded-lg text-sm 
                    font-semibold text-white flex-shrink-0 transition-all 
                    hover:-translate-y-0.5 hover:shadow-md"
                style={{ background: isActive 
                    ? 'linear-gradient(135deg, #22c55e, #16a34a)' 
                    : 'linear-gradient(135deg, #667eea, #764ba2)' }}>
                {isActive ? 'Upload Results' : 'Continue'}
                <svg width="14" height="14" viewBox="0 0 24 24" fill="none" 
                    stroke="currentColor" strokeWidth="2.5" strokeLinecap="round" 
                    strokeLinejoin="round">
                    {isActive 
                        ? <path d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-8l-4-4m0 0L8 8m4-4v12"/>
                        : <path d="M5 12h14M12 5l7 7-7 7"/>}
                </svg>
            </button>
        </div>
    );
}
```

#### 1.4 Helper Functions

Add a `formatTimeAgo` utility near `generateSessionId` (line ~6094):

```javascript
function formatTimeAgo(dateStr) {
    if (!dateStr) return '';
    const ms = Date.now() - new Date(dateStr).getTime();
    const minutes = Math.floor(ms / 60000);
    if (minutes < 60) return `${minutes}m ago`;
    const hours = Math.floor(minutes / 60);
    if (hours < 24) return `${hours}h ago`;
    const days = Math.floor(hours / 24);
    return `${days}d ago`;
}
```

### 2. Session Lifecycle Integration on Home Page

#### 2.1 Continue Flow (Pending → Active)

The existing flow is already implemented in v8.67.0:

1. User clicks **Continue** on a pending card
2. `ExploreInChatModal` opens, generates brief via `IdeationBriefGenerator`
3. User configures lens/mode, previews brief
4. User clicks **Download Zip** → `IdeaManager.activateSession()` fires (line ~16030)
5. Card transitions from pending to active state

No changes needed to this flow. The card visual update happens reactively because `globalIdeas` updates via Firebase listener when `activeSession` is set.

#### 2.2 Brief Generation Safeguard

Already implemented in v8.67.0 (line ~16003-16008): If the user clicks Continue on an idea with an active session, the modal alerts and blocks.

Additional safeguard for the card level: the card itself changes its action button from "Continue" to "Upload Results" when active, preventing accidental re-dispatch at the card level before the modal even opens.

#### 2.3 Upload Results Flow (Active → Complete → Pending)

When a user clicks **Upload Results** on an active card, trigger a file picker or drop zone for the session package zip. The ingestion flow then:

1. User selects/drops the session package zip
2. CC extracts files, detects `session.json` via `detectInboundArtifactType()` (line ~3006)
3. `SessionPackageProcessor.validate()` runs (line ~2812)
4. If valid, show `ODRCImportChecklistModal` (existing component) pre-linked to the active idea
5. On successful import, `executeODRCImport()` (line ~3066) writes concepts and calls `IdeaManager.completeSession()` (line ~3115)
6. `activeSession` clears → card returns to pending state with updated ODRC counts

**New handler on DashboardView** — add `workCardUploadResults`:

```javascript
const workCardUploadResults = async (idea) => {
    // Open file picker for session package
    const input = document.createElement('input');
    input.type = 'file';
    input.accept = '.zip,.json,.md';
    input.onchange = async (e) => {
        const files = Array.from(e.target.files);
        if (!files.length) return;
        // Route through existing file processing pipeline
        // Pre-set the linked idea so the ODRC import modal auto-selects it
        for (const file of files) {
            await onFileDrop([file], idea.id); // Pass ideaId hint
        }
    };
    input.click();
};
```

Pass this as `onUploadResults` prop to `SessionCard`.

#### 2.4 Abandon Session

When an active session has been stale for extended periods, the user needs a way to abandon it. Add an abandon action accessible from the active card's context menu or the ExploreInChatModal:

- On the active card, a small "✕" or overflow menu with "Abandon session"
- Calls `IdeaManager.abandonSession()` (already implemented at line ~5833)
- Confirms with the user first
- Logs the abandoned session in sessionLog with type 'abandoned'
- Clears activeSession → card returns to pending

### 3. Home Page Section Evolution

#### 3.1 Section Header

Update the "Recent Work" section header (line ~12467) to reflect that these are session-driven cards:

```jsx
<div className="text-xs font-semibold text-slate-600 uppercase tracking-widest">
    Active Work
</div>
```

"Active Work" better reflects that these cards are the operational surface, not just recent history.

#### 3.2 Card Count and Visibility

Currently `getRecentIdeas` returns `maxCount = 5` most recently active ideas. With the session card model:

- **Active session cards sort first** — Ideas with `activeSession` should appear at the top of the list regardless of lastSessionDate, since they represent in-flight work.
- **Pending cards sort by recency** — Below active cards, sort by lastSessionDate descending (existing behavior).
- **Consider increasing maxCount** — With active sessions always pinned to top, 5 may be tight. Consider 8-10, or make it configurable.

Update `getRecentIdeas`:

```javascript
function getRecentIdeas(globalIdeas, globalConcepts, apps, maxCount = 8) {
    const active = [...(globalIdeas || [])]
        .filter(idea => idea.status === 'active');

    // Partition: active sessions first, then pending by recency
    const withSession = active.filter(i => i.activeSession);
    const withoutSession = active.filter(i => !i.activeSession);

    const sortedWithout = withoutSession.sort((a, b) => {
        const dateA = a.lastSessionDate || new Date(a.updatedAt || a.createdAt || 0).getTime();
        const dateB = b.lastSessionDate || new Date(b.updatedAt || b.createdAt || 0).getTime();
        return (typeof dateB === 'number' ? dateB : 0) - (typeof dateA === 'number' ? dateA : 0);
    });

    const combined = [...withSession, ...sortedWithout].slice(0, maxCount);
    // ... existing enrichment logic
}
```

#### 3.3 Ideas Falling Off the Home Page

Ideas with `status === 'active'` appear on the home page, capped by maxCount. Ideas naturally fall off the bottom of the list as newer ideas push them down. They remain accessible via the IdeasView (Ideas tab) which shows all ideas regardless of recency.

Ideas leave the landing page entirely when they reach terminal state:
- `status === 'archived'` — developer explicitly archived
- `phase === 'complete'` — idea fully realized
- `status === 'graduated'` — idea shipped

No automatic archival based on time. The developer decides.

### 4. Chrome Session Package Detection

#### 4.1 Detection Mechanism

Reuse the existing completion file detection pattern (`CompletionFileService` + `buildDetectionSignatures`). Session packages are identified by filename pattern:

**Filename signature:** `session-brief-{slug}-S{NNN}.zip` (this is the existing zip naming from ExploreInChatModal line ~16044)

The detection flow:
1. CC has File System Access API directory handle (Chrome only, existing capability for completion files)
2. On tab focus / visibility change, poll the directory
3. Match files against session package filename patterns
4. When matched, surface an ingestion prompt banner on the dashboard

#### 4.2 Session Package Detection Signatures

Add session package patterns alongside completion file patterns in the detection config:

```javascript
// In buildDetectionSignatures or a parallel function:
buildSessionDetectionSignatures(globalIdeas) {
    return (globalIdeas || [])
        .filter(i => i.activeSession)
        .map(idea => ({
            ideaId: idea.id,
            sessionId: idea.activeSession.sessionId,
            pattern: new RegExp(
                `session-brief-${idea.slug.replace(/[^a-z0-9-]/g, '')}-S\\d{3}\\.zip$`, 'i'
            )
        }));
}
```

#### 4.3 Ingestion Prompt (Chrome)

When a session package is detected in the watched directory:

1. Surface a non-blocking notification banner at the top of the dashboard:
   ```
   📦 Session package detected for "{Idea Name}" — [Review & Ingest] [Dismiss]
   ```
2. Clicking "Review & Ingest" opens the file, runs validation, and shows the ODRC import checklist modal pre-linked to the matching idea.
3. Dismissing hides the banner for that file (track dismissed filenames to avoid re-prompting).

#### 4.4 Non-Chrome Fallback

For Safari/Firefox users, the session card's **Upload Results** button is the primary path. Additionally, the existing file drop zone on the dashboard accepts session packages. The staleness indicator on active cards provides the passive reminder.

### 5. Staleness Safeguard at Brief Generation

#### 5.1 State Tracking

Already implemented in v8.67.0. `IdeaManager.activateSession()` records:
```javascript
{
    sessionId, status: 'active', createdAt, lastActivityAt,
    briefDownloaded: false, artifactsReceived: 0, ideaPhaseAtStart
}
```

The safeguard fires when `activeSession` exists and the user attempts to start a new session (already guarded in ExploreInChatModal).

#### 5.2 Card-Level Visual Staleness

The card shows staleness progressively:
- **< 7 days:** Green active indicator, no warning
- **≥ 7 days:** Amber warning indicator on the card: "⚠️ {N}d"
- **≥ 14 days:** Consider adding a "stale" visual state — amber border instead of green, prompting the user to either upload results or abandon

These thresholds are starting values. Track actual usage patterns to refine.

### 6. Session Tab (IdeasView) — History View

The session tab remains the detail/history view. When a user clicks on a card to navigate to the idea detail (IdeasView mode='idea-detail'), the session history is already rendered from `idea.sessionLog`.

#### 6.1 Enriched Session History Display

The session log entries now carry richer fields from session.json ingestion (v8.67.0):
- `chain` — link count, summation flag, total concept blocks, total elapsed minutes
- `debriefSummary` — first 2-3 paragraphs from debrief.md
- `nextSession` — recommendation for next session focus
- `status` — complete, abandoned
- `ideaPhaseAtStart`, `ideaPhaseAtEnd` — phase transitions during the session

The existing session history rendering (in IdeasView idea-detail mode) should surface these when available:

```jsx
{sessionLog.map(session => (
    <div key={session.sessionId} className="border border-slate-700 rounded-lg p-3 mb-2">
        <div className="flex items-center justify-between mb-1">
            <span className="text-sm font-semibold">{session.sessionId}</span>
            <span className="text-xs text-slate-500">
                {new Date(session.date).toLocaleDateString()}
            </span>
        </div>
        <p className="text-sm text-slate-300">{session.summary}</p>
        
        {/* Chain metadata */}
        {session.chain && (
            <div className="text-xs text-slate-500 mt-1">
                {session.chain.linkCount} link(s) · 
                {session.chain.totalConceptBlocks} concept blocks · 
                {session.chain.totalElapsedMinutes}min
            </div>
        )}
        
        {/* Debrief summary (expandable) */}
        {session.debriefSummary && (
            <details className="mt-2">
                <summary className="text-xs text-indigo-400 cursor-pointer">
                    View debrief summary
                </summary>
                <p className="text-xs text-slate-400 mt-1 whitespace-pre-wrap">
                    {session.debriefSummary}
                </p>
            </details>
        )}
        
        {/* Next session recommendation */}
        {session.nextSession && (
            <div className="text-xs text-slate-500 mt-1 italic">
                Next: {session.nextSession}
            </div>
        )}
        
        {/* Phase transition */}
        {session.ideaPhaseAtStart && session.ideaPhaseAtEnd && 
         session.ideaPhaseAtStart !== session.ideaPhaseAtEnd && (
            <div className="text-xs mt-1">
                Phase: <span className="text-slate-400">{session.ideaPhaseAtStart}</span>
                → <span className="text-green-400">{session.ideaPhaseAtEnd}</span>
            </div>
        )}
    </div>
))}
```

---

## Existing Infrastructure Reference

| Component | Location | Purpose |
|-----------|----------|---------|
| `IdeaWorkCard` | Line ~15875 | Current home page card — evolves into SessionCard |
| `getRecentIdeas()` | Line ~6107 | Enriches ideas for card display — extend with session state |
| `PHASE_COLORS` | Line ~15867 | Phase-to-color mapping — unchanged |
| `ExploreInChatModal` | Line ~15926 | Session launch modal — already has active session guard |
| `IdeationBriefGenerator` | Line ~3180 | Brief generation — unchanged |
| `IdeaManager.activateSession()` | Line ~5800 | Creates activeSession on idea — already implemented |
| `IdeaManager.completeSession()` | Line ~5815 | Clears activeSession — already implemented |
| `IdeaManager.abandonSession()` | Line ~5833 | Logs abandoned session, clears activeSession — already implemented |
| `SessionPackageProcessor` | Line ~2811 | Validates and parses session.json — unchanged |
| `executeODRCImport()` | Line ~3066 | Writes concepts, clears activeSession — already handles lifecycle |
| `ODRCImportChecklistModal` | Exists | Review/confirm ODRC items before import — unchanged |
| `computeIdeaPhase()` | Line ~6081 | Ratio-based phase heuristic — unchanged |
| `DashboardView` | Line ~10494 | Home page container — hosts work cards section |
| `buildDetectionSignatures()` | Line ~4516 | Completion file detection — pattern for session packages |
| `CompletionFileService` | Line ~6141 | Completion file tracking — pattern to follow for session detection |

---

## Architecture Rules

### State Management Rules
- Session state (`activeSession`) lives on the idea record in Firebase, not in local component state. Cards react to Firebase changes via the existing `IdeaManager.listen()` → `setGlobalIdeas` pipeline.
- The card never directly mutates `activeSession`. All transitions go through `IdeaManager` methods which write to Firebase.
- `getRecentIdeas()` is the single enrichment point for card data. All computed fields (sessionState, staleDays, etc.) are derived there, not in the card component.

### Data Flow Rules
- Home page → card click → IdeasView navigation uses existing `setView('ideas')` + `setViewPayload({ ideaId })` pattern (line ~10506).
- Continue → ExploreInChatModal → Download Zip → `activateSession()` → Firebase updates → cards re-render. No manual refresh needed.
- Upload Results → file picker → existing drop handler → ODRC import → `completeSession()` → Firebase updates → card returns to pending.

---

## Conventions

- Session card component may keep the internal name `IdeaWorkCard` to minimize diff, but all new code and comments should reference the "session card" concept.
- Phase colors follow `PHASE_COLORS` constant. Active session green is `#22c55e` (Tailwind green-500).
- Staleness thresholds (7 days warning, 14 days escalated) are starting values. Add constants at the top of the component:
  ```javascript
  const STALE_WARNING_DAYS = 7;
  const STALE_ESCALATED_DAYS = 14;
  ```
- Time formatting uses the new `formatTimeAgo()` utility, not raw date strings.
- Filename: session package zips follow existing pattern from ExploreInChatModal: `session-brief-{slug}-S{NNN}.zip`

---

## File Structure

All changes are in `cc/index.html` (single-file application):

| Section | Changes |
|---------|---------|
| `getRecentIdeas()` (~6107) | Add session state enrichment, active-first sorting, increase maxCount |
| `formatTimeAgo()` (new, ~6094) | New utility function |
| `IdeaWorkCard` (~15875) | Evolve to dual-state card with active/pending rendering |
| `DashboardView` (~10494) | Add `workCardUploadResults` handler, pass to card |
| Work cards section (~12464) | Update section header, pass new prop |
| IdeasView idea-detail (~16191+) | Enrich session history rendering with chain/debrief fields |
| Session detection (new or ~4516) | Add session package detection signatures alongside completion files |

---

## Decision Validation Checkpoint

This section maps the Decisions from prior sessions to how the spec addresses them, and flags any gaps.

| Decision | Spec Coverage | Status |
|----------|--------------|--------|
| Home page displays session cards, not idea cards. Session state and idea state are independent dimensions. | §1.1-1.3: Card shows both dimensions. Phase badge = idea state, session state badge = active/pending. | ✅ Covered |
| Session states: pending → active → complete. Live landing page and session tab are same data at different lifecycle stages. | §1.1, §2.1-2.3: Full lifecycle on card. Complete is transient (clears to pending). Session tab shows history from sessionLog. | ✅ Covered |
| Ideas remain on landing page as active work across session cycles. Leave at terminal state. | §3.3: Explicit — no automatic archival. Terminal states are archived/complete/graduated. | ✅ Covered |
| Brief generation creates the contract. CC tracks dispatched sessions. | §2.1: activateSession fires on zip download. Card transitions to active. Already implemented v8.67.0. | ✅ Covered (existing) |
| Brief generation safeguard — prompt if brief downloaded but no output uploaded. | §2.2, §5.1-5.2: Active session guard in modal (existing). Card-level visual staleness with escalating indicators. | ✅ Covered |
| Session detection reuses completion file polling (Chrome only). | §4.1-4.3: Filename pattern matching against active sessions. Ingestion banner on detection. | ✅ Covered |
| All-browser fallback: state-based staleness warnings. | §4.4, §5.2: Upload Results button on active cards. Staleness escalation visible on card. | ✅ Covered |
| Session-to-idea is strict one-to-one. | §2.1: activateSession throws if idea already has active session. Card and modal both guard. | ✅ Covered (existing) |
| Continue button on home page card triggers brief generation. | §2.1: Continue → ExploreInChatModal → generate brief. No change to trigger mechanism. | ✅ Covered (existing) |
| Session card displays both session state and idea state. | §1.2-1.3: Phase badge + session state badge rendered independently. | ✅ Covered |
| Debrief is highest-value artifact, stored on session record. | §6.1: debriefSummary rendered in session history with expandable detail. | ✅ Covered |
| Primary success metric is warm-up speed in Session N+1. | §6.1: nextSession recommendation displayed in history. debriefSummary available for brief generation context. | ✅ Covered (feeds into brief generator, not this spec's scope) |

### Flags and Concerns

1. **File drop routing with idea hint**: §2.3 proposes passing an `ideaId` hint to `onFileDrop()` so the ODRC import modal auto-selects the correct idea. The existing `onFileDrop` signature may need a second parameter. Verify the DashboardView prop chain supports this.

2. **Detection signature lifecycle**: §4.2 builds detection signatures from active sessions in `globalIdeas`. These need to rebuild when ideas update (session activated or completed). Confirm this integrates with the existing polling interval — signatures should refresh on each poll, not be cached.

3. **Session card rename**: The spec allows keeping `IdeaWorkCard` as the internal component name to minimize diff. If this creates confusion during future maintenance, consider a clean rename in a follow-up commit.

4. **maxCount increase**: §3.2 increases from 5 to 8. If users have many active ideas, this could make the home page long. Consider a "Show all" toggle or collapsible section rather than a hard cap increase.

---

## Post-Task Obligations

RULE: Before reporting this task as complete:
1. Commit all code changes to the repo
2. Archive this CLAUDE.md to `cc/specs/sp_session_tab_unit4.md`
3. Generate a completion file to `.cc/completions/` per the format below
4. Commit the spec archive and completion file together in a separate commit

**Completion file naming:** `YYYY-MM-DDTHH-MM-SS_session-tab-unit4.md`

**Completion file format:**

```yaml
---
task: "Session Tab Unit 4 — Landing Page & Session Card UX"
status: complete | partial
cc-spec-id: sp_session_tab_unit4
files:
  - path: "cc/index.html"
    action: modified
commits:
  - sha: "{sha}"
    message: "{message}"
odrc:
  new_decisions:
    - "{implementation decisions}"
  resolved_opens:
    - "{resolved}"
  new_opens:
    - "{new questions}"
unexpected_findings:
  - "{unexpected}"
unresolved:
  - "{not completed}"
---

## Approach

{Brief narrative of build approach}

## Implementation Notes

{Key technical details}
```

Do not wait for the developer to ask. Generate the completion file automatically after committing code.
