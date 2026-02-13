---
task: "Fix ODRC auto-link cascade: reject placeholder slug markers, sort fallback by recency"
status: complete
cc-spec-id: bugfix-odrc-autolink-placeholder
files:
  - path: "index.html"
    action: modified
commits:
  - sha: "a509fa3"
    message: "Fix ODRC auto-link: reject placeholder slugs, sort fallback by recency (v8.64.2)"
odrc:
  new_decisions: []
  resolved_opens: []
  new_opens:
    - "Should extractODRCMetadata() also validate IdeaId format (Firebase push key pattern)?"
    - "When slug is rejected as placeholder, should CC prompt user to create a new Idea inline?"
unexpected_findings:
  - "Claude session output inserts '(new — needs creation)' as Idea header when no existing idea matches — this is a valid signal that should trigger new idea creation, not slug lookup"
  - "App-scope fallback was picking ideas by array position, not by recency — with 11+ ideas per app this became random"
unresolved: []
---

## Approach

Two fixes in extractODRCMetadata() and the auto-link cascade:

1. **Slug validation**: Added checks for parenthesized markers, em-dashes, ellipsis, common placeholder words (new/none/tbd/todo/create), and strings with no alphanumeric characters. Invalid slugs return null, bypassing Priority 2 entirely.

2. **Fallback sorting**: App-scope fallback (Priority 3) now sorts candidates by sequence descending then createdAt descending, picking the most recent active idea instead of the last element by array position.

## Implementation Notes

- Validation is intentionally broad — false positives (valid slugs rejected) will fall through to Priority 3 which still works correctly
- The `(new — needs creation)` marker from Claude could be used as a signal to auto-create ideas in a future enhancement
- All 34 Playwright tests pass
