---
task: "Redesign Idea History from flex-wrap button grid to clean vertical list layout"
status: complete
cc-spec-id: null
files:
  - path: "index.html"
    action: modified
commits: []
odrc:
  new_decisions:
    - "Idea History uses vertical list with one row per idea, aligned columns for sequence, name, status, phase, concept count, and action button"
    - "Status-based left border accent: indigo=active, green=graduated, slate=archived"
  resolved_opens: []
  new_opens: []
unexpected_findings: []
unresolved: []
---

## Approach

Replaced the `flex gap-2 flex-wrap` container (which produced a chaotic 2-column blob of variable-width button cards) with a `space-y-1` vertical list. Each idea is a single full-width row with aligned data columns.

## Implementation

Replaced lines 16486-16527 in IdeasView Mode 2 (App Aggregate). Each row shows: sequence number, idea name (truncated), status badge, phase pill, active concept count, and action button (active ideas only). Rows are clickable to navigate to idea detail. Action button uses `stopPropagation` to prevent row click.

## Verification

- 34/34 Playwright tests pass
- Version bumped to 8.65.3
- No Babel errors in headless browser
