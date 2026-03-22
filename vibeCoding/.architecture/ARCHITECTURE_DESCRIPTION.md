# ARCHITECTURE DESCRIPTION


# PROBLEM STATEMENT

## Objective

- System:
  - We want to design a micro-service architecture that allows file conversions for a host of files (all types of text files such as PDF, .docx, .md, etc.). The system should be easy to enlarge to other file types eventually (such as images, geo files, videos, etc.)
- Users / actors:
  - Anonymous users on the web
- Primary outcome:
  - Users should be able to send their files to the system
  - Users should be able to retrieve the converted file
  - It must be possible to provide conversion options & preferences

## Scope boundaries

- In scope:
  - Text-based file conversions (PDF, .docx, .md, .txt, .html, .rtf, .csv, .xlsx, etc.)
  - Upload and download of files via a web-facing API
  - Conversion options and preferences per request
  - Asynchronous job processing for conversions
  - Job status tracking and result retrieval
  - Infrastructure-as-code for all cloud resources
- Out of scope:
  - User authentication and account management (anonymous access only)
  - Image, geo, video conversions (future expansion, but not in this iteration)
  - Real-time collaborative editing
  - Long-term file archival or user-owned storage
  - Billing / metering
  - Long-running conversion processes (future iteration; current scope is seconds-to-minutes)

## Assumptions

- Everything needs to be hosted on AWS
- We favor FAST DEVELOPMENT over other priorities. This means:
  - favouring using managed, hosted services when they fill a need over writing our own code
- IaC: we will use terraform to manage & deploy the cloud infrastructure
- We will first focus on text file management only (keeping in mind the need for the structure to be able to expand to other file types)
- Files are ephemeral: uploaded files and conversion results are retained only long enough for the user to retrieve them (time-limited)
- Individual file sizes are bounded (a maximum upload size will be enforced)

## Architectural preferences

- **Maximal decoupling**: each service should be independently deployable. Atomic deployments are strongly preferred.
- **Serverless-first**: all compute elements should be serverless functions to enable scale-to-zero, pay-per-invocation, and instant scale-up. No reserved capacity.
- **Minimal storage**: storage is acceptable only when ephemeral and when it serves decoupling or scaling needs. No persistent data stores beyond what is required for job coordination.
- **Unpredictable traffic**: the system must handle bursty, unpredictable usage patterns with rapid scale-up and scale-to-zero.
- **Future extensibility**: choices should keep options open for longer-running conversion processes (e.g., container-based workers) in future iterations, even though the current scope is seconds-to-minutes.

## Architectural elements

### 10 — API Gateway

- Category:
  - orchestration
- Purpose:
  - Single entry point for all client requests. Handles routing, request validation, rate limiting, and CORS.
- Responsibilities:
  - Accept incoming HTTP requests from anonymous web users
  - Route requests to the appropriate back-end serverless function
  - Enforce rate limiting and throttling per client IP
  - Enforce maximum request size
  - Provide CORS headers for browser-based clients
- Interfaces:
  - Incoming:
    - HTTP requests from clients (request pre-signed upload URL, check job status, request download URL, list supported formats)
  - Outgoing:
    - Proxied requests to Conversion API Service (Element 20)
- Data / state:
  - Stateless. No persistent data. May cache route configurations.
- Interactions:
  - User-facing:
    - All user traffic enters through this element
  - Internal synchronous:
    - Forwards requests to Element 20 (Conversion API Service)
  - Internal asynchronous:
    - N/A
- Security / access considerations:
  - Trust boundary: this is the outermost edge. All input must be treated as untrusted.
  - Rate limiting prevents abuse from anonymous users.
  - TLS termination happens here.
- Observability / operational considerations:
  - Request logging (method, path, status code, latency)
  - Rate-limit breach alerts
  - Availability and error-rate monitoring
- Dependencies:
  - 20
- Constraints / notes:
  - Must be a managed service to align with fast-development priority.
  - Does not handle file transfer — files go directly to Temporary File Storage (Element 50) via pre-signed URLs.

