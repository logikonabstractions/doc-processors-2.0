# STATE

This file tracks large, "hot topics" that are ongoing. It may not be active all the time.

## State management rules
- Keep this file focused on current execution state. Put rollups and resolved items in `HISTORY.md`.
- You only use this file when major blockers/issues are present.

## Current focus
- Target element: 70
- Revision ID: Comp.70.0.1

## Objective (current breakdown)
Break down Architecture Element 70 (Format Registry) into concrete, implementable components.

## Active assumptions / constraints
- Format registry data is small and rarely changes — DynamoDB on-demand is cost-effective.
- Consumers cache responses, so the registry API is not on the critical latency path.
- Write access is admin/deploy-time only; the runtime API is strictly read-only.
- All consumers are internal micro-services within the same VPC.

## Active issues
- [ ] Comp-70.1: Can we eliminate the dedicated API and Admin components?
  - Impact: QUESTION
  - Status: IN_PROGRESS
  - Unblock condition: Human decision on whether to keep 3 components, 2, or simplify to DynamoDB + shared library (see DISCUSSION.md for full options)
  - Notes: Agent recommends Option B (DynamoDB-only + shared client library). Awaiting human decision.
