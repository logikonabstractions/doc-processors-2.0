# STATE

This file tracks large, "hot topics" that are ongoing. It may not be active all the time.


## State management rules

- The current target system should remain stable during a draft unless a human changes the problem statement or scope.
- The status can be set to `DONE` only when the human reviewer has explicitly approved the result
- Keep this file focused on current execution state. Put rollups and resolved items in `HISTORY.md`.

## Current focus

- Revision ID: Arch.0.1
- Status: IN_REVIEW  <!-- one of: NOT_STARTED | IN_PROGRESS | IN_REVIEW | DONE -->

## Objective (current draft)
<!-- 1 sentence. Keep aligned with `.architecture/ARCHITECTURE_DESCRIPTION.md`. -->
Define a micro-service architecture for a file conversion system that accepts text-based files from anonymous web users, converts them asynchronously, and returns results — designed for extensibility to other file types.

## Active assumptions / constraints
<!-- Keep only the assumptions or constraints that materially affect the current architecture draft. -->
- AWS-hosted, managed services preferred (fast development priority)
- IaC via terraform
- Initial scope: text file conversions only
- Anonymous users only (no auth)
- Ephemeral file storage (time-limited retention)

## Work log (current session)
<!-- Append-only bullets for what changed and why. Prefer file/section references. -->
- 2026-03-22: Completed initial architecture draft (Arch.0.1) with 7 elements: API Gateway (10), Conversion API Service (20), Conversion Worker (30), Message Broker (40), Temporary File Storage (50), Job Metadata Store (60), Format Registry (70). Filled in scope boundaries, system interaction summary, system-wide concerns, and open questions.
- 2026-03-22: Opened discussion items Arch-0.1 (upload pattern), Arch-0.2 (worker deployment model), Arch-0.3 (Format Registry deployment) in DISCUSSION.md.

## Workflow state
<!-- Dispatcher flags. Checked = active/needed. Cleared once handled. -->
- [x] PROBLEM_CLARIFIED
- [x] DRAFT_CREATED
- [x] HUMAN_REVIEW_REQUIRED
- [ ] DECISIONS_CAPTURED

## Active issues
<!-- Keep only active issues here. Move resolved items to HISTORY.md. -->
- [ ] Arch-0.1: Upload pattern — direct-to-storage vs. API-proxied
  - Impact: QUESTION
  - Status: IN_REVIEW
  - Unblock condition: Human decides preferred upload pattern
  - Notes: See DISCUSSION.md Arch-0.1

- [ ] Arch-0.2: Worker deployment model — container vs. serverless
  - Impact: QUESTION
  - Status: IN_REVIEW
  - Unblock condition: Human decides based on expected file sizes and conversion durations
  - Notes: See DISCUSSION.md Arch-0.2

- [ ] Arch-0.3: Format Registry deployment — embedded config vs. standalone service
  - Impact: QUESTION
  - Status: IN_REVIEW
  - Unblock condition: Human decides based on expected rate of format additions
  - Notes: See DISCUSSION.md Arch-0.3
