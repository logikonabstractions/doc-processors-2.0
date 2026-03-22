# COMPONENTS DESCRIPTIONS

The response must provide a component-level design for **one parent architectural element**.

# PROBLEM STATEMENT

## Parent architectural element

- Parent ID:
  - 70
- Parent name:
  - Format Registry
- Parent purpose:
  - Centralize supported formats, valid conversion paths, and per-path option metadata behind a read-only API that can be consumed independently by the Conversion API Service and Conversion Worker.

## Assumptions
- The initial registry is small, curated, and updated infrequently through deploy-time or admin-controlled data changes rather than end-user writes.
- Consumer services cache registry responses for a short TTL, so the registry favors correctness and stable contracts over high write throughput.
- The registry must remain independently deployable from Element 20 and Element 30, but it can still rely on managed cloud services such as a document store and serverless runtime.
- Conversion option definitions are metadata/schema-oriented; execution-specific converter code continues to live outside Element 70.

## Constraints
- The public surface for runtime consumers is read-only.
- The implementation must stay lightweight and serverless-friendly to align with the architecture's fast-development priority.
- Registry data changes rarely, so the design should optimize for simple operations, strong validation, and cache-friendly responses.
- Adding a new format should not require changes in Element 20; only registry data and, when necessary, worker strategy support should change.

## Component breakdown

### 70.1 — Registry Query API

- Category:
  - API service
- Purpose:
  - Expose the read-only HTTP contract used by internal consumers to validate conversion requests, list supported formats, and fetch option metadata.
- Technology choice:
  - Serverless HTTP function implemented in TypeScript on Node.js 22, deployed behind an internal HTTPS endpoint/API Gateway route.
- Responsibilities:
  - Serve endpoints for supported formats, valid source/target pairs, and option metadata
  - Normalize and validate incoming query parameters and path arguments
  - Translate persisted registry documents into stable JSON response contracts
  - Emit cache headers/ETag metadata suitable for short-TTL client caching
  - Return explicit not-supported / invalid-path responses for consumer validation flows
- Interfaces:
  - Incoming:
    - HTTP GET requests from Element 20 for format listing and path validation
    - HTTP GET requests from Element 30 for strategy-selection metadata and option definitions
  - Outgoing:
    - Read requests to 70.2 for format, path, and option documents
    - Structured logs/metrics to 70.4
- Data / state:
  - Stateless request handling only
  - In-memory per-instance short-lived cache for hot registry lookups (best-effort)
  - Response DTO mappings and API schema/version metadata
- Dependencies:
  - Internal:
    - 70.2
    - 70.4
  - External:
    - 20
    - 30
- Observability / operational considerations:
  - Track request count, latency, cache hit/miss, and unsupported format/path queries
  - Include endpoint name, source format, target format, and response status in structured logs
  - Publish contract/version identifiers in logs to correlate consumer behavior with registry revisions
- Constraints / notes:
  - The API must remain read-only for runtime consumers; any write/admin flow is separate from this surface
  - Prefer explicit versioned routes or version headers so future schema expansion does not break consumers
  - Responses should be deterministic and compact to maximize downstream cache efficiency
- Principal alternative (optional):
  - Embedding registry metadata directly into Element 20 and Element 30 configs was rejected because it breaks independent deployability and increases duplication/drift risk.

### 70.2 — Registry Metadata Store

- Category:
  - database
- Purpose:
  - Persist canonical definitions for formats, conversion-path rules, and option schemas used by the query API.
- Technology choice:
  - Managed document/key-value store with serverless-friendly access patterns (for example DynamoDB or Firestore in document mode).
- Responsibilities:
  - Store canonical format records with identifiers, labels, MIME types, and capability metadata
  - Store valid source-to-target conversion path records
  - Store conversion option schema documents per path and schema version
  - Support efficient lookups by source format, target format, and format identifier
  - Preserve registry revision metadata for auditing and cache invalidation
- Interfaces:
  - Incoming:
    - Read requests from 70.1
    - Controlled write/update operations from 70.3 during registry releases
  - Outgoing:
    - Registry documents returned to 70.1 and status/write acknowledgements to 70.3
- Data / state:
  - Format definitions
  - Conversion path matrix entries
  - Option schema/version records
  - Registry revision identifiers, timestamps, and change summaries
- Dependencies:
  - Internal:
    - 70.3
  - External:
    - None