### 20 — Conversion API Service

- Category:
  - domain service
- Purpose:
  - Handles business logic for conversion requests: validates inputs, creates conversion jobs, issues pre-signed URLs, and serves job status/results. Deployed as serverless functions.
- Responsibilities:
  - Validate requested conversion (source format, target format, options) against the Format Registry (Element 70)
  - Generate a pre-signed upload URL for the client to upload directly to Temporary File Storage (Element 50)
  - Create a conversion job record in the Job Metadata Store (Element 60)
  - Publish a conversion-requested event to the Message Broker (Element 40)
  - Serve job status queries
  - Generate a pre-signed download URL for completed conversions
  - Return the list of supported source/target format combinations (via Element 70)
- Interfaces:
  - Incoming:
    - HTTP requests from API Gateway (Element 10)
  - Outgoing:
    - Write job metadata to Element 60 (Job Metadata Store)
    - Generate pre-signed upload/download URLs for Element 50 (Temporary File Storage)
    - Publish conversion-requested event to Element 40 (Message Broker)
    - Read job status from Element 60
    - Query format information from Element 70 (Format Registry)
- Data / state:
  - Owns conversion job lifecycle logic but delegates persistence to Element 60.
  - Does not store or proxy files — delegates to Element 50 via pre-signed URLs.
- Interactions:
  - User-facing:
    - Indirectly via API Gateway
  - Internal synchronous:
    - Reads/writes to Element 60 (Job Metadata Store)
    - Generates pre-signed URLs for Element 50 (Temporary File Storage)
    - Queries Element 70 (Format Registry)
  - Internal asynchronous:
    - Publishes events to Element 40 (Message Broker)
- Security / access considerations:
  - Input validation: file type, size, format compatibility
  - Pre-signed URLs ensure users can only access their own files for a limited time
- Observability / operational considerations:
  - Structured logging of each request with job ID correlation
  - Metrics: request count, error rate, latency per endpoint
- Dependencies:
  - 40, 50, 60, 70
- Constraints / notes:
  - Deployed as serverless functions — stateless, scales to zero, instant scale-up.
  - Each API route can be an independent function for maximum deployment atomicity.
  - Never proxies file content; all file transfer is direct-to-storage via pre-signed URLs.

### 30 — Conversion Worker

- Category:
  - domain service
- Purpose:
  - Performs the actual file conversion. Triggered by queue events, retrieves the source file, executes the conversion, and stores the result. Deployed as serverless functions.
- Responsibilities:
  - Consume conversion-requested events from the Message Broker (Element 40)
  - Retrieve the source file from Temporary File Storage (Element 50)
  - Determine the appropriate conversion strategy based on source/target format pair (via Element 70)
  - Execute the conversion
  - Store the converted file in Temporary File Storage (Element 50)
  - Update job status (success/failure) in Job Metadata Store (Element 60)
  - Publish a conversion-completed (or conversion-failed) event to the Message Broker (Element 40)
- Interfaces:
  - Incoming:
    - Conversion-requested events from Element 40 (Message Broker) — triggers function invocation
  - Outgoing:
    - Read source file from Element 50 (Temporary File Storage)
    - Write converted file to Element 50 (Temporary File Storage)
    - Update job record in Element 60 (Job Metadata Store)
    - Publish conversion-completed / conversion-failed event to Element 40 (Message Broker)
    - Query format/strategy information from Element 70 (Format Registry)
- Data / state:
  - Transient only: holds file data in memory during conversion within a single function invocation.
  - Owns no persistent state.
- Interactions:
  - User-facing:
    - None
  - Internal synchronous:
    - Reads/writes Element 50 (Temporary File Storage)
    - Reads/writes Element 60 (Job Metadata Store)
    - Queries Element 70 (Format Registry)
  - Internal asynchronous:
    - Triggered by and publishes to Element 40 (Message Broker)
