# STATE

This file tracks large, "hot topics" that are ongoing. It may not be active all the time.

## State management rules
- Keep this file focused on current execution state. Put rollups and resolved items in `HISTORY.md`.
- You only use this file when major blockers/issues are present.

## Current focus
- Target element: 10
- Revision ID: Comp.10.0.1

## Objective (current breakdown)
Break down Element 10 (API Gateway) into concrete, implementable components with specific technology choices and clear responsibilities.

## Active assumptions / constraints
- AWS API Gateway HTTP API type selected over REST API for lower cost and simpler configuration.
- Per-IP rate limiting deferred to a future iteration; account/route-level throttling is sufficient initially.
- Custom domain + TLS is included as a component but may be deferred during early development (open question raised).
- CloudFront + WAF not included initially (open question raised).

## Active issues
- [ ] Comp.10.0.1: CloudFront front-loading decision
  - Impact: QUESTION
  - Status: NOT_STARTED
  - Unblock condition: Decision on whether CloudFront + WAF should be added from the start or deferred
  - Notes: Raised as open question in COMPONENTS_DESCRIPTIONS.md. Does not block initial component design.
