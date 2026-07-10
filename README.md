# Multi-Agent OpenCode Swarm

A skill suite for OpenCode that implements cost-efficient multi-agent orchestration. The expensive coordinator model (v4-pro) only does planning and verification, while a swarm of cheap worker agents (v4-flash) does the bulk coding.

Inspired by: Claude Fable 5 Bossed 20 Cheap AI Agents (Nate B Jones)

## How It Works

```
/swarm "add MFA to auth module"
  → Planner extracts project context (AGENTS.md, file tree, git diff)
  → Proposes spawn architecture (N workers, their scopes, contracts)
  → You refine and approve
  → Workers run in parallel (v4-flash)
  → Per-worker verifiers check each output (v4-flash)
  → Contract cross-checker validates alignment (v4-flash)
  → Failed workers retry with feedback (escalation ladder)
  → Builder aggregates verified results
```

**Cost model:** 90%+ of tokens burn on the cheapest available model. The expensive model only orchestrates.

## File Structure

```
skills/
├── using-swarm/
│   └── SKILL.md              # Meta-skill — rules of engagement, when swarm MUST be used
└── swarm/
    ├── SKILL.md              # Orchestration workflow (7 phases, DOT flowcharts)
    ├── worker-prompt.md      # Template for dispatching swarm-workers
    └── verifier-prompt.md    # Template for dispatching swarm-verifiers (both modes)

config/
└── agent-config.json         # Agent definitions to add to opencode.json
```

## Installation

### 1. Install the Skills

```bash
# Option A: Personal (all projects)
cp -r skills/swarm ~/.config/opencode/skills/swarm
cp -r skills/using-swarm ~/.config/opencode/skills/using-swarm

# Option B: Project-specific
cp -r skills/swarm .opencode/skills/swarm
cp -r skills/using-swarm .opencode/skills/using-swarm
```

### 2. Add the Agent Definitions

Add the `swarm-worker` and `swarm-verifier` agents to your `opencode.json` inside the `"agent"` object:

```json
{
  "agent": {
    "swarm-worker": {
      "description": "Cheap worker agent for executing a single scoped coding task.",
      "mode": "subagent",
      "model": "opencode-go/deepseek-v4-flash",
      "hidden": true,
      "steps": 30,
      "prompt": "You are a focused worker agent in a swarm..."
    },
    "swarm-verifier": {
      "description": "Verifies worker output against assigned tasks.",
      "mode": "subagent",
      "model": "opencode-go/deepseek-v4-flash",
      "hidden": true,
      "steps": 15,
      "prompt": "You are a verification agent..."
    }
  }
}
```

The full config is in `config/agent-config.json`. **Add these alongside your existing agents** — don't replace them.

### 3. Restart OpenCode

The skills and agents load at startup.

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

## Architecture

### The 7-Phase Workflow

1. **Context Extraction** — Reads AGENTS.md, file tree, git diff
2. **Spawn Proposal** — Defines workers, scopes, and contracts
3. **Refinement** — User adjusts the proposal until satisfied
4. **Dispatch** — All workers launched in parallel (v4-flash)
5. **Verification** — Per-worker checks + contract cross-check (v4-flash)
6. **Failure Handling** — 4-level escalation ladder for retries
7. **Aggregation** — Unified summary of all verified results

### The Escalation Ladder

| Level | Action | Model |
|-------|--------|-------|
| 1 | Retry same worker with verifier feedback | v4-flash |
| 2 | Retry with broader context | v4-flash |
| 3 | Planner re-scopes into smaller sub-tasks | v4-pro |
| 4 | Planner takes over directly | v4-pro |

The expensive model is always the last resort.

## Skills Reference

| Skill | When It Loads | What It Does |
|-------|---------------|--------------|
| `using-swarm` | Session start (if auto-injected) or when swarm concepts are invoked | Establishes the 1% rule, cost model, red flags, integration with other skills |
| `swarm` | User says "swarm:", "/swarm", "use swarm", "spawn agents", or auto-trigger fires | Executes the 7-phase orchestration workflow |

## Prompt Templates

| File | Purpose |
|------|---------|
| `worker-prompt.md` | Template for constructing worker dispatch prompts with SUBAGENT-STOP, context isolation, and output contracts |
| `verifier-prompt.md` | Template for both per-worker verification (Mode 1) and contract cross-check (Mode 2) |

## Design Principles

**Inspired by superpowers:**
- Meta-skill (`using-swarm`) establishes rules of engagement before any work begins
- DOT/Graphviz flowcharts as executable specifications for workflows
- SUBAGENT-STOP blocks prevent recursive skill loading in subagents
- Rationalization tables counter common excuses for skipping the workflow
- Red Flag tables define hard stops for dangerous shortcuts
- Cross-skill integration declares how swarm works with brainstorming, TDD, debugging

**Cost-first design:**
- Expensive model NEVER does bulk coding
- Workers are always the cheapest model
- Verification is cheap and parallel
- Context is trimmed to minimum needed
- Contracts prevent rework across workers

## Prerequisites

- **OpenCode** with the OpenCode Go subscription
- **AGENTS.md files** recommended — the planner reads these to scope workers correctly
- The `swarm-worker` and `swarm-verifier` agents configured in your `opencode.json`

## Tips

- **Write good AGENTS.md files.** The planner uses them to scope workers and give them correct context.
- **Define contracts clearly.** Contract mismatches are the most common failure mode across parallel workers.
- **Start with 2-3 workers.** More workers = more coordination overhead. Find the sweet spot.
- **Per-worker verifiers are cheap.** They only see one worker's output, not the whole project.
- **The verifier catches lies.** Cheap models sometimes hallucinate. The verifier's job is to catch that.
- **Escalate reluctantly.** Every escalation to v4-pro costs money. Exhaust v4-flash options first.
