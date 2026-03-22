# COMPONENTS DESCRIPTIONS

The response must provide a component-level design for **one parent architectural element**.

# PROBLEM STATEMENT

## Parent architectural element

- Parent ID:
  - 40
- Parent name:
  - Message Broker
- Parent purpose:
  - Decouples the API service from the conversion workers, enabling asynchronous processing, independent scaling, and independent deployment. Provides at-least-once delivery, dead-letter queuing, and native serverless function triggering.

## Assumptions
- AWS is the target cloud provider
- Managed services are preferred for fast development
- The message broker must natively support Lambda event-source mapping (serverless triggering)
- Messages carry only job metadata and references, never file content
- At-least-once delivery is acceptable; workers handle idempotency
- Current scope is text file conversions with seconds-to-minutes processing times
- Future extensibility to container-based workers consuming from the same queues is desired

## Constraints
- Must be fully managed (no self-hosted brokers) per the fast-development architectural preference
- Must support scale-to-zero and bursty traffic patterns
- Must natively trigger AWS Lambda functions (event-source mapping)
- Messages must not contain file content — only job metadata and storage references
- Dead-letter queue required for unprocessable messages
- Infrastructure defined via Terraform

## Component breakdown

### 40.1 — Conversion Request Queue

- Category:
  - queue
- Purpose:
  - Primary SQS queue that receives conversion-requested events from the Conversion API Service (Element 20) and triggers Conversion Worker Lambda functions (Element 30). This is the core decoupling mechanism between request acceptance and conversion execution.
- Technology choice:
  - Amazon SQS (Standard Queue)
- Responsibilities:
  - Accept conversion-requested messages published by Element 20
  - Buffer messages during traffic bursts to smooth load on workers
  - Trigger Conversion Worker Lambda functions via SQS event-source mapping
  - Manage message visibility timeout to prevent duplicate in-flight processing
  - Route unprocessable messages to the dead-letter queue (Component 40.2) after max receive count exceeded
- Interfaces:
  - Incoming:
    - `SendMessage` calls from Element 20 (Conversion API Service) containing job ID, source format, target format, conversion options, and file storage references
  - Outgoing:
    - SQS event-source mapping triggers Element 30 (Conversion Worker) Lambda functions
    - Failed messages forwarded to Component 40.2 (Dead-Letter Queue) via redrive policy
- Data / state:
  - Messages in transit containing job metadata (job ID, source format, target format, conversion options, S3 key references, timestamps)
  - No persistent business data — messages are transient
- Dependencies:
  - Internal:
    - 40.2 (Dead-Letter Queue — redrive policy target)
  - External:
    - Element 20 (producer), Element 30 (consumer via event-source mapping)
- Observability / operational considerations:
  - CloudWatch metrics: `ApproximateNumberOfMessagesVisible` (queue depth), `ApproximateAgeOfOldestMessage`, `NumberOfMessagesSent`, `NumberOfMessagesReceived`, `NumberOfMessagesDeleted`
  - Alarms on queue depth exceeding threshold (indicates workers not keeping up)
  - Alarms on age-of-oldest-message exceeding threshold (indicates processing stall)
- Constraints / notes:
  - Standard queue chosen over FIFO because strict ordering is not required for independent conversion jobs and standard queues offer higher throughput
  - Visibility timeout must be set to at least the Lambda function timeout plus a safety margin to avoid duplicate processing
  - Message retention period should be set to a reasonable window (e.g. 4 days) to handle extended outages
  - Batch size for Lambda event-source mapping should start at 1 for simplicity, tunable later for throughput optimization
- Principal alternative (optional):
  - Amazon SNS + SQS fan-out: would allow multiple subscribers per event, but adds complexity not needed in the current scope where only one consumer (Conversion Worker) exists. Can be introduced later if notification subscribers are added.

### 40.2 — Conversion Request Dead-Letter Queue

- Category:
  - queue
- Purpose:
  - Captures conversion-requested messages that repeatedly fail processing, preventing poison messages from blocking the main queue and enabling investigation of failures.
- Technology choice:
  - Amazon SQS (Standard Queue, configured as DLQ)
- Responsibilities:
  - Receive messages that exceed the max receive count on Component 40.1
  - Retain failed messages for a configurable period to allow manual inspection and replay
  - Serve as a source for operational alerting on processing failures
- Interfaces:
  - Incoming:
    - Messages redirected from Component 40.1 via SQS redrive policy
  - Outgoing:
    - Messages available for manual inspection, replay (via redrive-to-source), or programmatic consumption
- Data / state:
  - Failed conversion-requested messages with their original payload and SQS metadata (receive count, timestamps)
- Dependencies:
  - Internal:
    - 40.1 (source queue via redrive policy)
  - External:
    - None
