# HISTORY

## Rules

- This file is non-authoritative.
- Use it for completed component drafts, resolved issues, major review outcomes, and durable component-design decisions.
- Prefer pointers to files, commits, or review notes instead of copying large blocks.

## Entry Templates

Use the most relevant template below when adding an entry. All entries must conform to one such template.


## Resolved issues

## Component-design decisions
<!-- Keep only decisions worth preserving across revisions. -->
- 2026-03-22: Use DynamoDB (on-demand) as the Format Data Store instead of a static JSON file
  - Component ID: 70.2
  - Related Discussions if any: N/A
  - Rationale: DynamoDB allows admin updates to format registry data without redeploying the API Lambda, preserving deployment independence. The dataset is small enough to stay within free tier.
  - Impact: Enables runtime admin updates via 70.3 without touching 70.1 deployments.

- 2026-03-22: Separate read API (70.1) from admin write API (70.3) into distinct Lambda functions
  - Component ID: 70.1, 70.3
  - Related Discussions if any: N/A
  - Rationale: Enforces the read-only contract for consumer services at the infrastructure level (separate function, separate IAM role). Admin writes are IAM-authenticated and independently deployable.
  - Impact: Clear security boundary between consumer-facing reads and admin writes.

## Superseded choices / changes
<!-- Optional. Use when previous choices were later invalidated. -->