- Observability / operational considerations:
  - Monitor read latency, throttling, failed conditional writes, and item-count/storage growth
  - Record registry revision metadata so changes can be correlated with downstream incidents
  - Alert if the store becomes unavailable because it impacts request validation across the platform
- Constraints / notes:
  - The schema should support infrequent but safe updates, favoring clarity over denormalization complexity
  - Read access must be granted to the runtime query API only; write access is limited to controlled update workflows
  - Data design should keep the hot read path to one or two queries for common validation requests
- Principal alternative (optional):
  - Static JSON bundled with the function was rejected because it would require a code deploy for every registry change and weaken operational auditing.

### 70.3 — Registry Publication Workflow

- Category:
  - other
- Purpose:
  - Provide the controlled mechanism that validates and publishes registry data changes into the metadata store without exposing runtime write access.
- Technology choice:
  - CI/deployment workflow plus a TypeScript validation/publish CLI executed during release automation.
- Responsibilities:
  - Validate proposed format/path/option metadata against a canonical schema before publication
  - Enforce referential integrity between formats, conversion paths, and option definitions
  - Publish registry updates to 70.2 using idempotent upsert semantics
  - Stamp each successful publication with a registry revision/version
  - Generate change summaries for operators and downstream release notes
- Interfaces:
  - Incoming:
    - Registry data manifests from maintainers/repository source control
    - Deployment pipeline triggers for approved registry updates
  - Outgoing:
    - Validated writes to 70.2
    - Publication logs, validation failures, and revision metadata to 70.4
- Data / state:
  - Source-controlled registry manifests
  - Validation rules/schema definitions
  - Publication results and revision records
- Dependencies:
  - Internal:
    - 70.2
    - 70.4
  - External:
    - CI/CD platform or deployment runner
- Observability / operational considerations:
  - Track publication success/failure, validation error categories, and latest deployed registry revision
  - Keep an audit trail of who/what pipeline published each revision
  - Make dry-run validation available so changes can be reviewed before deployment
- Constraints / notes:
  - This workflow is an operator/admin concern, not part of the runtime consumer API
  - Publication must fail atomically when manifests are inconsistent, preventing partially updated registry state
  - The workflow should be runnable locally and in CI to reduce production-only surprises
- Principal alternative (optional):
  - A dedicated admin CRUD API was rejected for the initial design because it adds runtime attack surface and more operational complexity than deploy-time publication requires.

### 70.4 — Registry Observability and Cache Control Module

- Category:
  - observability
- Purpose:
  - Centralize the cross-cutting concerns that make registry reads observable, cache-aware, and operationally safe.
- Technology choice:
  - Shared library/module for structured logging, metrics emission, response cache metadata generation, and revision-aware instrumentation.
- Responsibilities:
  - Standardize structured log fields across query and publication paths
  - Emit metrics for unsupported formats, invalid conversion requests, store latency, and publication outcomes
  - Derive cache validators (ETag/revision/last-modified) from registry revision metadata
  - Surface revision information for dashboards and incident triage
  - Provide reusable instrumentation hooks so 70.1 and 70.3 stay lightweight
- Interfaces:
  - Incoming:
    - Event/log/metric calls from 70.1 and 70.3
    - Revision metadata from 70.2-backed reads/publications
  - Outgoing:
    - Telemetry to the platform logging/metrics backend
    - Cache-control and revision headers back to 70.1 responses
- Data / state:
  - No primary business data ownership
  - Reusable telemetry field definitions and revision-to-header mapping rules
- Dependencies:
  - Internal:
    - None
  - External:
    - Platform logging/metrics service
- Observability / operational considerations:
  - Ensure unsupported format requests are easy to aggregate for roadmap decisions
  - Separate consumer misuse signals from internal failure signals in dashboards/alerts
  - Expose the current active registry revision in a low-cost way for debugging cache issues
- Constraints / notes:
  - Must remain implementation-agnostic enough to support both the query API and publication workflow
  - Should avoid introducing heavy runtime dependencies that undermine the lightweight serverless design

## Open questions

- Should format/path metadata include a direct pointer to the worker strategy identifier used by Element 30, or should Element 30 own that mapping separately?
- Does the team want one global registry revision number or independent versioning for formats, paths, and option schemas?
- Is there a need for a dedicated internal endpoint that returns the full registry snapshot for warm-cache/bootstrap scenarios, or are path-specific queries sufficient for the first release?
