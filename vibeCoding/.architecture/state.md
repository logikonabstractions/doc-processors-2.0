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
- Fast development priority: favor managed services over custom code
- IaC via Terraform
- Text file conversions only in initial iteration (extensible to other file types)
- Files are ephemeral with time-limited retention
- Individual file sizes are bounded (maximum upload size enforced)
- All compute is serverless (scale-to-zero, no reserved capacity)
- All services independently deployable (atomic deployments)

## Work log (current session)

- 2026-03-22: Architecture FREEZE applied. All discussion items resolved, architecture reviewed and approved.

## Key Architecture Decisions

- **Cloud provider**: AWS (all infrastructure hosted on AWS)
- **Compute model**: Serverless-first — all compute elements are serverless functions (scale-to-zero, pay-per-invocation, no reserved capacity)
- **Deployment strategy**: Maximal decoupling — every service is independently deployable with atomic deployments
- **File transfer pattern**: Direct-to-storage via pre-signed URLs — API services never proxy file content
- **Processing model**: Asynchronous, queue-based — API service publishes events to a message broker, workers are triggered by queue events
- **Storage philosophy**: Minimal and ephemeral — files and job metadata auto-expire via lifecycle policies / TTL
- **Format Registry**: Standalone serverless service backed by a managed key-value store (not embedded configuration) — format updates deploy independently of API and Worker
- **Job Metadata Store**: Managed key-value / document store with TTL-based expiry, HTTP-based API (no connection pooling), serverless-compatible
- **Traffic pattern**: Designed for unpredictable, bursty traffic with rapid scale-up and scale-to-zero
- **Future extensibility**: Queue-based decoupling allows adding container-based workers for long-running conversions without breaking changes
- **IaC**: Terraform for all cloud infrastructure
- **Security model**: Anonymous access, rate limiting at API Gateway, pre-signed time-limited URLs, TLS, non-guessable job IDs (UUIDs)

## Active issues

(none — all issues resolved prior to FREEZE)
