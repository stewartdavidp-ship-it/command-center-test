---
task: "Fix satellite links — use absolute GitHub Pages URLs instead of relative paths"
status: complete
cc-spec-id: bugfix-satellite-urls
files:
  - path: "index.html"
    action: modified
commits:
  - sha: "071fb53"
    message: "Fix satellite links: use absolute GitHub Pages URLs instead of relative paths (v8.63.3)"
odrc:
  new_decisions:
    - "Satellite URLs always resolve to the production repo (command-center) — satellites do not exist in the test repo"
    - "getSatelliteUrl() added as a top-level utility function near getGitHubPagesUrl() — accepts optional apps param, falls back to window.location detection"
    - "Components without apps in scope (SettingsView, ProjectsTab) pass null — function handles gracefully via GitHub Pages hostname detection"
  new_opens: []
unexpected_findings:
    - "SettingsView and ProjectsTab do not have 'apps' in scope — calling getSatelliteUrl(apps, ...) caused ReferenceError that crashed the entire Settings view (blank page, 5 test failures)"
    - "The satellite launcher dropdown used window.location.pathname to build relative URLs — this resolved to file:// paths when running locally"
unresolved: []
---

## Approach

Added `getSatelliteUrl(apps, subPath)` helper function and replaced all 5 hardcoded relative satellite links.

## Implementation Notes

**Root cause:** All satellite links used relative paths (`infrastructure/`, `quality/`, `analytics/`) which:
- On local file:// — browsers block navigation to local file paths
- On hosted test repo (command-center-test) — satellites don't exist there, only in production repo

**getSatelliteUrl() resolution order:**
1. If `apps` provided and CC app has prodRepo → use `getGitHubPagesUrl(prodRepo, subPath)`
2. If on GitHub Pages (hostname ends with .github.io) → hardcode production repo path
3. Fallback → relative path (works when already on correct GitHub Pages base)

**5 link locations updated:**
- Satellite launcher dropdown (line ~8663)
- Repo health alert infrastructure link (line ~8893)
- Analytics product brief button in ProjectsTab (uses null)
- Firebase connection test link in SettingsView (uses null)
- Domain management link in SettingsView (uses null)
