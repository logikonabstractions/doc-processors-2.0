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

## Assumptions

- Everything needs to be hosted on AWS
- We favor FAST DEVELOPMENT over other priorities. This means:
  - favouring using managed, hosted services when they fill a need over writing our own code
- IaC: we will use terraform to manage & deploy the cloud infrastructure
- We will first focus on text file management only (keeping in mind the need for the structure to be able to expand to other file types)
- Files are ephemeral: uploaded files and conversion results are retained only long enough for the user to retrieve them (time-limited)
- Individual file sizes are bounded (a maximum upload size will be enforced)

## Architectural elements

### 10 — API Gateway

- Category:
  - orchestration
- Purpose:
  - Single entry point for all client requests. Handles routing, request validation, rate limiting, and CORS.
- Responsibilities:
  - Accept incoming HTTP requests from anonymous web users
  - Route requests to the appropriate back-end service
  - Enforce rate limiting and throttling per client IP
  - Enforce maximum request/upload size
  - Provide CORS headers for browser-based clients
- Interfaces:
  - Incoming:
    - HTTP requests from clients (upload file, check job status, download result, list supported formats)
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

### 20 — Conversion API Service

- Category:
  - domain service
- Purpose:
  - Handles business logic for conversion requests: validates inputs, creates conversion jobs, and serves results.
- Responsibilities:
  - Validate uploaded file type and conversion target format
  - Generate a pre-signed upload URL or accept file upload and store it in temporary file storage (Element 50)
  - Create a conversion job record in the Job Metadata Store (Element 60)
  - Publish a conversion-requested event to the Message Broker (Element 40)
  - Serve job status queries
  - Generate a pre-signed download URL for completed conversions
  - Return the list of supported source/target format combinations
- Interfaces:
  - Incoming:
    - HTTP requests from API Gateway (Element 10)
  - Outgoing:
    - Write job metadata to Element 60 (Job Metadata Store)
    - Upload source file to Element 50 (Temporary File Storage)
    - Publish conversion-requested event to Element 40 (Message Broker)
    - Read job status from Element 60
    - Generate download URLs from Element 50
- Data / state:
  - Owns conversion job lifecycle logic but delegates persistence to Element 60.
  - Does not store files itself; delegates to Element 50.
- Interactions:
  - User-facing:
    - Indirectly via API Gateway
  - Internal synchronous:
    - Reads/writes to Element 60 (Job Metadata Store)
    - Reads/writes to Element 50 (Temporary File Storage)
  - Internal asynchronous:
    - Publishes events to Element 40 (Message Broker)
- Security / access considerations:
  - Input validation: file type, size, format compatibility
  - Pre-signed URLs ensure users can only access their own files for a limited time
- Observability / operational considerations:
  - Structured logging of each request with job ID correlation
  - Metrics: request count, error rate, latency per endpoint
  - Health check endpoint
- Dependencies:
  - 40, 50, 60
- Constraints / notes:
  - Should be stateless and horizontally scalable (runs as a containerized service or serverless function).

### 30 — Conversion Worker

- Category:
  - domain service
- Purpose:
  - Performs the actual file conversion. Pulls jobs from the message broker, retrieves the source file, executes the conversion, and stores the result.
- Responsibilities:
  - Consume conversion-requested events from the Message Broker (Element 40)
  - Retrieve the source file from Temporary File Storage (Element 50)
  - Determine the appropriate conversion strategy based on source/target format pair
  - Execute the conversion
  - Store the converted file in Temporary File Storage (Element 50)
  - Update job status (success/failure) in Job Metadata Store (Element 60)
  - Publish a conversion-completed (or conversion-failed) event to the Message Broker (Element 40)
- Interfaces:
  - Incoming:
    - Conversion-requested events from Element 40 (Message Broker)
  - Outgoing:
    - Read source file from Element 50 (Temporary File Storage)
    - Write converted file to Element 50 (Temporary File Storage)
    - Update job record in Element 60 (Job Metadata Store)
    - Publish conversion-completed / conversion-failed event to Element 40 (Message Broker)
