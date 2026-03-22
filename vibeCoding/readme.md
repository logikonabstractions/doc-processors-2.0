# Vibe Coding-Agent Orchestration

This document is human-oriented and not to be considered for agent workflows.

## TODO & CURRENT STATE
=======
## ROADMAP - VIBECODING
* Add CLEAR instruction that is something is ask about a checkpoint X.Y that doesn't exist, the agent MUST stop & ask for claricafion and not do anything.

## TODO
* update this readme to explain new workflows

This repository tracks agentic work using state files only.

## Modes

- `.architecture/` — architectural design
- `.vibe/` — implementation / feature execution
- `.component/` — reserved for component design

## Architecture workflow files

- `.architecture/agents_architecture.md` — architecture execution contract and read order
- `.architecture/architecture_description.md` — required architecture response format
- `.architecture/plan.md` — architecture questions, investigations, and decision backlog
- `.architecture/state.md` — current architecture draft, active focus, blockers, and work log
- `.architecture/history.md` — resolved questions, review history, and durable decisions

## Vibe workflow files

- `.vibe/agents_vibe.md` — implementation workflow contract
- `.vibe/plan.md` — implementation checkpoint backlog
- `.vibe/state.md` — active checkpoint and current session state
- `.vibe/history.md` — completed checkpoints and resolved issues
- `.vibe/context.md` — optional handoff notes and durable context

## Canonical templates (DRY)

To avoid drift, treat these as the single source of truth for structure:

- `templates/META_TEMPLATES/plan.md`
- `templates/META_TEMPLATES/state.md`
- `templates/META_TEMPLATES/history.md`

## Workflow loop

1. Read `agents.md`, `.vibe/state.md`, and `.vibe/plan.md` (plus optional history/context if needed).
2. Pick the active checkpoint from `.vibe/state.md`.
3. Implement only that checkpoint.
4. Run required demo/test commands.
5. Update `.vibe/state.md` and `.vibe/history.md` (and `.vibe/context.md` if relevant)
7. Human reviews.

## Quick consistency checks (agent + human)

Use this before or after each checkpoint to keep planning artifacts aligned:

- **Checkpoint sync:** `state.md` objective/deliverables/acceptance exactly match the active checkpoint in `plan.md`.
- **Status accuracy:** `state.md` status is one of `NOT_STARTED`, `IN_PROGRESS`, `IN_REVIEW`, `BLOCKED`, `DONE` and reflects reality.
- **Issue hygiene:** active blockers/questions live in `state.md` with impact + unblock condition; resolved ones move to `history.md`.
- **Evidence quality:** all acceptance claims point to concrete commands, outputs, commits, or screenshots.
- **Scope discipline:** only one checkpoint is active unless explicitly requested otherwise.

## Templates and examples

When creating or updating planning docs, link to and follow the canonical templates in
`templates/META_TEMPLATES/` rather than copying template blocks into additional docs.

## Philosophy

Keep it simple:

- small checkpoints,
- frequent human review,
- no autonomous long-running loops.