- Security / access considerations:
  - Runs in a private network; no direct internet access.
  - File content should be treated as untrusted (potential malicious payloads in uploaded documents).
  - Serverless function execution is inherently sandboxed per invocation, providing isolation.
  - Memory and execution time limits provide natural resource bounding.
- Observability / operational considerations:
  - Per-job structured logging with job ID, source format, target format, duration, outcome
  - Metrics: conversion duration histogram, success/failure rate by format pair, queue depth
  - Dead-letter queue for repeatedly failing jobs
  - Alerting on sustained failure rates or queue backlog
- Dependencies:
  - 40, 50, 60, 70
- Constraints / notes:
  - Deployed as serverless functions triggered by queue events. Scales automatically based on queue depth; scales to zero when idle.
  - Serverless execution time limits (typically up to 15 minutes) are acceptable for the current scope (seconds-to-minutes conversions).
  - Must be designed with a plugin/strategy pattern so new format converters can be added without modifying the core worker logic. This is key for future extensibility.
  - **Future extensibility**: if conversions eventually exceed serverless time limits, the architecture can be extended with container-based workers consuming from the same queue, selected via routing logic in the Message Broker. The queue-based decoupling makes this a non-breaking change.

### 40 — Message Broker

- Category:
  - messaging
- Purpose:
  - Decouples the API service from the conversion workers, enabling asynchronous processing, independent scaling, and independent deployment.
- Responsibilities:
  - Accept conversion-requested events from the Conversion API Service (Element 20)
  - Trigger Conversion Worker functions (Element 30) on message arrival
  - Accept conversion-completed / conversion-failed events from workers
  - Provide dead-letter queue for unprocessable messages
  - Guarantee at-least-once delivery
- Interfaces:
  - Incoming:
    - Conversion-requested events from Element 20
    - Conversion-completed / conversion-failed events from Element 30
  - Outgoing:
    - Conversion-requested events trigger Element 30
    - Conversion-completed / conversion-failed events (available for any future subscriber, e.g., notification service)
- Data / state:
  - Messages in transit. Does not own business data.
  - Dead-letter queue retains failed messages for inspection.
- Interactions:
  - User-facing:
    - None
  - Internal synchronous:
    - N/A
  - Internal asynchronous:
    - Core async backbone connecting Element 20 and Element 30. Serves as the serverless function trigger.
- Security / access considerations:
  - Access restricted to internal services only (private network)
  - Messages should not contain file content, only job metadata and references
- Observability / operational considerations:
  - Queue depth and age-of-oldest-message monitoring
  - Dead-letter queue size alerting
  - Message throughput metrics
- Dependencies:
  - None (infrastructure element)
- Constraints / notes:
  - Must be a managed service to align with fast-development priority.
  - Must natively support serverless function triggering (event-source mapping).
  - At-least-once delivery means workers must handle duplicate messages idempotently.
  - **Future extensibility**: this same queue can be consumed by container-based workers if long-running conversions are added later, using message attributes or separate queues for routing.

### 50 — Temporary File Storage

- Category:
  - data persistence
- Purpose:
  - Ephemeral storage for uploaded source files and conversion result files. Exists solely to decouple the upload/download flow from the conversion process and to bridge the gap between the stateless serverless functions.
- Responsibilities:
  - Accept file uploads directly from clients via pre-signed URLs (source files)
  - Accept file writes from Conversion Workers (result files)
  - Serve file downloads via pre-signed, time-limited URLs
  - Automatically expire and delete files after a configurable retention period
  - Organize files by job ID for easy cleanup
- Interfaces:
  - Incoming:
    - File uploads directly from clients via pre-signed URLs
    - File writes from Element 30 (Conversion Worker — result files)
    - Pre-signed URL generation requests from Element 20 (Conversion API Service)
  - Outgoing:
    - File reads to Element 30 (source retrieval)
    - Pre-signed download URLs to clients (via Element 20)
