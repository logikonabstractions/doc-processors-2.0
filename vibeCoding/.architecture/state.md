# STATE

This file tracks large, "hot topics" that are ongoing. It may not be active all the time.


## State management rules

- The current target system should remain stable during a draft unless a human changes the problem statement or scope.
- The status can be set to `DONE` only when the human reviewer has explicitly approved the result
- Keep this file focused on current execution state. Put rollups and resolved items in `history.md`.

## Current focus

- Revision ID: Arch.0.2
- Status: FREEZE

## Objective (current draft)

Design a serverless, maximally-decoupled micro-service architecture on AWS for ephemeral text-based file conversions, accessible by anonymous web users.

## Active assumptions / constraints

- All infrastructure hosted on AWS
- Fast development priority: prefer managed services over custom code
- IaC via Terraform
- Text file conversions only in initial scope (extensible to other types)
- Files are ephemeral with time-limited retention
- Individual file sizes are bounded (maximum upload size enforced)

## Work log (current session)

- 2026-03-22: Initial architecture draft (Arch.0.1) created with 7 elements
- 2026-03-22: Resolved Arch-0.1 (direct-to-storage upload), Arch-0.2 (serverless workers), Arch-0.3 (standalone Format Registry)
- 2026-03-22: Architecture description updated to reflect all resolved decisions (Arch.0.2 revision)
- 2026-03-22: Architecture FROZEN — all elements reviewed and approved

## Workflow state

- [ ] PROBLEM_CLARIFIED
- [ ] DRAFT_CREATED
- [ ] HUMAN_REVIEW_REQUIRED
- [ ] DECISIONS_CAPTURED

## Key Architecture Decisions

- **Cloud provider**: AWS exclusively
- **Compute model**: Serverless-first for all compute elements — scale-to-zero, pay-per-invocation, no reserved capacity
- **File transfer pattern**: Direct-to-storage via pre-signed URLs; API services never proxy file content
- **Async processing**: Queue-based decoupling between API and conversion workers; at-least-once delivery with idempotent workers
- **Worker deployment**: Serverless functions triggered by queue events (container-based workers preserved as future extensibility path via same queue)
- **Format Registry**: Standalone serverless service backed by managed key-value store; independently deployable from API and Worker services
- **Storage philosophy**: Minimal and ephemeral only — files and job metadata auto-expire via lifecycle policies and TTLs
- **Deployment atomicity**: Each service independently deployable; maximal decoupling between all elements
- **IaC**: Terraform for all cloud resource management
- **Development priority**: Fast development — favor managed/hosted services over custom code

## Active issues

(none — all issues resolved prior to FREEZE)
