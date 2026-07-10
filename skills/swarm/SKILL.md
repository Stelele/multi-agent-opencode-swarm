---
name: swarm
description: Use when the user says "swarm:", "/swarm", "use swarm", "spawn agents", or when a task spans >3 files across multiple project layers (frontend+backend), is a refactor, or has multiple independent work units. This skill implements multi-agent orchestration for cost efficiency — the expensive model only coordinates while cheap swarm workers do the bulk work.
---

# Swarm: Multi-Agent Cost-Optimized Orchestration

## Overview

You are an orchestrator. Your job is to route work to cheap `swarm-worker` agents (deepseek-v4-flash) and verify their output with `swarm-verifier` agents (also cheap). You only do coordination, context reading, and final aggregation. You do NOT do the bulk coding yourself unless the escalation ladder forces it.

## When You MUST Use This

1. **Explicit invocation:** User says "swarm:", "/swarm", "use swarm", or "spawn agents"
2. **Auto-trigger:** Task touches >3 files across >1 project layer (e.g., frontend + backend), is a "refactor", "rename across", "move all", "full stack", or "end to end" task, or the user asks for multiple independent units of work in one request

<HARD-GATE>
Before auto-triggering, ALWAYS ask the user first. Do NOT silently dispatch workers.
</HARD-GATE>

## The Workflow

### Phase 1: Context Extraction

Before proposing anything, read:

1. **AGENTS.md files** — at project root, `frontend/AGENTS.md`, `backend/AGENTS.md`, etc. These define tech stack, conventions, and rules.
2. **Project file tree** — understand the structure to scope worker assignments
3. **Git diff** — what's changed recently?
4. **The user's task description** — parse into discrete work units

### Phase 2: Spawn Proposal

Present to the user:

```
SWARM PROPOSAL
───────────────

Task: [restated task]

Workers:
  worker-1: [scope — which files/directories, what to do]
  worker-2: [scope]
  ...

Contracts:
  worker-1 exports: [what it produces that others might need]
  worker-2 expects: [what it needs from others]
  ...

Proceed? Adjust by saying which workers to add/remove/modify.
```

**Rules for scoping:**
- Each worker gets 1-5 files maximum
- Each worker handles ONE project layer (frontend OR backend OR database, not mixed)
- If a task naturally spans layers, split into one worker per layer
- Never assign more than one worker to the same file

**Rules for contracts:**
- Define what each worker will CREATE or MODIFY (exports)
- Define what each worker NEEDS from others (imports)
- Contract format: `POST /api/endpoint`, `function auth.login()`, `table MfaEnrollment`, `component <MfaSetup>`

**Rules for number of workers:**
- 1-2 independent work units → 2 workers
- 3-5 independent work units → 3-5 workers
- 6+ → limit to 6 max (split remainder into a second round)

### Phase 3: Refinement

Let the user adjust the proposal. They may:
- Merge or split workers
- Change worker scopes
- Add specific instructions to a worker
- Add/remove workers
- Adjust contracts

Re-present the modified proposal until the user approves.

### Phase 4: Dispatch

After approval:

1. Dispatch ALL workers in PARALLEL via the Task tool
2. Use agent type: `swarm-worker`
3. Each worker's prompt includes:
   - The relevant AGENTS.md fragment (only the parts for their layer)
   - Their specific task scope
   - Their output contract
   - Instruction: "Follow these instructions precisely. Do not exceed your scope. Return: (1) files changed, (2) summary, (3) warnings."

```
Dispatch format:
  Task tool, subagent_type="swarm-worker"
  description="[role] - [scope summary]"
  prompt="[AGENTS.md fragment if relevant]
Task: [specific instructions]
Files: [list of files to work on]
Output contract: [what you must produce]
Return format: 1) files changed 2) summary 3) warnings"
```

Wait for ALL workers to complete before proceeding to verification.

### Phase 5: Verification

After all workers complete:

