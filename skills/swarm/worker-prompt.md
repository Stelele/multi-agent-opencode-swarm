# Swarm Worker Prompt Template

Use this template when dispatching a `swarm-worker` subagent.

**Purpose:** Execute a single scoped coding task with precise instructions and a clear output contract.

```
Task tool (swarm-worker):
  description: "[role] - [scope summary]"
  prompt: |
    <SUBAGENT-STOP>
    You are a swarm worker. Do NOT try to orchestrate or spawn other agents.
    Execute ONLY the task assigned to you below.
    </SUBAGENT-STOP>

    ## Context

    [Relevant AGENTS.md fragment — ONLY the section for this layer. 
     e.g., for a frontend worker: conventions, import ordering, naming rules.
     Strip out backend/database sections.]

    ## Task

    [Specific instructions — what to build, modify, or fix. 
     Be precise about expected behavior.]

    ## Files

    [List of files this worker may touch. Workers must NOT touch files
     outside this list unless explicitly stated.]

    ## Output Contract

    You must produce:
    [Declare what this worker will CREATE or EXPOSE:
     e.g., POST /api/auth/mfa/enroll
           component <MfaSetup>
           function validateMfaToken()
           table MfaEnrollments]

    ## Instructions

    1. Follow these instructions precisely
    2. Do not exceed your assigned scope
    3. Do not modify files outside the Files list
    4. Follow existing codebase patterns and conventions
    5. [Optional: Follow TDD — write failing tests first]
    6. [Optional: Run existing tests to verify no regressions]

    ## Return Format

    When done, return EXACTLY:
    1) Files changed: [list with paths]
    2) Summary: [what you did, what the code does]
    3) Warnings: [any issues, edge cases not handled, assumptions made]

    Be concise. Do not narrate your process. Just the results.
```

## Usage Notes

- **AGENTS.md fragments:** Only include the section relevant to this worker's layer. For a frontend worker, include Vue conventions, import ordering, naming rules. Skip backend, database, and deployment sections.
- **File lists:** Never assign more than 5 files to one worker. If the task needs more files, split it into multiple workers.
- **Contracts:** Each worker must declare what it produces (exports) and what it needs from others (imports). This feeds into the contract cross-checker.
- **TDD flag:** Include "Follow TDD: write failing tests first" only when appropriate. Skip for config changes, schema updates, or purely structural refactors.
- **Scope discipline:** A worker that exceeds its scope creates merge conflicts and breaks contract verification. The prompt must make boundaries explicit.
