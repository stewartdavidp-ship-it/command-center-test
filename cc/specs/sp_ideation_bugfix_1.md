# CLAUDE.md — ODRC Ingestion Bugfix + Pipeline Gaps
# cc-spec-id: sp_ideation_bugfix_1
# App: Command Center (index.html)
# Base version: 8.61.0
# Target version: 8.61.1
# Depends on: sp_ideation_pipeline (v8.61.0)

---

## Task Summary

Fix three bugs in the ODRC ingestion pipeline and address two pipeline gaps found during flow audit. The Idea-to-Chat pipeline (v8.61.0) was built correctly but the return path has parsing issues that silently drop RESOLVE lines, a false positive detection problem, and the existing Generate CLAUDE.md feature doesn't respect idea phase maturity.

---

## What to Fix

### Bug 1: RESOLVE OPEN Parser Drops All Freeform Resolutions (CRITICAL)

**Location:** `ODRCUpdateIngestionService.parse()` at ~line 2683

**Problem:** The only RESOLVE pattern is:
```javascript
/^- RESOLVE OPEN:\s*"?(.+?)"?\s*→\s*matched to concept_id\s+(.+)$/i
```
This only matches `- RESOLVE OPEN: "desc" → matched to concept_id {id}`.

Real ODRC output uses freeform resolution text:
```
- RESOLVE OPEN: "Where do session docs live?" → docs/sessions/{slug}/S-YYYY-MM-DD-NNN.md
```
And sometimes numbered format from older files:
```
- RESOLVE OPEN 13: "Bulk import of ODRC concepts" → Subsumed by Smart Intake design
```

**All RESOLVE lines are silently dropped.** The parser logs them as "unparseable" but the import continues with only NEW items. The developer sees 19 items in the checklist instead of 24 — missing 5 RESOLVE actions — and has no way to know they were lost.

**Fix:** Add two flexible RESOLVE patterns BEFORE the strict concept_id pattern:

```javascript
// RESOLVE OPEN: "desc" → any freeform resolution text
const resolveFlexMatch = normalized.match(
  /^- RESOLVE OPEN:\s*"?(.+?)"?\s*→\s*(.+)$/i
);

// RESOLVE OPEN {N}: "desc" → any resolution text (numbered format)
const resolveNumberedMatch = normalized.match(
  /^- RESOLVE OPEN\s+\d+:\s*"?(.+?)"?\s*→\s*(.+)$/i
);
```

**Pattern priority (check in this order):**
1. `resolveMatch` — strict concept_id format (existing, keep as-is)
2. `resolveNumberedMatch` — numbered format: `RESOLVE OPEN 13: "desc" → text`
3. `resolveFlexMatch` — freeform: `RESOLVE OPEN: "desc" → resolution text`

For flex/numbered matches where there's no concept_id, set `conceptId: null` and store the resolution text in a new `resolution` field. The checklist modal can use this to display what the resolution was, even if it can't auto-match to a specific concept.

```javascript
// For freeform resolves, try to fuzzy-match against existing OPENs
updates.push({
  action: 'resolve',
  type: 'OPEN',
  description: matchedDescription.trim(),
  conceptId: null,  // No explicit concept_id
  resolution: matchedResolutionText.trim()  // Store for display
});
```

**In the checklist modal:** When `conceptId` is null for a RESOLVE item, show a dropdown of active OPENs on the linked Idea so the developer can select which OPEN to resolve. Pre-select using fuzzy word matching against existing OPEN descriptions (reuse `findDuplicateConcepts` logic).

---

### Bug 2: False Positive ODRC Detection on Non-ODRC Files

**Location:** `detectODRCContent()` at ~line 2760

**Problem:** The detection matched `CLAUDE-md-Phase3-OrphanODRC.md` which is a CLAUDE.md spec file that *mentions* ODRC concepts in its documentation — it's not an ODRC updates file. The parser then found unparseable lines like `- Type badge (OPEN, DECISION, RULE, CONSTRAINT)` and `- **ConceptManager.resolve():** Line ~4994 — marks an OPEN as resolved`.

