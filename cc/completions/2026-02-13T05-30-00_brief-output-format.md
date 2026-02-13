---
task: "Enrich IdeationBriefGenerator SYSTEM_PROMPT and Expected Output Format — add 6-line header block, all five ODRC line types with examples, and Session Notes section template"
status: complete
cc-spec-id: sp_brief_output_format
files:
  - path: "index.html"
    action: modified
commits:
  - sha: "a015c6e"
    message: "Enrich IdeationBriefGenerator output format — 6-line header, all ODRC types, Session Notes (v8.63.4)"
odrc:
  new_decisions:
    - "Only updated the exploration (default) SYSTEM_PROMPT case as specified — spec and claude-md cases left unchanged"
    - "Template fallback enriched with identical format to ensure consistency when AI generation fails"
  new_opens: []
unexpected_findings:
    - "The spec and claude-md SYSTEM_PROMPT cases also have terse Expected Output Format instructions — could be enriched in a follow-up"
unresolved: []
---

## Approach

Pure prompt/template update — no new features, services, or UI changes. Two edit points:

1. **B1:** Updated the `exploration` default case in `getSystemPrompt()` — replaced the terse 4-line Expected Output Format instruction with the full specification including 6-line header block, all 5 ODRC line types with rationale examples, and Session Notes section
2. **B2:** Updated the static template fallback in `buildTemplateBrief()` — enriched the Expected Output Format code block from 3 header lines + 3 line types to 6 header lines + 5 line types + Session Notes section template

## Implementation Notes

**SYSTEM_PROMPT change (B1):**
- Replaced lines 3074-3079 (terse format) with 10-line enriched instruction
- The instruction tells the AI to show (a) 6-line header, (b) all 5 line types with substantive rationale examples, (c) Session Notes with 3 subsections
- Uses "Show the complete format as a copyable template" to ensure the AI produces the full format

**Template fallback change (B2):**
- Header block now includes: title with idea name, date, session with description prompt, idea slug, app ID, context summary prompt
- All 5 ODRC line types: RESOLVE OPEN, NEW DECISION, NEW OPEN, NEW RULE, NEW CONSTRAINT — each with guidance on what good descriptions look like
- Session Notes section: What Was Accomplished, Key Design Principles Established, Session Status
- Adds instruction "Each ODRC line should include substantive rationale — not just a label"

**Compatibility verified:**
- ODRCUpdateIngestionService already parses all 5 line types (NEW DECISION, NEW OPEN, NEW RULE, NEW CONSTRAINT, RESOLVE OPEN)
- Header metadata lines (# Idea:, # Session:, # App:) unchanged
- New lines (# Date:, # Context:) are informational — ingestion service ignores unknown # lines
- getHandshakePrompt() unchanged as specified
