---
task: "Add projectId to Idea data model + one-off migration to create 11 Ideas and import 119 ODRC concepts from consolidated session documents"
status: complete
cc-spec-id: migration_odrc_batch_import
files:
  - path: "index.html"
    action: modified
commits:
  - sha: "343661e"
    message: "Add projectId to Idea data model + ODRC batch import migration (v8.64.1)"
odrc:
  new_decisions:
    - "projectId added to Idea schema — lightweight grouping that persists through graduation"
    - "11 Ideas established under command-center project covering workflow model through stress test"
  resolved_opens:
    - "Bulk import of ODRC concepts from existing documents — needed for seeding, deferred from Phase 1"
  new_opens:
    - "How does projectId relate to the existing config.projects structure? Should they merge?"
    - "Project as first-class object — UI for creating, viewing, and managing Projects"
unexpected_findings:
  - "Spec said 125 concepts but actual count in data is 119. The 11 idea keys produce: workflow-model 11, completion-files 7, ingestion-pipeline 14, validation-bundle 16, orphan-detection 5, spec-packaging 5, test-infrastructure 8, task-tracking 7, ideation-platform 8, architecture-rules 16, stress-test 22. Includes 2 cross-cutting OPENs from Repo-Aware Artifact Placement routed to workflow-model."
  - "Architecture-rules comment in spec said 14 but data contained 16 entries (1 DECISION + 11 RULEs + 4 OPENs). Fixed comment to match actual count."
unresolved:
  - "Migration function not yet executed — awaiting developer to run runODRCBatchImport() in console and verify"
  - "Migration function must be removed after successful execution"
---

## Approach

Two-part build: (1) Added `projectId` field to the Idea data model across create, update, and backfill paths. (2) Placed a temporary console-callable migration function `runODRCBatchImport()` after the IdeaManager object that creates 11 Ideas and imports 119 ODRC concepts using existing service methods (IdeaManager.create, ConceptManager.create, IdeaManager.addSessionLogEntry).

## Implementation Notes

- **P1**: `IdeaManager.create()` now accepts `projectId` param (default null), writes it to the idea object
- **P2**: `IdeaManager.update()` allowed fields extended with `'projectId'`
- **P3**: `backfillMissingFields()` infers projectId from appId for existing ideas (appId IS projectId for CC apps; smarter inference can come later)
- **M1**: Migration function uses 3-way idea matching (slug, generated slug from name, case-insensitive name) to find or create ideas. Warns on existing concepts (3-second delay). Adds session log entries with migration type. Exposed on `window.runODRCBatchImport` for console access.
- All 34 Playwright tests pass
- Version bumped to 8.64.1
