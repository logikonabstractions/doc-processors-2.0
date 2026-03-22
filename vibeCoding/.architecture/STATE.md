# STATE

## State management rules

- The current target system should remain stable during a draft unless a human changes the problem statement or scope.
- The status can be set to `DONE` only when the human reviewer has explicitly approved the result
- Keep this file focused on current execution state. Put rollups and resolved items in `HISTORY.md`.

## Current focus

- Revision ID: Arch.0.2
- Status: FREEZE

## Objective (current draft)

Microservices-based file conversion system enabling anonymous web users to upload text-based files and receive conversions to other supported formats, deployed on AWS with serverless-first architecture.

## Key Architecture Decisions

- **Cloud provider**: AWS exclusively
- **Compute model**: Serverless-first — all compute elements are AWS Lambda functions (scale-to-zero, pay-per-invocation, no reserved capacity)
- **Deployment philosophy**: Maximal decoupling — each service independently deployable with atomic deployments
- **File transfer pattern**: Direct-to-storage via pre-signed URLs — API services never proxy file content (Arch-0.1)
- **Worker deployment**: Serverless functions triggered by queue events; container-based workers reserved as future extensibility path (Arch-0.2)
- **Format Registry**: Standalone service (serverless function + managed key-value store) for full deployment independence (Arch-0.3)
- **Storage model**: Strictly ephemeral — all files and job metadata auto-expire via lifecycle policies and TTL
- **Messaging**: Queue-based async decoupling between API and workers; at-least-once delivery with idempotent consumers
- **Traffic model**: Designed for bursty, unpredictable usage with rapid scale-up and scale-to-zero
- **IaC**: Terraform for all cloud resource management
- **Security**: Anonymous-only access, rate limiting, pre-signed time-limited URLs, TLS, private internal networking, non-guessable job IDs
- **Architectural elements**: 7 elements — API Gateway (10), Conversion API Service (20), Conversion Worker (30), Message Broker (40), Temporary File Storage (50), Job Metadata Store (60), Format Registry (70)

## Active assumptions / constraints

- Files are ephemeral with configurable retention period
- Individual file sizes are bounded (maximum upload size enforced)
- Current scope is text-based conversions only (seconds-to-minutes duration)
- Fast development priority: favor managed services over custom code

## Workflow state

- [x] PROBLEM_CLARIFIED
- [x] DRAFT_CREATED
- [x] HUMAN_REVIEW_REQUIRED
- [x] DECISIONS_CAPTURED

## Active issues

None. All issues resolved — see HISTORY.md.
