---
task: "Fix ClaudeAPIService to use dedicated Anthropic API key instead of internal CC token"
task_type: bug_fix
status: complete
tracking:
  type: bug_fix
  source_spec: bug_api_key_mismatch
  priority: medium
files:
  - path: "index.html"
    action: modified
  - path: "cc/specs/bug_api_key_mismatch.md"
    action: created
commits:
  - sha: "794db76"
    message: "Fix ClaudeAPIService API key mismatch — use cc_anthropic_api_key (v8.60.1)"
  - sha: "0375c02"
    message: "Archive bug spec and generate completion file"
odrc:
  resolved_opens:
    - "ClaudeAPIService uses wrong API key: cc_api_key is an internal CC token, not an Anthropic API key — Now reads from cc_anthropic_api_key"
  applied_decisions:
    - "API keys stay in localStorage, not Firebase — Anthropic key stored as cc_anthropic_api_key in localStorage only, never sent to Firebase"
---

## Approach

Four-point fix as specified in the bug report:

1. **ClaudeAPIService.getApiKey()** — Changed from `localStorage.getItem('cc_api_key')` to `localStorage.getItem('cc_anthropic_api_key')`. Updated error message to direct users to the new Settings section.

2. **StorageManager.PROTECTED_KEYS** — Added `cc_anthropic_api_key` to the protected keys array so it survives storage maintenance.

3. **Settings view — Anthropic API Key section** — Added new section before Completion Files with: password input (show/hide toggle), Save button, Clear button, status indicator (configured vs. not set), link to console.anthropic.com.

4. **Error messages** — Updated all "API key not configured" alerts to reference the specific Settings path.

## Notes

- The existing `cc_api_key` / `gs_api_key` keys remain untouched — they continue to serve the internal deploy queue authentication.
- Version bumped to 8.60.1 (patch for bug fix).