- Data / state:
  - Transient only: holds file data in memory or local temp disk during conversion.
  - Owns no persistent state.
- Interactions:
  - User-facing:
    - None
  - Internal synchronous:
    - Reads/writes Element 50 (Temporary File Storage)
    - Reads/writes Element 60 (Job Metadata Store)
  - Internal asynchronous:
    - Consumes from and publishes to Element 40 (Message Broker)
- Security / access considerations:
  - Runs in a private subnet; no direct internet access.
  - File content should be treated as untrusted (potential malicious payloads in uploaded documents).
  - Conversion processes should be sandboxed or resource-limited to prevent resource exhaustion from malformed files.
- Observability / operational considerations:
  - Per-job structured logging with job ID, source format, target format, duration, outcome
  - Metrics: conversion duration histogram, success/failure rate by format pair, queue depth
  - Dead-letter queue for repeatedly failing jobs
  - Alerting on sustained failure rates or queue backlog
- Dependencies:
  - 40, 50, 60
- Constraints / notes:
  - Must be designed with a plugin/strategy pattern so new format converters can be added without modifying the core worker logic. This is key for future extensibility to images, geo, video, etc.
  - Workers should be independently scalable based on queue depth.
- Principal alternative (optional)
  - Instead of a long-running worker consuming from a queue, a serverless function-based approach could be used (triggered by queue events). Trade-off: serverless functions have execution time limits which may be problematic for large file conversions; a container-based worker offers more control over resource limits and execution duration.

### 40 — Message Broker

- Category:
  - messaging
- Purpose:
  - Decouples the API service from the conversion workers, enabling asynchronous processing and independent scaling.
- Responsibilities:
  - Accept conversion-requested events from the Conversion API Service (Element 20)
  - Deliver events to Conversion Workers (Element 30)
  - Accept conversion-completed / conversion-failed events from workers
  - Provide dead-letter queue for unprocessable messages
  - Guarantee at-least-once delivery
- Interfaces:
  - Incoming:
    - Conversion-requested events from Element 20
    - Conversion-completed / conversion-failed events from Element 30
  - Outgoing:
    - Conversion-requested events to Element 30
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
    - Core async backbone connecting Element 20 and Element 30
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
  - At-least-once delivery means workers must handle duplicate messages idempotently.

### 50 — Temporary File Storage

- Category:
  - data persistence
- Purpose:
  - Stores uploaded source files and conversion result files for short-term retrieval.
- Responsibilities:
  - Accept file uploads (source files from Element 20, result files from Element 30)
  - Serve file downloads via pre-signed, time-limited URLs
  - Automatically expire and delete files after a configurable retention period
  - Organize files by job ID for easy cleanup
- Interfaces:
  - Incoming:
    - File writes from Element 20 (source upload) and Element 30 (conversion result)
    - Pre-signed URL generation requests from Element 20
  - Outgoing:
    - File reads to Element 30 (source retrieval)
    - Pre-signed download URLs to Element 20 (for client retrieval)
- Data / state:
  - Source files (uploaded by users)
  - Converted result files (produced by workers)
  - All data is ephemeral with automatic expiry (lifecycle policy)
- Interactions:
  - User-facing:
    - Users upload/download files via pre-signed URLs (direct-to-storage pattern)
  - Internal synchronous:
    - Element 20 and Element 30 read/write files
  - Internal asynchronous:
    - Lifecycle policies automatically clean up expired files
- Security / access considerations:
  - No public listing of stored files
  - All access via pre-signed URLs with short expiry
  - Encryption at rest
  - Files should be scanned or treated as untrusted
- Observability / operational considerations:
  - Storage usage monitoring
  - Lifecycle policy execution monitoring
  - Failed upload/download alerts
- Dependencies:
  - None (infrastructure element)
- Constraints / notes:
  - Must be a managed object storage service to align with fast-development priority.
  - Maximum file size enforced at this layer as well as at the API Gateway.

