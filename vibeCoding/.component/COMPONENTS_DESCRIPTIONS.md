# COMPONENTS DESCRIPTIONS

The response must provide a component-level design for **one parent architectural element**.

# PROBLEM STATEMENT

## Parent architectural element

- Parent ID:
  - 70
- Parent name:
  - Format Registry
- Parent purpose:
  - Standalone service that centralizes knowledge of supported file formats, valid conversion paths (source → target), and conversion options available for each path. Independently deployable from all other services.

## Assumptions
- The registry data changes rarely — only when new formats or conversion paths are added.
- Consumers (Elements 20 and 30) will cache responses with a short TTL, so the registry is not on the hot path for every request.
- Write access (adding/updating formats) is an admin/deploy-time concern, not a runtime user operation.
- The initial scope covers text-based file formats only (PDF, .docx, .md, .txt, .html, .rtf, .csv, .xlsx, etc.).
- AWS-hosted, serverless-first, managed services preferred.

## Constraints
- Must be independently deployable (atomic deployments).
- Serverless compute — scales to zero, no reserved capacity.
- Backed by a managed key-value or document store.
- Read-only API for consumer services; write access is admin-only.
- Adding a new format must not require changes to Element 20 (Conversion API Service).
- Responses must be cache-friendly to minimize cross-service call overhead.

## Component breakdown

### 70.1 — Format Registry API

- Category:
  - API service
- Purpose:
  - Exposes a read-only HTTP API that lets consumer services (Elements 20 and 30) query supported formats, validate conversion paths, and retrieve available conversion options per path.
- Technology choice:
  - AWS Lambda (Node.js runtime) behind an internal API Gateway (or Lambda Function URL for internal-only access). Deployed via Terraform.
- Responsibilities:
  - Serve `GET /formats` — list all supported source and target formats
  - Serve `GET /formats/{sourceFormat}/targets` — list valid target formats for a given source format
  - Serve `GET /conversions/{sourceFormat}/{targetFormat}` — validate a conversion path and return available options/preferences schema
  - Serve `GET /conversions/{sourceFormat}/{targetFormat}/options` — return the detailed options schema for a specific conversion path
  - Return structured JSON responses with appropriate HTTP status codes and cache-control headers
  - Delegate all data retrieval to the Format Data Store (70.2)
- Interfaces:
  - Incoming:
    - HTTP requests from Element 20 (Conversion API Service) — format listing, conversion validation, options retrieval
    - HTTP requests from Element 30 (Conversion Worker) — conversion strategy/options lookup
  - Outgoing:
    - Read queries to Format Data Store (70.2)
- Data / state:
  - Stateless. All data is read from the Format Data Store (70.2) at request time (or from an in-memory cache within the Lambda execution context for warm invocations).
- Dependencies:
  - Internal:
    - 70.2 (Format Data Store)
  - External:
    - None
- Observability / operational considerations:
  - Structured logging: each request logged with format parameters, response status, latency
  - Logging of requests for unsupported formats (useful for prioritizing new format development)
  - CloudWatch metrics: invocation count, error rate, latency (p50/p95/p99)
  - Cache-Control headers on responses (e.g., `max-age=300`) to allow consumer-side caching
- Constraints / notes:
  - Purely read-only at runtime. No mutation endpoints exposed to consumer services.
  - Lambda in-memory caching (loading full registry on cold start) is viable given the small dataset size, reducing reads to the data store.
  - Each endpoint can be a separate Lambda function for deployment atomicity, or a single function with path-based routing — decision deferred to implementation.
- Principal alternative (optional):
  - AWS AppSync (GraphQL): would provide flexible querying and built-in caching. Not chosen because the access patterns are simple and well-defined — REST is simpler and faster to develop.

### 70.2 — Format Data Store

- Category:
  - database
- Purpose:
  - Persists the format registry data: supported formats, valid conversion paths, and option schemas. Provides low-latency reads for the Format Registry API.
- Technology choice:
  - Amazon DynamoDB (on-demand capacity mode). Single table design.
- Responsibilities:
  - Store format definitions (format ID, display name, file extensions, MIME types, category)
  - Store conversion path records (source format → target format, validity flag, associated option schema)
  - Store option schemas per conversion path (JSON Schema or equivalent describing available options)
  - Support efficient queries by format ID and by source-target pair
  - Support TTL on records if needed for future use (though current data is long-lived)