The detection fires because the file contains 2+ lines matching the ODRC patterns:
```javascript
/^-\s+NEW\s+(DECISION|OPEN|RULE|CONSTRAINT):/m
```
But these appear inside code blocks or documentation, not as actual ODRC update commands.

**Fix:** Tighten the detection:

```javascript
function detectODRCContent(fileContent) {
  if (!fileContent || typeof fileContent !== 'string') return false;
  
  // Primary signal: the ## ODRC Updates header
  const hasODRCHeader = /^##\s+ODRC Updates/m.test(fileContent);
  
  // If the header exists, that's sufficient — this is an ODRC file
  if (hasODRCHeader) return true;
  
  // Secondary signal: 3+ ODRC action lines (raised from 2 to reduce false positives)
  // AND they must not be inside code blocks
  const codeBlockRanges = [];
  const codeBlockRegex = /```[\s\S]*?```/g;
  let match;
  while ((match = codeBlockRegex.exec(fileContent)) !== null) {
    codeBlockRanges.push([match.index, match.index + match[0].length]);
  }
  
  const isInCodeBlock = (lineIndex) => {
    const lineStart = fileContent.lastIndexOf('\n', lineIndex) + 1;
    return codeBlockRanges.some(([start, end]) => lineStart >= start && lineStart < end);
  };
  
  const odrcPatterns = [
    /^-\s+NEW\s+(DECISION|OPEN|RULE|CONSTRAINT):/m,
    /^-\s+RESOLVE\s+OPEN/m
  ];
  
  let realMatches = 0;
  for (const pattern of odrcPatterns) {
    const m = pattern.exec(fileContent);
    if (m && !isInCodeBlock(m.index)) realMatches++;
  }
  
  return realMatches >= 2;
}
```

