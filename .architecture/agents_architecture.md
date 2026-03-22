# Architecture workflow contract

## Purpose

Use `.architecture/` to capture the current architecture, open architectural questions, and durable decisions.

## Read order

1. `vibeCoding/agents.md`
2. `.architecture/agents_architecture.md`
3. `.architecture/state.md`
4. `.architecture/plan.md`
5. `.architecture/history.md` when additional context is needed
6. `.architecture/architecture_description.md` for the current frozen architecture snapshot

## Rules

- `plan.md` is a backlog of open architectural questions and investigations, not an implementation task list.
- `state.md` records the current architectural focus, status, blockers, and next review expectation.
- `history.md` is append-only and records architecture decisions, review rounds, and transitions to `DONE`.
- `architecture_description.md` is the canonical architecture snapshot. Update it only when architecture decisions change.
- When architecture is explicitly frozen, record that in `state.md`, append the decision to `history.md`, and clear `plan.md` of active questions.