- Observability / operational considerations:
  - CloudWatch alarm on `ApproximateNumberOfMessagesVisible > 0` — any message in the DLQ indicates a processing failure that needs attention
  - Dashboard widget showing DLQ depth over time to spot recurring failure patterns
  - Retention period should be longer than the main queue (e.g. 14 days) to allow investigation
- Constraints / notes:
  - Must be a Standard queue (matching the source queue type)
  - Max receive count on the source queue (Component 40.1) should be set to a reasonable number (e.g. 3) before redirecting to DLQ
  - SQS redrive-to-source feature can be used to replay messages back to the main queue after root cause is fixed

### 40.3 — Conversion Result Topic

- Category:
  - messaging / event bus
- Purpose:
  - Receives conversion-completed and conversion-failed events published by Conversion Workers (Element 30). Provides a fan-out point so that future subscribers (e.g., notification service, analytics pipeline) can consume these events without modifying the worker.
- Technology choice:
  - Amazon SNS (Standard Topic)
- Responsibilities:
  - Accept conversion-completed and conversion-failed event publications from Element 30
  - Fan out events to subscribed endpoints (currently none required; future-ready)
  - Provide message filtering capability so future subscribers can filter by event type or format
- Interfaces:
  - Incoming:
    - `Publish` calls from Element 30 (Conversion Worker) containing job ID, status (completed/failed), result file reference (if completed), error details (if failed), and timestamps
  - Outgoing:
    - Notifications delivered to subscribed endpoints (SQS queues, Lambda functions, HTTP endpoints) — currently no active subscriptions required
- Data / state:
  - Messages in transit only; SNS does not persist messages
  - No business data ownership
- Dependencies:
  - Internal:
    - None
  - External:
    - Element 30 (publisher)
- Observability / operational considerations:
  - CloudWatch metrics: `NumberOfMessagesPublished`, `NumberOfNotificationsDelivered`, `NumberOfNotificationsFailed`
  - Logging of publish failures
  - Message delivery status logging can be enabled for debugging
- Constraints / notes:
  - SNS chosen because the architecture description explicitly calls out that conversion-completed/failed events should be "available for any future subscriber" — SNS provides native fan-out to multiple subscribers
  - Standard topic (not FIFO) because ordering is not required and throughput should be maximized
  - In the current scope, this topic may have no active subscriptions. It exists to establish the event-publishing contract from workers, making it zero-effort to add subscribers later
  - If no subscribers exist, published messages are simply discarded by SNS — no cost or backlog concern
- Principal alternative (optional):
  - Amazon EventBridge: richer filtering and routing rules, but heavier setup and higher per-event cost. Overkill for the current simple fan-out need. Can be considered if complex routing rules emerge later.

### 40.4 — Queue-to-Lambda Event Source Mapping

- Category:
  - adapter / integration
- Purpose:
  - Configures the SQS-to-Lambda event-source mapping that triggers Conversion Worker functions when messages arrive in the Conversion Request Queue. This is the "glue" that makes the message broker trigger serverless compute.
- Technology choice:
  - AWS Lambda Event Source Mapping (SQS trigger), defined via Terraform (`aws_lambda_event_source_mapping` resource)
- Responsibilities:
  - Poll Component 40.1 (Conversion Request Queue) for new messages
  - Invoke Element 30 (Conversion Worker) Lambda function(s) with message batches
  - Manage concurrency scaling based on queue depth
  - Handle partial batch failure reporting so that only failed messages are retried
- Interfaces:
  - Incoming:
    - Messages from Component 40.1 (via SQS polling)
  - Outgoing:
    - Lambda invocations of Element 30 (Conversion Worker) with SQS event payload
- Data / state:
  - No data ownership — acts as a connector
- Dependencies:
  - Internal:
    - 40.1 (source queue)
  - External:
    - Element 30 (target Lambda function)
- Observability / operational considerations:
  - Lambda metrics: `Invocations`, `Errors`, `Throttles`, `ConcurrentExecutions` (on the worker function, correlated to this trigger)
  - Event source mapping state monitoring (enabled/disabled)
  - Partial batch failure reporting should be enabled (`FunctionResponseTypes: ["ReportBatchItemFailures"]`) to avoid reprocessing entire batches on single-message failures
- Constraints / notes:
  - Batch size should start at 1 for simplicity (one conversion job per invocation), tunable later
  - Maximum concurrency can be configured to cap parallel worker invocations if needed (e.g., to prevent downstream resource saturation)
  - The event source mapping is a Terraform-managed resource, separate from both the queue and the Lambda function, enabling independent configuration changes
  - Partial batch failure reporting is critical for efficiency — without it, a single failing message causes the entire batch to be retried

### 40.5 — Message Schema & Contract

- Category:
  - adapter / schema
