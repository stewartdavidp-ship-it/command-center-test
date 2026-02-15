---
task: "Fix deploy button stuck in spinning state after doc-only pushes"
status: complete
cc-spec-id: null
files:
  - path: "index.html"
    action: modified
commits:
  - sha: "dcac637"
    message: "Fix deploy button stuck in spinning state after doc-only pushes (v8.69.1)"
odrc:
  new_decisions: []
  resolved_opens: []
  new_opens: []
unexpected_findings:
  - "The isDeploying state had two dead-end paths: (1) doc-only pushes never create an activeDeployment, so the effect that resets isDeploying never fires; (2) when activeDeployment is auto-closed after 4s, it becomes null, but the effect had no else branch to handle null."
unresolved: []
---

## Approach

Traced the deploy button state machine from user symptom (button stuck spinning) through the state management code. Found that `isDeploying` is set to `true` when Deploy is clicked but only reset by the `activeDeployment` watcher effect. For doc-only pushes (like .md files classified as `push-doc`), no `activeDeployment` is ever created, so `isDeploying` stays `true` forever.

## Implementation Notes

Two fixes applied:
1. Added `else` branch in the `activeDeployment` effect (line ~10962) to call `setIsDeploying(false)` when `activeDeployment` is `null` — handles post-auto-close and doc-only scenarios
2. Added `isDeploying` reset in the `finally` block (line ~12102) specifically for doc-only pushes (`selectedDeployFiles.length === 0`) — provides immediate feedback without waiting for effect cycle
