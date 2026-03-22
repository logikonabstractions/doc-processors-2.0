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
- All compute is serverless (AWS Lambda) per architectural preferences
- Fast development is the priority — use managed AWS services wherever possible
- The registry data changes infrequently (only when new formats are added)
- Consumers (Element 20 and Element 30) should cache registry responses to minimize cross-service calls
- Write/admin access to registry data is a deploy-time concern, not a runtime API concern
- The registry must be independently deployable from all other elements

## Constraints
- Must be hosted on AWS (architecture constraint)
- Serverless-first: scale-to-zero, pay-per-invocation (architecture preference)
- Infrastructure managed via Terraform (architecture assumption)
- Read-only API for consumer services; write access is admin/deploy-time only
- Must support future extensibility to non-text file types without architectural changes

## Component breakdown

### 70.1 — Format Registry API (Lambda)

- Category:
  - API service
- Purpose:
  - Serverless function that exposes a read-only HTTP API for querying supported formats, validating conversion paths, and retrieving conversion options. This is the single interface through which Element 20 and Element 30 interact with the Format Registry.
- Technology choice:
  - AWS Lambda (Node.js or Python runtime), fronted by API Gateway (a dedicated REST resource or a lightweight HTTP API endpoint). The Lambda reads from the Format Registry Data Store (70.2) and returns JSON responses.
- Responsibilities:
  - Serve `GET /formats` — list all supported source and target formats
  - Serve `GET /formats/{sourceFormat}/targets` — list valid target formats for a given source format
  - Serve `GET /conversions/{sourceFormat}/{targetFormat}/options` — return the option schema for a specific conversion path
  - Serve `GET /conversions/{sourceFormat}/{targetFormat}/validate` — validate whether a conversion path is supported
  - Return structured JSON error responses for unsupported format queries
  - Implement in-memory caching of registry data per Lambda invocation (warm-start optimization)
- Interfaces:
  - Incoming:
    - HTTP requests from Element 20 (Conversion API Service) and Element 30 (Conversion Worker) — routed via API Gateway (Element 10) or direct Lambda invocation
  - Outgoing:
    - Read queries to Format Registry Data Store (70.2)
    - JSON responses to callers
- Data / state:
  - Stateless. Reads all data from the Format Registry Data Store (70.2). May hold an in-memory cache of registry data for the duration of a warm Lambda invocation.
- Dependencies:
  - Internal:
    - 70.2 (Format Registry Data Store)
  - External:
    - Element 10 (API Gateway) for routing, if exposed through the shared gateway
- Observability / operational considerations:
  - Structured logging of each query (requested format pair, result, latency)
  - Logging of unsupported format requests to inform future format development priorities
  - Metrics: request count, latency, cache hit/miss ratio (if in-memory cache is used)
  - CloudWatch alarms on error rates and latency
- Constraints / notes:
  - The API is strictly read-only at runtime. No mutation endpoints are exposed to consumer services.
  - Responses should include `Cache-Control` headers to allow consumer-side HTTP caching (short TTL, e.g., 5 minutes).
  - The Lambda should be lightweight and fast-starting — minimal dependencies, small deployment package.
  - Can be invoked directly via Lambda function URL or AWS SDK by internal services to avoid an extra API Gateway hop for internal calls.
- Principal alternative (optional):
  - Embedding format data directly in a shared configuration file (e.g., S3 JSON file fetched by each consumer). Rejected because it sacrifices deployment independence — updating the registry would require redeploying or cache-busting consumers, and there would be no centralized validation logic.

### 70.2 — Format Registry Data Store (DynamoDB)

- Category:
  - storage
- Purpose:
  - Persists the format registry data: supported formats, valid conversion paths, and per-path option schemas. Serves as the single source of truth for all format-related configuration.
- Technology choice:
  - Amazon DynamoDB (on-demand capacity mode). Chosen for: managed serverless data store (no connection pooling needed), HTTP-based API compatible with Lambda, native JSON document support for option schemas, and TTL support if needed. On-demand mode aligns with low and bursty read patterns.