- Purpose:
  - Defines the canonical JSON message schemas for all events flowing through the message broker, ensuring producers and consumers agree on payload structure. Serves as the contract between Element 20, Element 30, and the broker components.
- Technology choice:
  - JSON Schema definitions stored in the repository, validated at publish time in application code (using a lightweight JSON schema validation library, e.g., `ajv` for Node.js or `jsonschema` for Python)
- Responsibilities:
  - Define the schema for `conversion-requested` events (published by Element 20)
  - Define the schema for `conversion-completed` events (published by Element 30)
  - Define the schema for `conversion-failed` events (published by Element 30)
  - Provide schema validation utilities that producers use before publishing
  - Version schemas to support future evolution without breaking consumers
- Interfaces:
  - Incoming:
    - N/A (library/schema artifact, not a runtime service)
  - Outgoing:
    - Schemas consumed by Element 20 (producer) and Element 30 (producer/consumer) at build/deploy time and optionally at runtime for validation
- Data / state:
  - Schema definitions (JSON Schema files in the repository)
  - No runtime state
- Dependencies:
  - Internal:
    - None
  - External:
    - Consumed by Element 20 and Element 30
- Observability / operational considerations:
  - Schema validation failures should be logged with full message context in the producing service
  - Schema version tracked in message metadata to support forward/backward compatibility
- Constraints / notes:
  - Schemas must include at minimum: `jobId` (UUID), `sourceFormat`, `targetFormat`, `conversionOptions` (object), `sourceFileKey` (S3 reference), `timestamp`, and `schemaVersion`
  - Result event schemas add: `status` (completed/failed), `resultFileKey` (for completed), `errorMessage` and `errorCode` (for failed)
  - Keeping schemas in the repository (rather than a schema registry service) is sufficient for the current scope and aligns with fast-development priority
  - A shared schema registry service can be introduced later if the number of event types grows significantly

### 40.6 — Broker Infrastructure (Terraform Module)

- Category:
  - infrastructure / IaC
- Purpose:
  - Terraform module that provisions and configures all message broker AWS resources as a cohesive, independently deployable unit. Ensures all queue, topic, and integration resources are defined as code and can be deployed atomically.
- Technology choice:
  - Terraform (HCL), organized as a reusable module
- Responsibilities:
  - Provision the Conversion Request Queue (Component 40.1) with appropriate settings (visibility timeout, message retention, etc.)
  - Provision the Dead-Letter Queue (Component 40.2) with redrive policy linking to 40.1
  - Provision the Conversion Result Topic (Component 40.3)
  - Configure IAM policies granting Element 20 publish access to 40.1, Element 30 consume access from 40.1 and publish access to 40.3
  - Configure the Lambda event-source mapping (Component 40.4) — or expose the queue ARN for the Lambda module to configure it
  - Set up CloudWatch alarms for queue depth, DLQ messages, and oldest message age
  - Export resource ARNs and URLs as outputs for consumption by other Terraform modules (Elements 20, 30)
- Interfaces:
  - Incoming:
    - Terraform variables: Lambda function ARN (for event-source mapping), alarm SNS topic ARN (for alerts), environment tag, retention periods, visibility timeout, max receive count
  - Outgoing:
    - Terraform outputs: queue URL, queue ARN, DLQ URL, DLQ ARN, SNS topic ARN, IAM policy ARNs
- Data / state:
  - Terraform state (managed externally, e.g., S3 backend)
- Dependencies:
  - Internal:
    - Provisions 40.1, 40.2, 40.3, 40.4
  - External:
    - Requires Element 30 Lambda function ARN as input (for event-source mapping)
    - Outputs consumed by Element 20 and Element 30 Terraform modules
- Observability / operational considerations:
  - Terraform plan/apply output for change review
  - CloudWatch alarms defined within this module for all broker-related operational concerns
  - Tagging strategy for cost allocation and resource identification
- Constraints / notes:
  - Module should be self-contained and independently deployable (atomic deployment per architectural preference)
  - All configurable values (timeouts, retention, thresholds) exposed as variables with sensible defaults
  - The event-source mapping (40.4) may alternatively be defined in the Conversion Worker's Terraform module if that simplifies dependency management — this is a deployment-time decision to be resolved during implementation

## Open questions

- Should the Conversion Result Topic (40.3) have any active subscribers in the initial release, or should it be provisioned empty as a future extension point?
- What should the max receive count be before messages are sent to the DLQ? (Proposed: 3)
- Should the Lambda event-source mapping batch size start at 1, or should a small batch (e.g., 5-10) be used from the start for throughput?
- Should the event-source mapping (40.4) be owned by the Message Broker Terraform module (40.6) or by the Conversion Worker's Terraform module? Both approaches are valid — co-locating with the broker keeps messaging concerns together, while co-locating with the worker keeps trigger configuration close to the consumer.
