# CC Streams Evolution — Session 2 Handoff

## Starting Point
- **Source zip:** cc_streams_evo_s1.zip (drop into CC to deploy, or extract to work on)
- **CC Core version:** 8.55.2
- **Satellite versions:** Analytics 1.0.1, Infrastructure 1.0.1, Quality 1.0.1
- **Skills:** Read /mnt/skills/user/ skills first (gs-active, firebase-patterns, ui-components, session-continuity)
- **Transcripts:** Check /mnt/transcripts/ for full session 1 context

## Session 1 Completed ✅
1. **Data Model Extension** (analytics satellite) — WorkStreamService extended with appIds[], project, concepts[], artifacts[], sessions[], tests{}, codeReview{}, openItems[], next. StreamEditModal updated. Stream cards show badges.
2. **Concept Parser** (CC core) — `parseConcepts()` extracts typed concepts from `<!-- cc-concepts -->` markers. `checkPushedDocsForConcepts()` runs after Deploy All and Session Wizard doc pushes, prompts user to attach to a stream.
3. **Bug fixes during testing:**
   - `updateApp` passed as prop to DashboardView (was out of scope)
   - `primaryAppId` fallback for shared/doc-only batches (was resolving to `_shared`)
   - Deploy All button now has spinner + try/catch/finally
   - Doc file validation skips API calls (only shared files checked)
   - Info-level issues collapsed into expandable `<details>` line

## Session 2 Goal: Workstream Brief Generator

### Chunk 3: Brief Generator (CC Core)
Per the spec in `docs/STREAMS_EVOLUTION_PROJECT.md`:

- [ ] New function: `generateWorkstreamBrief(stream, apps, versions)` → markdown string
- [ ] Include: identity, active concepts, open items, test status, code review status, last session, next
- [ ] Follow pattern of existing SessionBriefGenerator / ProductBriefGenerator  
- [ ] Add "Copy Brief" button to stream detail in Analytics satellite
- [ ] Add brief preview panel (show what will be copied)
- [ ] Test: generate brief for a stream with concepts, verify output

### Brief Template
```markdown
# Stream: {name}
Status: {status} | Project: {project} | Apps: {appIds}

## Active Concepts
- RULE: ...
- DECISION: ...
- CONSTRAINT: ...

## Open Items
- ...

## Test Status: N/M pass, N FAIL, N untested

## Code Review: {status}

## Last Session ({date})
{summary}

## Next
{next}
```

### Files to Modify
- `command-center/index.html` — generateWorkstreamBrief function (~100 lines)
- `analytics/index.html` — Copy Brief button + preview (~50 lines)

## Known Issues (not blocking S2)
- Deploy All sends satellites to prod directly (they only have prod repos configured)
- Verification shows `expected version: "batch"` for shared/doc-only batches (cosmetic)
- Content-hash comparison to skip unchanged docs — design agreed, implementation deferred

## How to Start
1. Read skills in /mnt/skills/user/
2. Review /mnt/transcripts/ for session 1 context
3. Extract the zip, find the existing generators as pattern reference
4. Build the brief generator, test with a stream that has concepts
