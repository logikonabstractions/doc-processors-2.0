# STATE

This file tracks large, "hot topics" that are ongoing. It may not be active all the time.


## State management rules

- The current target system should remain stable during a draft unless a human changes the problem statement or scope.
- The status can be set to `DONE` only when the human reviewer has explicitly approved the result
- Keep this file focused on current execution state. Put rollups and resolved items in `HISTORY.md`.

## Current focus

- Revision ID: Arch.0.2
- Status: FREEZE

## Objective (current draft)

Design a serverless micro-service architecture on AWS for ephemeral, asynchronous text-based file conversions, accessible by anonymous web users.

## Active assumptions / constraints

- All infrastructure hosted on AWS
- Fast development prioritized — managed services preferred over custom code
- Infrastructure managed via Terraform
- Text file conversions only in initial scope (extensible to other file types)
- Files are ephemeral with time-limited retention
- Individual file sizes are bounded (maximum upload size enforced)

## Work log (current session)

- 2026-03-22: Architecture FREEZE executed. All discussion items resolved; architecture description finalized.

## Workflow state

## Key Architecture Decisions

- **Cloud provider**: AWS (all infrastructure hosted on AWS)
- **IaC tool**: Terraform for all cloud resource management and deployment
- **Compute model**: Serverless-first — all compute elements are serverless functions (scale-to-zero, pay-per-invocation, no reserved capacity)
- **File transfer pattern**: Direct-to-storage via pre-signed URLs — API services never proxy file content
- **Worker deployment**: Serverless functions triggered by queue events; container-based workers preserved as a future extensibility path via the same queue
- **Async processing**: Queue-based decoupling between API and workers (at-least-once delivery, dead-letter queue for failures)
- **Format Registry**: Standalone serverless micro-service backed by a managed key-value store — fully independently deployable
- **Storage philosophy**: Minimal and ephemeral only — object storage for files, key-value store for job metadata, both with automatic TTL-based expiry
- **Decoupling**: Maximal — each service independently deployable with atomic deployments
- **Scalability**: Designed for unpredictable, bursty traffic with rapid scale-up and scale-to-zero
- **Security**: Anonymous access with rate limiting, pre-signed time-limited URLs, TLS, private internal network, non-guessable job IDs

## Active issues
