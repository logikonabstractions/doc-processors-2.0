# STATE

This file tracks large, "hot topics" that are ongoing. It may not be active all the time.

## State management rules
- Keep this file focused on current execution state. Put rollups and resolved items in `HISTORY.md`.
- You only use this file when major blockers/issues are present.

## Current focus
- Target element: 40
- Revision ID: Comp.40.0.1
- Status: IN_REVIEW  <!-- one of: NOT_STARTED | IN_PROGRESS | IN_REVIEW | DONE -->

## Objective (current breakdown)
Break down the Message Broker (Element 40) into concrete, implementable components covering the SQS queues, SNS topic, event-source mapping, message schemas, and Terraform IaC module.

## Active assumptions / constraints
- AWS SQS Standard Queue chosen over FIFO (no ordering requirement, higher throughput)
- Amazon SNS chosen for result events to enable future fan-out
- Message payloads carry only job metadata and S3 references, never file content
- Terraform module should be self-contained and independently deployable

## Work log (current session)
- 2026-03-22: Created initial component breakdown for Element 40 with 6 components (40.1–40.6)
- 2026-03-22: Components: Conversion Request Queue (SQS), DLQ (SQS), Conversion Result Topic (SNS), Queue-to-Lambda Event Source Mapping, Message Schema & Contract, Broker Infrastructure Terraform Module

## Workflow state
- [x] ELEMENT_REVIEWED
- [x] DRAFT_CREATED
- [ ] HUMAN_REVIEW_REQUIRED
- [ ] DECISIONS_CAPTURED

## Active issues
<!-- No active blockers. Open questions logged in COMPONENTS_DESCRIPTIONS.md. -->