- Data / state:
  - Source files (uploaded by users)
  - Converted result files (produced by workers)
  - All data is ephemeral with automatic expiry (lifecycle policy)
- Interactions:
  - User-facing:
    - Users upload source files and download result files directly via pre-signed URLs (direct-to-storage pattern)
  - Internal synchronous:
    - Element 30 reads source files and writes result files
    - Element 20 generates pre-signed URLs
  - Internal asynchronous:
    - Lifecycle policies automatically clean up expired files
- Security / access considerations:
  - No public listing of stored files
  - All access via pre-signed URLs with short expiry
  - Encryption at rest
  - Files should be treated as untrusted
- Observability / operational considerations:
  - Storage usage monitoring
  - Lifecycle policy execution monitoring
  - Failed upload/download alerts
- Dependencies:
  - None (infrastructure element)
- Constraints / notes:
  - Must be a managed object storage service to align with fast-development priority.
  - Maximum file size enforced at this layer as well as at the API Gateway.
  - This is the only storage in the system; it is strictly ephemeral. Files are auto-deleted after a short retention window.

### 60 — Job Metadata Store

- Category:
  - data persistence
- Purpose:
  - Persists conversion job metadata: status, file references, timestamps, conversion parameters, and error details. Ephemeral — records auto-expire with TTL.
- Responsibilities:
  - Store and retrieve job records (job ID, status, source format, target format, conversion options, timestamps, error info)
  - Support queries by job ID
  - Support TTL-based automatic expiry of old job records (aligned with file retention)
- Interfaces:
  - Incoming:
    - Create/update job records from Element 20 (Conversion API Service) and Element 30 (Conversion Worker)
    - Query job status from Element 20
  - Outgoing:
    - Job records returned to querying services
- Data / state:
  - Job records with fields: job ID, status (pending, in-progress, completed, failed), source file reference, result file reference, source format, target format, conversion options, created-at, updated-at, error message
  - All records are ephemeral with TTL-based auto-expiry
- Interactions:
  - User-facing:
    - None (accessed indirectly via Element 20)
  - Internal synchronous:
    - Element 20 and Element 30 read/write job records
  - Internal asynchronous:
    - TTL-based automatic record expiry
- Security / access considerations:
  - Access restricted to internal services only
  - No sensitive user data stored (anonymous users)
  - Job IDs should be non-guessable (UUIDs)
- Observability / operational considerations:
  - Read/write latency monitoring
  - Storage capacity monitoring
  - TTL expiry metrics
- Dependencies:
  - None (infrastructure element)
- Constraints / notes:
  - A managed key-value or document store is appropriate given the simple access pattern (lookup by job ID) and the need for fast development.
  - Must support TTL-based item expiry natively.
  - Serverless-compatible: must be accessible from serverless functions without connection pooling concerns. Prefer HTTP-based APIs over persistent connections.

### 70 — Format Registry

- Category:
  - domain service
- Purpose:
  - Standalone service that centralizes knowledge of supported file formats, valid conversion paths (source → target), and conversion options available for each path. Independently deployable from all other services.
- Responsibilities:
  - Maintain a registry of supported source formats and target formats
  - Define which source-to-target conversion paths are valid
  - Define available conversion options/preferences per conversion path
  - Expose this information via a read-only API to Element 20 (for input validation and user-facing format listing) and Element 30 (for conversion strategy selection)
- Interfaces:
  - Incoming:
    - HTTP/API queries from Element 20 and Element 30: "is this conversion valid?", "what options are available?", "list all supported formats"
  - Outgoing:
    - Format metadata and validation results (JSON responses)
- Data / state:
  - Format definitions, conversion path matrix, option schemas
  - Backed by a managed key-value or document store for persistence
  - This data changes rarely (only when new formats are added)
- Interactions:
  - User-facing:
    - Indirectly: Element 20 exposes format listing based on this registry
  - Internal synchronous:
    - Queried by Element 20 and Element 30 via API
  - Internal asynchronous:
    - N/A
