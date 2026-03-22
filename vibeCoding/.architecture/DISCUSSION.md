# PLAN

## How to use this file

- This is the **architecture discussions**.
- Use it to track architecture questions that require clarification, comparison, investigation, or explicit decision.
- Keep one item per question or decision.
- When a question is resolved, keep the item here for traceability, mark it `RESOLVED`, and summarize the outcome in `HISTORY.md`.
- Any Arch-X.Y with a status `RESOVLED` should not be updated later. If the issue needs to be revisited, favor opening a new Arch-X.Y and reference the previous discussion.

## Question resolution

When a question is resolved:
  - Add a relevant entry to `HISTORY.md`
  - Update this file's status to `DONE` along with any field (notes, etc.)

## Active architecture issues

### Arch-0.1 — File upload pattern: direct-to-storage vs API-proxied

- Type: DECISION
- Status: OPEN
- Related element(s):
  - API Gateway (10), Conversion API Service (20), Temporary File Storage (50)
- Description:
  - The architecture supports two upload patterns: (A) the client uploads the file directly to Temporary File Storage via a pre-signed URL obtained from the Conversion API Service, or (B) the client uploads the file through the API Gateway / Conversion API Service, which then stores it. This choice materially affects API Gateway sizing, latency, cost, and complexity.
- Known options / hypotheses:
  - **Option A — Direct-to-storage (pre-signed URL)**: Client requests a pre-signed upload URL from the API, then uploads directly to storage. Advantages: removes the API service as a file-transfer bottleneck, reduces API compute costs, handles large files more gracefully. Disadvantages: requires a two-step upload flow, slightly more complex client logic, harder to validate file content before storage.
  - **Option B — API-proxied upload**: Client uploads the file as a multipart form to the API service, which stores it. Advantages: simpler client flow (single request), allows server-side validation before storage. Disadvantages: API service must handle large file transfers, increases compute/memory requirements, potential timeout issues for large files.
- Required input / evidence:
  - Expected file size distribution (are files typically small or can they be large?)
  - Client complexity tolerance (is a two-step upload acceptable?)
  - Whether pre-upload file validation (beyond format/size) is needed
- Resolution criteria:
  - A single upload pattern is chosen and reflected in the architecture description

### Arch-0.2 — Worker deployment model: container-based vs serverless

- Type: DECISION
- Status: OPEN
- Related element(s):
  - Conversion Worker (30)
- Description:
  - The Conversion Worker can be deployed as a long-running containerized service polling the message broker, or as serverless functions triggered by queue events. This affects scalability characteristics, cost model, execution time limits, and operational complexity.
- Known options / hypotheses:
  - **Option A — Container-based workers**: Long-running containers consuming from the queue. Advantages: no execution time limits, full control over runtime environment (can install conversion libraries freely), better for large/long conversions. Disadvantages: requires capacity planning or auto-scaling configuration, may have idle cost.
  - **Option B — Serverless functions**: Functions triggered per message. Advantages: automatic scaling to zero, pay-per-invocation, no infrastructure management. Disadvantages: execution time limits (typically 15 minutes max), limited runtime environment (deployment package size limits), cold start latency.
  - **Option C — Hybrid**: Use serverless for quick conversions (small files, simple format pairs) and containers for heavy conversions. Advantages: cost-optimized. Disadvantages: added routing complexity.
- Required input / evidence:
  - Expected conversion duration range (seconds vs minutes)
  - Expected concurrency / traffic patterns (bursty vs steady)
  - Budget sensitivity (pay-per-use vs reserved capacity)
- Resolution criteria:
  - A deployment model is chosen for the initial release

### Arch-0.3 — Format Registry: embedded configuration vs standalone service

- Type: DECISION
- Status: OPEN
- Related element(s):
  - Format Registry (70), Conversion API Service (20), Conversion Worker (30)
- Description:
  - The Format Registry can be implemented as a shared configuration file/module embedded within the API Service and Worker, or as a standalone micro-service. This affects deployment coupling, update frequency, and operational overhead.
- Known options / hypotheses:
  - **Option A — Embedded configuration**: Format definitions are a shared config file or library bundled with Element 20 and Element 30. Advantages: simplest to implement, no extra service to deploy/monitor, fast development. Disadvantages: adding a new format requires redeploying both services.
  - **Option B — Standalone service**: A separate micro-service with its own API. Advantages: format changes don't require redeploying consumers, clean separation. Disadvantages: more infrastructure, network latency for lookups, overkill for a small initial format set.
- Required input / evidence:
  - Expected frequency of format additions
  - Whether format additions should be deployable independently of the core services
- Resolution criteria:
  - A deployment approach is chosen for the Format Registry

## Response template

Use the following template to record a response to any Arch item.
Append the response block directly under the item it answers.

```
#### Response — Arch-<N.N>

- Respondent: <human | agent>
- Answer:
  - <direct answer, decision, or information provided>
- Additional Details:
  - <Context, caveats etc.>
```