- Responsibilities:
  - Store format definitions (format ID, name, MIME types, file extensions, category)
  - Store conversion path records (source format → target format, with enabled/disabled flag)
  - Store option schemas per conversion path (JSON schema describing available conversion options)
  - Serve read queries from the Format Registry API (70.1)
  - Accept write operations from the Format Registry Seed/Admin component (70.3) during deployment or admin operations
- Interfaces:
  - Incoming:
    - Read queries from 70.1 (Format Registry API)
    - Write operations from 70.3 (Format Registry Seed/Admin)
  - Outgoing:
    - Query results to 70.1
- Data / state:
  - **Formats table**: partition key = `formatId` (e.g., `pdf`, `docx`, `md`). Attributes: name, MIME types, file extensions, category (text, image, geo, video — for future extensibility), description.
  - **ConversionPaths table**: partition key = `sourceFormat`, sort key = `targetFormat`. Attributes: enabled (boolean), option schema (JSON), notes.
  - Data is long-lived (not ephemeral), but small in volume. Changes are infrequent.
- Dependencies:
  - Internal:
    - None
  - External:
    - None
- Observability / operational considerations:
  - DynamoDB built-in metrics: read/write capacity consumption, throttling events, latency
  - CloudWatch alarms on throttling (should be near-zero given low traffic)
  - Point-in-time recovery enabled for safety, even though data can be re-seeded
- Constraints / notes:
  - On-demand capacity mode — no provisioned throughput to manage, aligns with serverless-first and bursty access.
  - Table design should be simple and denormalized given the small dataset and simple access patterns.
  - Encryption at rest enabled by default (AWS managed key is sufficient).
  - Could use a single table design (single-table DynamoDB pattern) to reduce operational surface, with `PK` / `SK` key structure distinguishing formats from conversion paths.
- Principal alternative (optional):
  - Amazon S3 (JSON files). Simpler and cheaper for static data, but lacks query capabilities and atomic updates. Would require full-file reads and client-side filtering. Rejected in favor of DynamoDB for its query flexibility and consistency guarantees, even though the dataset is small.

### 70.3 — Format Registry Seed / Admin (Lambda + Terraform)

- Category:
  - adapter
- Purpose:
  - Manages the lifecycle of format registry data: seeds initial data on deployment, and provides an admin mechanism for adding or updating formats and conversion paths. This is the write-side counterpart to the read-only API (70.1).
- Technology choice:
  - A combination of: (1) Terraform-managed seed data for initial deployment (DynamoDB table items defined in Terraform or loaded via a Lambda-backed custom resource), and (2) an optional admin Lambda function for runtime updates (invoked via CLI, CI/CD pipeline, or a restricted admin API endpoint — not exposed publicly).
- Responsibilities:
  - Define and maintain the initial set of supported formats and conversion paths as structured data (JSON/YAML seed files in the repository)
  - Seed the DynamoDB tables on first deployment or when seed data changes
  - Provide an admin Lambda for adding, updating, or disabling formats and conversion paths at runtime without a full redeployment
  - Validate seed data and admin inputs against a schema before writing to DynamoDB
  - Support idempotent seeding (re-running seed does not create duplicates or corrupt data)
- Interfaces:
  - Incoming:
    - Terraform apply (deployment-time seeding)
    - CLI / CI/CD invocation of admin Lambda (runtime updates)
  - Outgoing:
    - Write operations to 70.2 (Format Registry Data Store)
- Data / state:
  - Owns the seed data files (JSON/YAML) checked into the repository. These are the declarative source of truth for the initial registry content.
  - The admin Lambda is stateless; it writes directly to DynamoDB.
- Dependencies:
  - Internal:
    - 70.2 (Format Registry Data Store)
  - External:
    - Terraform (IaC toolchain)
- Observability / operational considerations:
  - Logging of all seed and admin write operations (what changed, who triggered it, timestamp)
  - Terraform plan output shows registry data changes before apply
  - Version tracking: seed data files are version-controlled in git, providing a full audit trail
