# HISTORY

## Rules

- This file is non-authoritative.
- Use it for completed component drafts, resolved issues, major review outcomes, and durable component-design decisions.
- Prefer pointers to files, commits, or review notes instead of copying large blocks.

## Entry Templates

Use the most relevant template below when adding an entry. All entries must conform to one such template.


## Resolved issues
- <Comp-N.N>: <TITLE>
  - YYYY-MM-DD 
  - Component ID: <ID>
  - Resolution: <1-2 lines>
  - Notes: <optional>

## Component-design decisions
<!-- Keep only decisions worth preserving across revisions. -->
- 2026-03-22: Initial component breakdown for Element 10 (API Gateway) — Revision Comp.10.0.1
  - Component ID: 10.1, 10.2, 10.3, 10.4, 10.5
  - Related Discussions if any: N/A
  - Rationale: Broke Element 10 into 5 components: HTTP API (10.1), CORS config (10.2), rate limiting (10.3), custom domain & TLS (10.4), and Terraform IaC module (10.5). Chose AWS API Gateway HTTP API type over REST API for lower cost, lower latency, and alignment with serverless-first preference.
  - Impact: Defines the full scope and technology choices for the API Gateway layer. All components are AWS-managed or Terraform-provisioned, consistent with the fast-development priority.

## Superseded choices / changes
<!-- Optional. Use when previous choices were later invalidated. -->
- YYYY-MM-DD: <old choice>
  - Component ID: <Component ID>
  - Related Discussions if any: <Comp-N.N>
  - Replaced by: <new direction>
  - Reason: <why it changed>