1. **Per-worker verification** — dispatch one verifier PER worker (parallel):

```
Task tool, subagent_type="swarm-verifier"
description="verify worker-[N]"
prompt="Verify this worker's output:

TASK ASSIGNED:
[worker's original task scope]

WORKER OUTPUT:
[paste worker's complete output]

Check:
1. Does the output implement what was asked?
2. Any references to files/functions/types that don't exist?
3. Obvious bugs or incomplete work?

Return: PASS or FAIL - [reason]"
```

2. **Contract cross-check** — dispatch ONE verifier for contract alignment:

```
Task tool, subagent_type="swarm-verifier"
description="verify contracts"
prompt="Cross-check contracts across workers:

CONTRACTS:
worker-1 exports: [what it produced]
worker-2 expects: [what it needs]
...

WORKER OUTPUTS:
worker-1: [summary of changes]
worker-2: [summary of changes]
...

Check: Do the exports/imports align? Is worker-2 looking for something worker-1 didn't produce? Are there name mismatches?

Return: PASS or FAIL - [specific mismatches]"
```

<HARD-GATE>
Wait for ALL verifiers to complete. Do NOT proceed to aggregation with pending verifications.
</HARD-GATE>

### Phase 6: Failure Escalation

If any verifier returns FAIL:

| Attempt | Action | Agent | Notes |
|---------|--------|-------|-------|
| 1st fail | Re-dispatch the SAME worker with the verifier's feedback | swarm-worker (v4-flash) | Include the specific FAIL reason |
| 2nd fail | Re-dispatch with BROADER context (more files, more AGENTS.md) | swarm-worker (v4-flash) | Give the worker more context to work with |
| 3rd fail | YOU re-scope the task into smaller pieces | You (v4-pro) | Split the failing task and re-dispatch as 2+ workers |
| Critical fail | YOU take over directly | You (v4-pro) | Only for: code corruption, same error class repeating, verifier catching structural issues |

After re-dispatch, re-run verification. Loop until all workers PASS or you reach Level 3/4.

<EXTREMELY-IMPORTANT>
NEVER re-dispatch a worker more than twice without re-scoping. The cheap model can't solve everything.
</EXTREMELY-IMPORTANT>

### Phase 7: Aggregation

Once all verifiers return PASS:

Present to the user:

```
SWARM COMPLETE
──────────────

Files changed: [list all files across all workers]
Summary: [2-3 sentence summary of all changes]
Verification: All workers passed. Contracts aligned.

Worker performance:
  worker-1: PASS on [first/second] attempt — [files changed]
  worker-2: PASS on first attempt — [files changed]
  ...

Total workers: N | Total cost model: [what model was used for what]
```

If multiple rounds were needed, mention: "worker-3 required 2 attempts. Consider scoping similar tasks smaller in the future."

## Anti-Patterns — DO NOT DO

| Excuse | Reality |
|--------|---------|
| "It's faster if I just do it myself" | You're the expensive model. Doing the work yourself defeats the purpose. |
| "This task is too complex for flash" | Split it smaller. Decompose until each piece fits. |
| "Let me dispatch workers without proposing first" | User MUST approve the spawn architecture. No surprises. |
| "I'll aggregate while verifiers are still running" | Wait for ALL verifiers. Do not present partial results. |
| "The worker failed, let me just fix it directly" | Follow the escalation ladder. Re-dispatch first. |

## Red Flags

- Dispatching workers without presenting the proposal → STOP
- Writing code directly instead of dispatching → STOP (unless escalation level 4)
- Aggregating before verification completes → STOP
- Re-dispatching the same worker more than twice without changing approach → STOP
- Not reading AGENTS.md before scoping workers → STOP

## Context Efficiency

- Only send the relevant AGENTS.md FRAGMENT to each worker, not the whole file
- Workers only get file paths they need, not the entire tree
- Keep worker prompts concise — less context = cheaper
- Verifier prompts are even smaller — just the task + output
