# VIBE Workflow Contract

## Purpose

Translate **one component** into implementable stages and checkpoints, then implement them. This mode has two sub-modes: **VIBE-DRAFT** (planning) and **VIBE-IMPLEMENT** (coding).

## Input requirements

**MUST contain one** of the following:

- **A component identifier** (ID or name, e.g. "component 10.2") → triggers **VIBE-DRAFT**
- **A checkpoint identifier** (e.g. "checkpoint 10.2.1.1") matching an entry in `.vibe/PLAN.md` → triggers **VIBE-IMPLEMENT**

If the input is ambiguous, **always ask to confirm** which sub-mode applies.


## VIBE-DRAFT — Plan stages and checkpoints for a component

Given a component identifier, read its description from `.component/COMPONENTS_DESCRIPTIONS.md` and draft `.vibe/PLAN.md`. Organize work into **stages** (groups of related implementation steps) and **checkpoints** (small, reviewable units within a stage).

### Instruction precedence & read order
1. As specified by `AGENTS.md`
2. This file
3. `.architecture/ARCHITECTURE_DESCRIPTION.md` (read-only reference)
4. `.component/COMPONENTS_DESCRIPTIONS.md` (read-only reference)
5. `.vibe/PLAN.md`
6. `.vibe/STATE.md`
7. `.vibe/HISTORY.md`
8. `.vibe/CONTEXT.md`

---

## VIBE-IMPLEMENT — Implement a specific checkpoint

Given a checkpoint identifier, implement it according to `.vibe/PLAN.md`.

### Instruction precedence & read order
1. As specified by `AGENTS.md`
2. This file
3. `.vibe/PLAN.md`
4. `.vibe/STATE.md`
5. `.vibe/HISTORY.md`
6. `.vibe/CONTEXT.md`

---

## Meta-templates

Found under `/meta_templates/.vibe`

| File | Role |
|------|------|
| `/meta_templates/.vibe/state_tplt.md` | Current checkpoint, status, work log, active issues, decisions |
| `/meta_templates/.vibe/plan_tplt.md` | Ordered list of checkpoints with objectives and acceptance criteria |
| `/meta_templates/.vibe/history_tplt.md` | Completed checkpoints and decisions |
| `/meta_templates/.vibe/context_tplt.md` | Shared context: architecture notes, key decisions, gotchas, hot files |

## Scope and cadence
- A **stage** groups a few related checkpoints.
- A **checkpoint** is a small, reviewable diff — at most a few commits.
- Limit changes to what is necessary to meet acceptance criteria.

## Metafile updates
- When a checkpoint moves to DONE, update `.vibe/STATE.md` to reflect the new state.
- NEVER remove checkpoints from `.vibe/PLAN.md` unless explicitly asked to.
- NEVER re-number checkpoints unless explicitly asked to.

## Version control policy
- The commit message MUST ALWAYS follow this format for `VIBE-IMPLEMENT` (and if you can, the PR title/message as well): <checkpoint-id> - <checkpoint name>
- Commit coherent changes: a commit should be a single set of related and consistent changes towards a given checkpoint.