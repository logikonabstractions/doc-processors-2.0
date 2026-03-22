# STATE

This file tracks large, "hot topics" that are ongoing. It may not be active all the time.

## State management rules
- Keep this file focused on current execution state. Put rollups and resolved items in `history.md`.
- You only use this file when major blockers/issues are present.

## Current focus
- Target element: 70 (Format Registry)
- Revision ID: Comp.70.0.1

## Objective (current breakdown)
Break down Element 70 (Format Registry) into concrete, implementable components: API layer, data store, seed/admin mechanism, and infrastructure-as-code module.

## Active assumptions / constraints
- **WARNING**: Architecture status is `NOT_STARTED` (not `FREEZE`). Component breakdown is proceeding as instructed, but architectural decisions may still change.
- All compute is serverless (AWS Lambda) per architectural preferences
- DynamoDB chosen for key-value storage (HTTP API, no connection pooling, serverless-compatible)
- Terraform for all IaC
- Read-only runtime API; write access is deploy-time/admin only

## Active issues
- [ ] Comp.70.0.1: Architecture not frozen
  - Impact: QUESTION
  - Status: IN_PROGRESS
  - Unblock condition: Architecture reaches FREEZE status, or explicit confirmation to proceed with current architecture
  - Notes: Proceeding as instructed per workflow contract (log warning but continue). Component design may need revision if architecture changes.
