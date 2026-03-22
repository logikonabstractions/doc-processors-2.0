# STATE

This file tracks large, "hot topics" that are ongoing. It may not be active all the time.


## State management rules

- The current target system should remain stable during a draft unless a human changes the problem statement or scope.
- The status can be set to `DONE` only when the human reviewer has explicitly approved the result
- Keep this file focused on current execution state. Put rollups and resolved items in `history.md`.

## Current focus

- Revision ID: Arch.0.1
- Status: FREEZE

## Objective (current draft)

Design a serverless, maximally-decoupled micro-service architecture for ephemeral file conversions on AWS, starting with text-based formats.

## Active assumptions / constraints

- Everything hosted on AWS
- Fast development priority: favor managed services over custom code
- IaC via Terraform
- Text file conversions only in initial scope (extensible to other types)
- Files are ephemeral with time-limited retention
- Individual file sizes are bounded (maximum upload size enforced)

## Work log (current session)

- 2026-03-22: Architecture FREEZE executed. All discussion items resolved. Key decisions captured.

## Workflow state

- [x] PROBLEM_CLARIFIED
- [x] DRAFT_CREATED
- [x] HUMAN_REVIEW_REQUIRED
- [x] DECISIONS_CAPTURED

## Key Architecture Decisions

- **Cloud provider**: AWS exclusively
- **Compute model**: Serverless-first for all compute elements — scale-to-zero, pay-per-invocation, no reserved capacity
- **Deployment philosophy**: Maximal decoupling — each service independently deployable with atomic deployments
- **File transfer pattern**: Direct-to-storage via pre-signed URLs; API services never proxy file content
- **Processing model**: Asynchronous, queue-based job processing; Message Broker decouples API from workers
- **Storage philosophy**: Minimal and ephemeral only — files and job metadata auto-expire via lifecycle policies / TTL
- **Worker deployment**: Serverless functions triggered by queue events (seconds-to-minutes conversions); container-based workers preserved as future extensibility path via the same queue
- **Format Registry**: Standalone serverless service backed by managed key-value store; independently deployable from API and Worker services
- **Traffic profile**: Designed for unpredictable, bursty usage with rapid scale-up and scale-to-zero
- **Security posture**: Anonymous users, rate limiting at API Gateway, pre-signed time-limited URLs for all file access, TLS, non-guessable job IDs
- **IaC**: Terraform for all cloud infrastructure

## Active issues

None. All issues resolved prior to freeze (see history.md).
