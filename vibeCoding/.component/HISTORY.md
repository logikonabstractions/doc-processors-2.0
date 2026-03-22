# HISTORY

## Rules

- This file is non-authoritative.
- Use it for completed component drafts, resolved issues, major review outcomes, and durable component-design decisions.
- Prefer pointers to files, commits, or review notes instead of copying large blocks.

## Entry Templates

Use the most relevant template below when adding an entry. All entries must conform to one such template.


## Resolved issues
- Comp-70.0.1: Initial Format Registry component draft
  - 2026-03-22
  - Component ID: 70
  - Resolution: Drafted the Element 70 component breakdown covering the query API, metadata store, publication workflow, and shared observability/cache-control concerns.
  - Notes: See `vibeCoding/.component/COMPONENTS_DESCRIPTIONS.md` revision Comp.70.0.1.

## Component-design decisions
- 2026-03-22: Keep runtime reads and administrative writes split into separate components
  - Component ID: 70
  - Related Discussions if any: None
  - Rationale: The architecture requires a read-only API for consumers, while registry updates are infrequent and better handled through controlled publication workflows.
  - Impact: Element 70 is decomposed into a read-only query API plus a publication workflow instead of a single mutable service.

- 2026-03-22: Treat cache/revision metadata as a first-class cross-cutting concern
  - Component ID: 70
  - Related Discussions if any: None
  - Rationale: Consumers are expected to cache registry responses aggressively, so revision-aware cache headers and telemetry need explicit design coverage.
  - Impact: Added a dedicated observability/cache-control module to support both runtime queries and publication events.

## Superseded choices / changes
<!-- Optional. Use when previous choices were later invalidated. -->