- Interfaces:
  - Incoming:
    - Read queries from Format Registry API (70.1)
    - Write operations from Format Registry Admin (70.3) — create/update/delete format and conversion path records
  - Outgoing:
    - Query results to 70.1 and 70.3
- Data / state:
  - Format records: `{ PK: "FORMAT#pdf", name: "PDF", extensions: [".pdf"], mimeTypes: ["application/pdf"], category: "text" }`
  - Conversion path records: `{ PK: "PATH#pdf#docx", sourceFormat: "pdf", targetFormat: "docx", valid: true, optionsSchema: {...} }`
  - A `FORMATS_INDEX` record or GSI for listing all formats efficiently
  - Data volume is small (tens to low hundreds of records) — well within DynamoDB free tier
- Dependencies:
  - Internal:
    - None
  - External:
    - None (infrastructure element)
- Observability / operational considerations:
  - DynamoDB built-in metrics: read/write capacity consumption, throttling, latency
  - CloudWatch alarms on throttling events (should not occur with on-demand mode, but worth monitoring)
  - Point-in-time recovery enabled for data protection
- Constraints / notes:
  - On-demand capacity mode aligns with serverless-first and unpredictable (but very low) traffic patterns.
  - Single-table design keeps it simple: partition key encodes entity type and ID.
  - Data rarely changes — could alternatively be a static JSON file in S3 or baked into the Lambda deployment package. DynamoDB is chosen for independent updatability without redeploying the API function.
- Principal alternative (optional):
  - Static JSON configuration file bundled with the Lambda or stored in S3: simpler and zero-cost, but updating formats would require redeploying the Lambda or invalidating caches. DynamoDB was chosen to allow admin updates without API redeployment, preserving deployment independence.

### 70.3 — Format Registry Admin

- Category:
  - API service
- Purpose:
  - Provides a write-path for managing the format registry data. Used at deploy-time or by administrators to add, update, or remove supported formats and conversion paths.
- Technology choice:
  - AWS Lambda (Node.js runtime) with Lambda Function URL (IAM-authenticated). Alternatively invoked via AWS CLI / SDK during CI/CD pipelines. Deployed via Terraform.
- Responsibilities:
  - Serve `POST /admin/formats` — add a new supported format
  - Serve `PUT /admin/formats/{formatId}` — update an existing format definition
  - Serve `DELETE /admin/formats/{formatId}` — remove a format (with validation that no active conversion paths reference it, or cascade-remove)
  - Serve `POST /admin/conversions` — add a new conversion path with its option schema
  - Serve `PUT /admin/conversions/{sourceFormat}/{targetFormat}` — update a conversion path or its options
  - Serve `DELETE /admin/conversions/{sourceFormat}/{targetFormat}` — remove a conversion path
  - Validate input data (format definitions, option schemas) before writing to the data store
  - Optionally support bulk import from a JSON seed file (useful for initial setup and CI/CD)
- Interfaces:
  - Incoming:
    - HTTP requests from authorized administrators or CI/CD pipelines (IAM-authenticated)
  - Outgoing:
    - Write operations to Format Data Store (70.2)
- Data / state:
  - Stateless. Delegates all persistence to 70.2.
- Dependencies:
  - Internal:
    - 70.2 (Format Data Store)
  - External:
    - None
- Observability / operational considerations:
  - Audit logging: every write operation logged with timestamp, actor (IAM principal), and change details
  - CloudWatch metrics: invocation count, error rate
  - Version tracking: optionally store a version counter or timestamp in the data store to let consumers detect registry changes
- Constraints / notes:
  - Not exposed to anonymous users or consumer services. Access is strictly IAM-authenticated.
  - Expected to be invoked infrequently (only when formats are added/changed). Scales to zero.
  - Could alternatively be implemented as a Terraform-managed seed script that runs `aws dynamodb put-item` commands during deployment. The Lambda approach is chosen for flexibility (allows runtime admin updates without a full Terraform apply).
- Principal alternative (optional):
  - Direct DynamoDB writes via Terraform `aws_dynamodb_table_item` resources or a CI/CD script: simpler for initial setup, but less flexible for runtime updates. Lambda chosen to support both deploy-time seeding and occasional runtime admin changes.

## Open questions

- Should the Format Registry API support a versioning mechanism (e.g., an ETag or version number) so consumers can detect when the registry has changed and invalidate their caches?
- Should conversion option schemas use JSON Schema, or a simpler custom format?
- Should the admin component support a "dry-run" mode for validating changes without applying them?
- For the initial release, what is the exact set of text formats and valid conversion paths to seed?
