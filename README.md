# Multi-Agent OpenCode Swarm

A skill and agent configuration for OpenCode that implements cost-efficient multi-agent orchestration. The expensive coordinator model (v4-pro) only does planning and verification, while a swarm of cheap worker agents (v4-flash) does the bulk coding.

Inspired by: Claude Fable 5 Bossed 20 Cheap AI Agents (Nate B Jones)

## How It Works

```
/swarm "add MFA to auth module"
  → Planner extracts project context (AGENTS.md, file tree, git diff)
  → Proposes spawn architecture (N workers, their scopes, contracts)
  → You refine and approve
  → Workers run in parallel (v4-flash)
  → Verifiers check each worker's output (v4-flash)
  → Contract cross-checker validates alignment
  → Failed workers retry with feedback (escalation ladder)
  → Builder aggregates verified results
```

**Cost model:** 90%+ of tokens burn on the cheapest available model. The expensive model only orchestrates.

## Installation

### 1. Add the Skill

Clone or copy the `skills/swarm/` directory to your OpenCode skills path:

```bash
# Option A: Personal (all projects)
cp -r skills/swarm ~/.config/opencode/skills/swarm

# Option B: Project-specific
cp -r skills/swarm .opencode/skills/swarm
```

### 2. Add the Agent Definitions

Add the `swarm-worker` and `swarm-verifier` agents to your `opencode.json`:

```json
{
  "agent": {
    "general": { "model": "opencode-go/deepseek-v4-flash" },
    "explore": { "model": "opencode-go/deepseek-v4-flash" },
    "swarm-worker": {
      "description": "Cheap worker agent for executing a single scoped coding task.",
      "mode": "subagent",
      "model": "opencode-go/deepseek-v4-flash",
      "hidden": true,
      "steps": 30,
      "prompt": "You are a focused worker agent in a swarm. You are given ONE scoped task to complete. Follow the instructions precisely. Work only on files assigned to you. Do not expand your scope beyond what was assigned. When done, return: (1) list of files changed, (2) summary of what you did, (3) any warnings or issues you noticed. Be concise."
    },
    "swarm-verifier": {
      "description": "Verifies worker output against assigned tasks.",
      "mode": "subagent",
      "model": "opencode-go/deepseek-v4-flash",
      "hidden": true,
      "steps": 15,
      "prompt": "You are a verification agent. Your job is to catch errors, hallucinations, and inconsistencies. Given a worker's output and its assigned task, check: (1) Does the output actually implement what was asked? (2) Are there any references to files, functions, or types that don't exist? (3) Are there obvious bugs or incomplete work? (4) For contract verification: do the declared inputs/outputs match across workers? Return EXACTLY: 'PASS' or 'FAIL - [specific reason]'. Be thorough and skeptical."
    }
  }
}
```

A full example agent config is in `config/agent-config.json`. Add these entries inside your existing `"agent"` object — don't replace your existing agents.

### 3. Restart OpenCode

The skill and agents are loaded at startup.

## Usage

### Explicit Invocation

```
swarm: refactor the auth module to support MFA
```
```
/swarm add end-to-end tests for the payment flow
```
```
use swarm to update the login component and the user API
```

### Auto-Trigger

The swarm auto-activates when a task is complex enough:
- Touches >3 files across >1 project layer (frontend + backend)
- Is a "refactor", "rename across", or "move all"
- Has multiple independent work units ("Add MFA and also update login")
- Is "full stack" or "end to end"

You'll always be asked before workers are dispatched — no surprises.

### The Refinement Loop

Before any work starts, you'll see a spawn proposal:

```
SWARM PROPOSAL
───────────────

Task: Add MFA to auth module

Workers:
  worker-1: backend/Auth/ — new MediatR commands, FluentValidation rules
  worker-2: frontend/ — MFA setup flow UI with components
  worker-3: DB migrations — new MFA tables in EF Core

Contracts:
  worker-1 exports: POST /api/auth/mfa/enroll, POST /api/auth/mfa/verify
  worker-2 expects: endpoints with shape {method, secret, code}
  worker-3 exports: MfaEnrollment table schema

Proceed? Adjust by saying which workers to add/remove/modify.
```

You can merge workers, split them, change scopes, add instructions — iterate until the plan looks right.

## Prerequisites

- **OpenCode** with the OpenCode Go subscription
- **AGENTS.md files** recommended — the planner reads these to scope workers correctly
- The `swarm-worker` and `swarm-verifier` agents configured in your `opencode.json`

## Failure Handling

The swarm has a 4-level escalation ladder to handle failures without burning expensive tokens:

| Level | Action | Model |
|-------|--------|-------|
| 1 | Retry same worker with verifier feedback | v4-flash |
| 2 | Retry with broader context | v4-flash |
| 3 | Planner re-scopes into smaller sub-tasks | v4-pro |
| 4 | Planner takes over directly | v4-pro |

The expensive model is always the last resort.

## Files

```
skills/swarm/
  SKILL.md              # The swarm orchestration skill

config/
  agent-config.json     # Agent definitions for opencode.json
```

## Tips

- **Write good AGENTS.md files.** The planner uses them to scope workers and give them correct context.
- **Define contracts clearly.** Contract mismatches are the most common failure mode across parallel workers.
- **Start with 2-3 workers.** More workers = more coordination overhead. Find the sweet spot.
- **Per-worker verifiers are cheap.** They only see one worker's output, not the whole project.
- **The verifier catches lies.** Cheap models sometimes hallucinate. The verifier's job is to catch that.
