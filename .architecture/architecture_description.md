# Frozen Architecture Snapshot

## System purpose

This repository is a lightweight orchestration workspace for agent-driven planning. Its primary concern is maintaining workflow contracts and state files rather than shipping runtime application code.

## Top-level structure

- `AGENTS.md` routes all agent work to `vibeCoding/agents.md` as the canonical workflow entry point.
- `CLAUDE.md` mirrors the same startup instruction for compatibility with other agents.
- `vibeCoding/agents.md` defines the shared workflow contract, including the three supported modes: architecture, component, and vibe.
- `vibeCoding/readme.md` explains the intended repository layout and the expected planning artifacts for each mode.
- `.architecture/` stores the frozen architecture baseline and future architectural decisions.

## Architectural decisions frozen by this snapshot

1. The repository remains documentation-first: workflow state is represented by markdown artifacts instead of executable orchestration code.
2. `vibeCoding/agents.md` stays the canonical human/agent workflow contract that other entry-point files defer to.
3. Architecture work is tracked in `.architecture/` using four durable artifacts:
   - `architecture_description.md` for the current approved architecture,
   - `plan.md` for open questions,
   - `state.md` for active focus and blockers,
   - `history.md` for append-only decision history.
4. Changes that would alter workflow modes, artifact responsibilities, or canonical entry points require an explicit architecture unfreeze and a new decision log entry.

## Unfreeze trigger

The architecture should be considered frozen until a future request explicitly asks to revise the workflow model, repository structure, or ownership of planning artifacts.
