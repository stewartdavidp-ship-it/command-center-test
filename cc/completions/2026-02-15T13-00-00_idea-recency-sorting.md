---
task: "Fix ideas not appearing on home page due to weak recency sorting"
status: complete
cc-spec-id: null
files:
  - path: "index.html"
    action: modified
commits:
  - sha: "26dc68f"
    message: "Fix ideas not appearing on home page due to weak recency sorting (v8.69.3)"
odrc:
  new_decisions:
    - "getRecencyTimestamp() considers all available signals: lastSessionDate, sessionLog entries, activeSession timestamps, updatedAt, createdAt — returns the most recent"
    - "activateSession and completeSession now bump updatedAt on the idea root so ideas with lifecycle activity always sort as recent"
  resolved_opens: []
  new_opens: []
unexpected_findings:
  - "activateSession only wrote to the activeSession child node, not the idea root — updatedAt was never bumped, causing ideas with active sessions to sort by their original creation date"
  - "lastSessionDate is only set via addSessionLogEntry (ODRC import path), so ideas worked on outside the ODRC pipeline had permanently stale sort timestamps"
unresolved: []
---

## Root Cause

The "Argument game" idea wasn't appearing on the home page because `getRecentIdeas` sorted by `lastSessionDate || updatedAt || createdAt`. Two problems:

1. `lastSessionDate` is only set during `addSessionLogEntry()` (ODRC import). If sessions were started via ExploreInChatModal but results never imported, `lastSessionDate` stayed `null`.

2. `activateSession()` wrote to `activeSession` child only — it never bumped `updatedAt` on the idea root. So `updatedAt` stayed at the idea's creation timestamp.

With the home page limited to 5 recent ideas (`maxCount = 5`), "Argument game" sorted behind ideas that happened to have more recent creation dates.

## Fix

1. **`getRecencyTimestamp()`**: New function that gathers timestamps from all sources (lastSessionDate, sessionLog entries, activeSession timestamps, updatedAt, createdAt) and returns `Math.max()` of all valid candidates.

2. **`activateSession()`**: Now uses `ideaRef.update()` to set both `activeSession` and `updatedAt` atomically.

3. **`completeSession()`**: Now uses `ideaRef.update()` to clear `activeSession` and bump `updatedAt` atomically.
