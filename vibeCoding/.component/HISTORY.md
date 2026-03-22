# HISTORY

## Rules

- This file is non-authoritative.
- Use it for completed component drafts, resolved issues, major review outcomes, and durable component-design decisions.
- Prefer pointers to files, commits, or review notes instead of copying large blocks.


## Resolved issues

## Component-design decisions
<!-- Keep only decisions worth preserving across revisions. -->
- 2026-03-22: Amazon SQS Standard Queue selected for conversion request queue (40.1)
  - Component ID: 40.1
  - Related Discussions if any: N/A
  - Rationale: Standard queue provides higher throughput than FIFO and strict ordering is not needed for independent conversion jobs. Aligns with managed-service and fast-development priorities.
  - Impact: Determines the core messaging technology for the broker. Workers must handle at-least-once delivery idempotently.

- 2026-03-22: Amazon SNS Standard Topic selected for conversion result events (40.3)
  - Component ID: 40.3
  - Related Discussions if any: N/A
  - Rationale: SNS provides native fan-out to multiple subscribers, matching the architecture's requirement that result events be "available for any future subscriber." Lightweight and zero-cost when no subscribers exist.
  - Impact: Conversion Workers publish to SNS rather than a second SQS queue, enabling easy addition of notification/analytics subscribers later.

- 2026-03-22: Message schemas defined as JSON Schema in the repository (40.5) rather than a managed schema registry
  - Component ID: 40.5
  - Related Discussions if any: N/A
  - Rationale: Aligns with fast-development priority. A full schema registry (e.g., EventBridge Schema Registry) adds operational overhead not justified by the current small number of event types.
  - Impact: Schema evolution must be managed via versioned schema files in the repo. Producers validate at publish time using a lightweight library.

## Superseded choices / changes
<!-- None yet. -->
