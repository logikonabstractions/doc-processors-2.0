# HISTORY

## Rules

- This file is non-authoritative.
- Use it for completed component drafts, resolved issues, major review outcomes, and durable component-design decisions.
- Prefer pointers to files, commits, or review notes instead of copying large blocks.

## Entry Templates

Use the most relevant template below when adding an entry. All entries must conform to one such template.


## Resolved issues

## Component-design decisions
- 2026-03-22: Initial component breakdown for Element 70 (Format Registry) — 4 components
  - Component ID: 70.1, 70.2, 70.3, 70.4
  - Related Discussions if any: N/A
  - Rationale: Element 70 decomposes naturally into a read-only API (70.1), a data store (70.2), a seed/admin mechanism (70.3), and a Terraform module (70.4). DynamoDB chosen over S3 for query flexibility; Terraform seeding preferred over manual updates for auditability.
  - Impact: Sets the component structure for the Format Registry. All four components required for a deployable, independently manageable registry service.

## Superseded choices / changes