Key changes:
- `## ODRC Updates` header is sufficient on its own (no false positive risk — this is our known header)
- Without header, require 3+ pattern matches (up from 2)
- Skip matches inside code blocks (``` fences)

---

### Bug 3: RESOLVE Items Need OPEN Picker in Checklist

**Location:** `ODRCImportChecklistModal` at ~line 12152

**Problem:** Currently the checklist shows RESOLVE items but if `conceptId` is null (which it always will be with the new flex parser), there's no way to select which OPEN to resolve. The user sees the description but can't link it to a specific concept.

**Fix:** For RESOLVE items where `conceptId` is null:
1. Query active OPENs scoped to the linked Idea
2. Fuzzy-match the RESOLVE description against OPEN content (reuse `findDuplicateConcepts`)
3. Show a dropdown of matching OPENs, pre-selecting the best match
4. Developer confirms or changes the selection

```
┌─────────────────────────────────────────────────────────────┐
│  ☑ RESOLVE OPEN                                             │
│    "Where do session docs live in the repo?"                │
│    Resolution: docs/sessions/{slug}/S-YYYY-MM-DD-NNN.md     │
│    Resolves: [Where do ideation session documents live ▾]    │
│              ↑ dropdown of active OPENs, best match first   │
└─────────────────────────────────────────────────────────────┘
```

---

### Gap 1: Generate CLAUDE.md Has No Phase Awareness

**Location:** `GenerateCLAUDEModal` at ~line 17832

**Problem:** The Generate CLAUDE.md button on App Aggregate view (Mode 2) assembles a CLAUDE.md from all active concepts regardless of idea maturity. An idea in "exploring" phase with 1 OPEN and 0 DECISIONs produces a CLAUDE.md that tells Code to build — skipping the entire Chat exploration cycle.

**Fix:** Add a phase warning:

```javascript
// In GenerateCLAUDEModal, check if any active ideas are still exploring
const exploringIdeas = appIdeas.filter(i => {
  if (i.status !== 'active') return false;
  const concepts = globalConcepts.filter(c => c.ideaOrigin === i.id);
  const phase = i.phase || computeIdeaPhase(concepts);
  return phase === 'exploring';
});

// If exploring ideas exist, show warning banner at top of modal
```

Display a warning banner inside the modal:
```
⚠️ {N} active idea(s) are still in "exploring" phase.
Consider using "Explore in Chat" to develop ideas further before generating a CLAUDE.md.
[Generate Anyway] [Cancel]
```

Don't block — just warn. The developer may have valid reasons to generate a CLAUDE.md early (quick fix, simple task). But they should know they're skipping the exploration cycle.

---

### Gap 2: Brief Generator Should Shift Framing at Spec-Ready

**Location:** `IdeationBriefGenerator.SYSTEM_PROMPT` at ~line 3387

**Problem:** The system prompt always says "generate a session brief for structured ideation conversation" regardless of phase. When an idea is `spec-ready`, the brief should shift from exploration framing to spec generation framing.

**Fix:** Make the system prompt phase-aware:

```javascript
getSystemPrompt(phase) {
  if (phase === 'spec-ready') {
    return `You are generating a session brief for a SPECIFICATION session.
The idea has reached spec-ready status — most OPENs are resolved, decisions are locked.
Generate a brief that instructs Chat to produce a formal specification from the accumulated ODRC state.
The spec should consolidate all DECISIONs, RULEs, and CONSTRAINTs into an implementable document.
Any remaining OPENs should be called out as items to resolve during the session.
Include the same ODRC Updates output format at the end.`;
  }
  
  // Default exploration prompt (existing)
  return `You are generating a session brief for a structured ideation conversation...`;
}
```

Update `generate()` to pass phase:
```javascript
async generate(idea, app, concepts, globalConcepts) {
  const phase = idea.phase || computeIdeaPhase(concepts);
  // ...
  const brief = await ClaudeAPIService.call({
    system: this.getSystemPrompt(phase),  // phase-aware
    // ...
  });
}
```

Also update the template fallback to reflect spec-ready framing.

---

## Existing Infrastructure Reference

| Component | Location | What to Change |
|-----------|----------|---------------|
| `ODRCUpdateIngestionService.parse()` | ~line 2683 | Add flex RESOLVE patterns |
| `detectODRCContent()` | ~line 2760 | Tighten detection, skip code blocks |
| `ODRCImportChecklistModal` | ~line 12152 | Add OPEN picker for RESOLVE items |
| `GenerateCLAUDEModal` | ~line 17832 | Add phase warning |
| `IdeationBriefGenerator` | ~line 3386 | Phase-aware system prompt |
| `findDuplicateConcepts()` | ~line 2794 | Reuse for RESOLVE-to-OPEN matching |

---

## Architecture Rules

### State Management Rules
- All shared Firebase-backed data lives as top-level state in App component with `global` prefix
- Firebase listeners are set up once in the App component's auth useEffect
- Views own local UI state only
- Write to Firebase via service methods, let listener update state

### Data Flow Rules
- Data flows down via props, events flow up via callbacks
- Service objects are global singletons (ConceptManager, IdeaManager, etc.)
- One listener per collection per user
- All listener useEffect blocks must return a cleanup function

---

## File Structure

```
cc/
  index.html                       ← All changes here
  specs/
    sp_ideation_bugfix_1.md        ← This CLAUDE.md archived after completion
```

---

## Post-Task Obligations

RULE: Before reporting this task as complete, execute this checklist:

1. Commit all code changes to the repo
2. Archive this CLAUDE.md to `cc/specs/sp_ideation_bugfix_1.md`
3. Generate a completion file to `.cc/completions/` per the format below
4. Commit the spec archive and completion file together in a separate commit

**Completion file naming:** `YYYY-MM-DDTHH-MM-SS_ideation-bugfix-1.md`

**Completion file format:**

```yaml
---
task: "Fix ODRC ingestion bugs: RESOLVE parser, false positive detection, OPEN picker, phase awareness"
status: complete | partial
cc-spec-id: sp_ideation_bugfix_1
files:
  - path: "cc/index.html"
    action: modified
commits:
  - sha: "{sha}"
    message: "{message}"
odrc:
  resolved_opens:
    - "{any resolved}"
  new_opens:
    - "{any new questions}"
unexpected_findings:
  - "{anything unexpected}"
unresolved:
  - "{anything not completed}"
---

## Approach

{Brief narrative of how fixes were applied}

## Implementation Notes

{Key details for validation}
```

Do not wait for the developer to ask. Generate the completion file automatically after committing code.