- Constraints / notes:
  - The seed data files in the repo are the canonical definition of the initial format registry. Any format added via the admin Lambda should eventually be codified in the seed files to prevent drift.
  - The admin Lambda should NOT be exposed via the public API Gateway. Access should be restricted to IAM roles used by CI/CD or operators.
  - Idempotent seeding is critical — `terraform apply` should be safe to run repeatedly.
  - For the initial iteration, Terraform-based seeding may be sufficient without the admin Lambda. The admin Lambda is a natural extension when runtime updates become needed.
- Principal alternative (optional):
  - Direct DynamoDB console or CLI updates. Rejected because it leaves no audit trail, is error-prone, and does not enforce schema validation.

### 70.4 — Format Registry Terraform Module

- Category:
  - other (infrastructure-as-code)
- Purpose:
  - Defines all AWS infrastructure required by the Format Registry as a self-contained Terraform module: DynamoDB tables, Lambda functions, IAM roles, API Gateway integration, and CloudWatch alarms. Ensures the Format Registry is independently deployable.
- Technology choice:
  - Terraform (HCL). A dedicated Terraform module (`modules/format-registry/`) encapsulating all Element 70 resources. Uses AWS provider.
- Responsibilities:
  - Provision DynamoDB table(s) for the Format Registry Data Store (70.2)
  - Provision Lambda function(s) for the Format Registry API (70.1) and Seed/Admin (70.3)
  - Configure IAM roles and policies: read-only DynamoDB access for 70.1, read-write for 70.3
  - Configure API Gateway route integration for the Format Registry API (or Lambda function URL)
  - Set up CloudWatch log groups, metric alarms, and dashboards
  - Expose outputs (API endpoint URL, Lambda ARN, DynamoDB table name) for consumption by other Terraform modules (Element 20 and 30 need the API URL)
  - Support independent `terraform apply` for the Format Registry without affecting other elements
- Interfaces:
  - Incoming:
    - Terraform variables: environment name, API Gateway ID (if shared), tags, alarm thresholds
  - Outgoing:
    - Terraform outputs: Format Registry API URL, Lambda function ARNs, DynamoDB table ARN
- Data / state:
  - Terraform state for Element 70 resources (stored in a shared or per-module S3 backend)
- Dependencies:
  - Internal:
    - None (this module creates the infrastructure for 70.1, 70.2, 70.3)
  - External:
    - Element 10 (API Gateway) — if the Format Registry API is exposed through the shared gateway, this module needs the gateway ID as input. Otherwise, a Lambda function URL provides full independence.
- Observability / operational considerations:
  - Terraform plan/apply logs for audit
  - Module versioning via git tags or Terraform registry
  - Drift detection via `terraform plan` in CI/CD
- Constraints / notes:
  - The module must be fully self-contained so that `terraform apply` for Element 70 does not require or affect other elements' state.
  - Follows the project-wide Terraform conventions (backend config, provider versioning, tagging strategy).
  - Least-privilege IAM: 70.1 gets `dynamodb:GetItem`, `dynamodb:Query`, `dynamodb:Scan` only. 70.3 additionally gets `dynamodb:PutItem`, `dynamodb:UpdateItem`, `dynamodb:DeleteItem`.

## Open questions

- Should the Format Registry API (70.1) be exposed through the shared API Gateway (Element 10) or use a Lambda function URL for fully independent deployment? Using the shared gateway simplifies consumer routing but creates a deployment dependency on Element 10.
- Should a single-table DynamoDB design be used (formats and conversion paths in one table) or separate tables? Single-table reduces operational surface but adds key design complexity.
- What specific text formats should be included in the initial seed data? The architecture lists PDF, .docx, .md, .txt, .html, .rtf, .csv, .xlsx — should all pairwise conversions be supported, or only a curated subset?
- Should the option schemas per conversion path follow JSON Schema format, or a simpler custom structure?
- Is the admin Lambda (70.3) needed for the initial release, or is Terraform-based seeding sufficient?
