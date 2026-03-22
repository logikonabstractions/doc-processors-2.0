# STATE

This file tracks large, "hot topics" that are ongoing. It may not be active all the time.


## State management rules

- The current target system should remain stable during a draft unless a human changes the problem statement or scope.
- The status can be set to `DONE` only when the human reviewer has explicitly approved the result
- Keep this file focused on current execution state. Put rollups and resolved items in `HISTORY.md`.

## Current focus

- Revision ID: Arch.0.2
- Status: IN_REVIEW  <!-- one of: NOT_STARTED | IN_PROGRESS | IN_REVIEW | DONE -->

## Objective (current draft)
<!-- 1 sentence. Keep aligned with `.architecture/ARCHITECTURE_DESCRIPTION.md`. -->
Define a serverless-first, maximally-decoupled micro-service architecture for an ephemeral file conversion system that accepts text-based files from anonymous web users, converts them asynchronously, and returns results via pre-signed URLs.

## Active assumptions / constraints
<!-- Keep only the assumptions or constraints that materially affect the current architecture draft. -->
- AWS-hosted, managed services preferred (fast development priority)
- IaC via terraform
- Initial scope: text file conversions only (seconds-to-minutes duration)
- Anonymous users only (no auth)
- Serverless-first: all compute is serverless functions, scale-to-zero, no reserved capacity
- Maximal decoupling: each service independently deployable with atomic deployments
- Minimal storage: ephemeral only, no persistent data stores beyond job coordination
- Unpredictable, bursty traffic patterns — must scale up rapidly

## Work log (current session)
<!-- Append-only bullets for what changed and why. Prefer file/section references. -->
- 2026-03-22: Completed initial architecture draft (Arch.0.1) with 7 elements: API Gateway (10), Conversion API Service (20), Conversion Worker (30), Message Broker (40), Temporary File Storage (50), Job Metadata Store (60), Format Registry (70). Filled in scope boundaries, system interaction summary, system-wide concerns, and open questions.
- 2026-03-22: Opened discussion items Arch-0.1 (upload pattern), Arch-0.2 (worker deployment model), Arch-0.3 (Format Registry deployment) in DISCUSSION.md.
- 2026-03-22: Human provided 4 architectural preferences: maximal decoupling, serverless-first, minimal storage, unpredictable traffic. Resolved all 3 discussion items: Arch-0.1 → direct-to-storage (Option A), Arch-0.2 → serverless (Option B), Arch-0.3 → standalone service (Option B). Updated ARCHITECTURE_DESCRIPTION.md to Arch.0.2 revision — all elements now reflect serverless deployment, pre-signed URL upload pattern, and Format Registry as standalone service. Added "Architectural preferences" section. Removed open question about conversion time limits (answered: seconds-to-minutes, fail beyond serverless limits). Updated system-wide concerns for serverless model.

## Workflow state
<!-- Dispatcher flags. Checked = active/needed. Cleared once handled. -->
- [x] PROBLEM_CLARIFIED
- [x] DRAFT_CREATED
- [x] HUMAN_REVIEW_REQUIRED
- [x] DECISIONS_CAPTURED

## Active issues
<!-- Keep only active issues here. Move resolved items to HISTORY.md. -->
- (all prior issues resolved — see HISTORY.md)
