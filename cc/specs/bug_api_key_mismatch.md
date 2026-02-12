# CLAUDE.md — Bug Fix: ClaudeAPIService Uses Wrong API Key
# cc-spec-id: bug_api_key_mismatch
# App: Command Center (index.html)
# Base version: 8.59.0 (or current)
# Task type: bug_fix
# Priority: medium

---

## Bug

ClaudeAPIService (added in v8.59.0 / Phase 2) reads its API key from `localStorage.getItem('cc_api_key')`. This key is actually a CC-generated random token used for the internal deploy queue authentication between Claude and CC. It is NOT an Anthropic API key.

When the user clicks "Package for Check" in Job History, the bundle assembler calls ClaudeAPIService to generate a review prompt. The call fails because `cc_api_key` contains something like `cc_a8f3k2m9x...` instead of `sk-ant-api03-...`.

**Root cause:** Phase 2 spec incorrectly identified `cc_api_key` as the Anthropic API key. The existing `generateApiKey()` function in SessionLogView (line ~11687) generates a random internal token and stores it under that same key name.

## Fix

### 1. New localStorage key for Anthropic API key

Use `cc_anthropic_api_key` for the real Anthropic API key. Leave `cc_api_key` untouched — it still serves the deploy queue.

### 2. Update ClaudeAPIService

Change `getApiKey()` to read from the new key:

```javascript
getApiKey() {
    try { return localStorage.getItem('cc_anthropic_api_key') || ''; } catch { return ''; }
}
```

### 3. Add Anthropic API Key input to Settings view

Add a new section in the Settings view (near the existing AI Engines section) with:

- Section title: "🔑 Anthropic API Key"
- Description: "Required for AI-generated review prompts in the validation bundle pipeline. Get your key from console.anthropic.com."
- Password-style input field (type="password") showing masked key, with show/hide toggle
- Save button that writes to `localStorage.setItem('cc_anthropic_api_key', key)`
- Status indicator: "✅ Configured" or "⚠️ Not set — review prompt generation will use static fallback"
- Clear button to remove the stored key

### 4. Update StorageManager protected keys

Add `cc_anthropic_api_key` to the `PROTECTED_KEYS` array in StorageManager (line ~875) so it doesn't get cleaned up during storage maintenance.

## Files Likely Affected

- `index.html` — ClaudeAPIService.getApiKey(), SettingsView, StorageManager.PROTECTED_KEYS

## Post-Task Obligations

RULE: Before reporting this task as complete, execute this checklist:

1. Commit all code changes
2. Archive this CLAUDE.md to `cc/specs/bug_api_key_mismatch.md`
3. Generate a completion file to `cc/completions/` with these frontmatter fields:
   ```yaml
   task: "Fix ClaudeAPIService to use dedicated Anthropic API key instead of internal CC token"
   task_type: bug_fix
   status: complete
   tracking:
     type: bug_fix
     source_spec: bug_api_key_mismatch
     priority: medium
   files: [...]
   commits: [...]
   ```
4. Commit spec archive and completion file together
