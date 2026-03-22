# STATE

This file tracks large, "hot topics" that are ongoing. It may not be active all the time.


## State management rules

- The current target system should remain stable during a draft unless a human changes the problem statement or scope.
- The status can be set to `DONE` only when the human reviewer has explicitly approved the result
- Keep this file focused on current execution state. Put rollups and resolved items in `history.md`.

## Current focus

- Revision ID: Arch.0.1
- Status: NOT_STARTED  <!-- one of: NOT_STARTED | IN_PROGRESS | IN_REVIEW | DONE | FREEZE -->

## Objective (current draft)
<!-- 1 sentence. Keep aligned with `.architecture/architecture_description.md`. -->

## Active assumptions / constraints
<!-- Keep only the assumptions or constraints that materially affect the current architecture draft. -->
- <assumption or constraint>

## Work log (current session)
<!-- Append-only bullets for what changed and why. Prefer file/section references. -->
- YYYY-MM-DD: <change made and reason>

## Workflow state
<!-- Dispatcher flags. Checked = active/needed. Cleared once handled. -->
- [ ] PROBLEM_CLARIFIED
- [ ] DRAFT_CREATED
- [ ] HUMAN_REVIEW_REQUIRED
- [ ] DECISIONS_CAPTURED

## Key Architecture Decisions
<!-- Populated at FREEZE. Summarize the major orientations chosen during the architecture review.
     This section is the primary reference for downstream agents (component, vibe).
     Use concise bullet points. Examples: cloud provider, core patterns, technology families, key constraints. -->
- <decision>

## Active issues
<!-- Keep only active issues here. Move resolved items to history.md. -->
- [ ] Arch.0.1: <short title>
  - Impact: QUESTION <!-- QUESTION | MINOR | MAJOR | BLOCKER -->
  - Status: NOT_STARTED <!-- one of: NOT_STARTED | IN_PROGRESS | IN_REVIEW | DONE -->
  - Unblock condition: <what must be true to proceed>
  - Notes: <optional context>