### 60 — Job Metadata Store

- Category:
  - data persistence
- Purpose:
  - Persists conversion job metadata: status, file references, timestamps, conversion parameters, and error details.
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

### 70 — Format Registry

- Category:
  - domain service
- Purpose:
  - Centralizes knowledge of supported file formats, valid conversion paths (source → target), and conversion options available for each path.
- Responsibilities:
  - Maintain a registry of supported source formats and target formats
  - Define which source-to-target conversion paths are valid
  - Define available conversion options/preferences per conversion path
  - Expose this information to Element 20 (for input validation and user-facing format listing) and Element 30 (for conversion strategy selection)
- Interfaces:
  - Incoming:
    - Queries from Element 20 and Element 30: "is this conversion valid?", "what options are available?", "list all supported formats"
  - Outgoing:
    - Format metadata and validation results
- Data / state:
  - Format definitions, conversion path matrix, option schemas
  - This data changes rarely (only when new formats are added). Can be configuration-driven.
- Interactions:
  - User-facing:
    - Indirectly: Element 20 exposes format listing based on this registry
  - Internal synchronous:
    - Queried by Element 20 and Element 30
  - Internal asynchronous:
    - N/A
- Security / access considerations:
  - Read-only for all consumers. Write access is an admin/deploy-time concern.
- Observability / operational considerations:
  - Version tracking of format registry changes
  - Logging of unsupported format requests (useful for prioritizing new format development)
- Dependencies:
  - None
- Constraints / notes:
  - This can start as a static configuration file or module embedded within Element 20 and Element 30, rather than a separate deployed service. It is called out as a distinct architectural element because it represents a clear responsibility boundary that will grow as formats expand.

## System interaction summary

- Primary request / control paths:
  - Client → API Gateway (10) → Conversion API Service (20): submit conversion request, check status, download result
  - Conversion API Service (20) → Message Broker (40) → Conversion Worker (30): dispatch and process conversion jobs
- Primary data flows:
  - Upload: Client → API Gateway (10) → Conversion API Service (20) → Temporary File Storage (50) (or Client direct-to-storage via pre-signed URL)
  - Conversion: Conversion Worker (30) reads source from Temporary File Storage (50), converts, writes result to Temporary File Storage (50)
  - Download: Client requests download URL from Conversion API Service (20), receives pre-signed URL, downloads directly from Temporary File Storage (50)
- Primary event flows:
  - Conversion API Service (20) publishes `conversion-requested` → Message Broker (40) → Conversion Worker (30) consumes
  - Conversion Worker (30) publishes `conversion-completed` or `conversion-failed` → Message Broker (40) → (available for future subscribers)

## System-wide concerns

- Security and access control:
  - All users are anonymous; no authentication required
  - Rate limiting at the API Gateway to prevent abuse
  - Pre-signed, time-limited URLs for file access
  - All file content treated as untrusted; conversion processes should be sandboxed
  - TLS for all external traffic
  - Internal services communicate over a private network
  - Non-guessable job IDs (UUIDs) prevent unauthorized status/result access
- Reliability and recovery:
  - At-least-once message delivery with idempotent workers
  - Dead-letter queue for failed conversion jobs
  - Automatic retries on transient failures (configurable retry count)
  - File storage and metadata store use managed services with built-in durability
  - Health checks on all services for automated recovery
- Observability and operations:
  - Structured logging with job ID correlation across all services
  - Centralized log aggregation
  - Metrics dashboards: request rates, conversion durations, error rates, queue depths
  - Alerting on error rate spikes, queue backlog, storage thresholds
  - Dead-letter queue monitoring
- Performance and scalability:
  - Workers scale horizontally based on queue depth
  - API service is stateless and horizontally scalable
  - Direct-to-storage file transfer avoids API service as a bottleneck for large files
  - File size limits prevent resource exhaustion
  - Conversion timeout enforcement prevents runaway jobs
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
- How should the system handle conversions that exceed a time limit — cancel and fail, or allow extended processing?