- Security / access considerations:
  - Read-only API for all consumer services. Write access is an admin/deploy-time concern (separate admin endpoint or direct data store updates).
- Observability / operational considerations:
  - Version tracking of format registry changes
  - Logging of unsupported format requests (useful for prioritizing new format development)
  - Cache-friendly: responses can be aggressively cached by consumers since data changes rarely
- Dependencies:
  - None
- Constraints / notes:
  - Deployed as an independent serverless function backed by a managed key-value store. This keeps it lightweight while achieving full deployment independence.
  - Adding a new format requires only updating the Format Registry's data store and (if needed) deploying a new conversion strategy to Element 30 — no changes to Element 20.
  - Consumers should cache registry responses to minimize cross-service calls. A short TTL cache (minutes) is sufficient given the low change frequency.

## System interaction summary

- Primary request / control paths:
  - Client → API Gateway (10) → Conversion API Service (20): request pre-signed upload URL, submit conversion job, check status, request download URL
  - Conversion API Service (20) → Message Broker (40) → Conversion Worker (30): dispatch and process conversion jobs
  - Conversion API Service (20) → Format Registry (70): validate formats, list supported conversions
- Primary data flows:
  - Upload: Client requests pre-signed upload URL from Conversion API Service (20) → uploads file directly to Temporary File Storage (50)
  - Conversion: Conversion Worker (30) reads source from Temporary File Storage (50), converts, writes result to Temporary File Storage (50)
  - Download: Client requests download URL from Conversion API Service (20), receives pre-signed URL, downloads directly from Temporary File Storage (50)
- Primary event flows:
  - Conversion API Service (20) publishes `conversion-requested` → Message Broker (40) → triggers Conversion Worker (30)
  - Conversion Worker (30) publishes `conversion-completed` or `conversion-failed` → Message Broker (40) → (available for future subscribers)

## System-wide concerns

- Security and access control:
  - All users are anonymous; no authentication required
  - Rate limiting at the API Gateway to prevent abuse
  - Pre-signed, time-limited URLs for all file access (upload and download)
  - All file content treated as untrusted; serverless function sandboxing provides per-invocation isolation
  - TLS for all external traffic
  - Internal services communicate over a private network
  - Non-guessable job IDs (UUIDs) prevent unauthorized status/result access
- Reliability and recovery:
  - At-least-once message delivery with idempotent workers
  - Dead-letter queue for failed conversion jobs
  - Automatic retries on transient failures (configurable retry count)
  - File storage and metadata store use managed services with built-in durability
  - Serverless functions are inherently stateless — failed invocations are retried by the platform
- Observability and operations:
  - Structured logging with job ID correlation across all services
  - Centralized log aggregation
  - Metrics dashboards: request rates, conversion durations, error rates, queue depths
  - Alerting on error rate spikes, queue backlog, storage thresholds
  - Dead-letter queue monitoring
- Performance and scalability:
  - All compute is serverless: scales to zero when idle, scales up instantly on demand
  - No reserved capacity; purely event-driven scaling
  - Direct-to-storage file transfer avoids compute as a bottleneck for file transfers
  - File size limits prevent resource exhaustion
  - Conversion timeout enforcement via serverless function time limits
  - Format Registry responses are cache-friendly, minimizing cross-service latency
- Compliance / audit / governance:
  - Ephemeral file storage with automatic expiry reduces data retention risk
  - No personally identifiable information stored (anonymous users)
  - Infrastructure defined as code (terraform) for auditability and reproducibility

## Open questions
- What is the maximum file upload size to enforce?
- What is the file and job metadata retention period before automatic cleanup?
- Should the system support batch conversions (multiple files in one request) or single-file only?
- What specific text file formats should be supported in the initial release, and which conversion paths are valid (e.g., can every format convert to every other format, or only specific paths)?
- Should there be a notification mechanism (e.g., webhook callback) when a conversion completes, or is polling the only retrieval method?
