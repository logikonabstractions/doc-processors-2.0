# COMPONENTS DESCRIPTIONS

The response must provide a component-level design for **one parent architectural element**.

# PROBLEM STATEMENT

## Parent architectural element

- Parent ID:
  - 10
- Parent name:
  - API Gateway
- Parent purpose:
  - Single entry point for all client requests. Handles routing, request validation, rate limiting, and CORS.

## Assumptions

- AWS is the sole cloud provider.
- Fast development is the top priority — managed services are strongly preferred over custom code.
- All users are anonymous (no authentication/authorization flows needed at the gateway level).
- The gateway does NOT proxy file content — file upload/download uses pre-signed URLs directly to S3 (Element 50).
- Traffic is bursty and unpredictable; the gateway must scale instantly and to zero.
- The gateway fronts a small, well-defined set of API routes (pre-signed URL requests, job submission, status checks, download URL requests, format listing).

## Constraints

- Must be a fully managed AWS service (inherited from architecture: fast-development priority).
- Serverless-first: no reserved capacity, pay-per-request pricing.
- TLS termination happens at this layer.
- Maximum request payload size enforcement is required (but large files bypass the gateway via pre-signed URLs).
- Rate limiting must be per-client-IP since users are anonymous.
- Must integrate natively with AWS Lambda (Element 20 is deployed as Lambda functions).

## Component breakdown

### 10.1 — API Gateway HTTP API

- Category:
  - API service
- Purpose:
  - Managed HTTP API that receives all client requests, enforces request-level policies, and routes them to the appropriate Lambda function in the Conversion API Service (Element 20).
- Technology choice:
  - AWS API Gateway (HTTP API type). HTTP API is chosen over REST API for lower cost, lower latency, and simpler configuration. It natively supports Lambda proxy integrations, CORS, and JWT authorizers (available if auth is added later).
- Responsibilities:
  - Define and expose API routes:
    - `POST /conversions` — request a new conversion (returns pre-signed upload URL + job ID)
    - `GET /conversions/{jobId}` — check job status
    - `GET /conversions/{jobId}/download` — get pre-signed download URL for completed conversion
    - `GET /formats` — list supported source/target formats and options
  - Route each endpoint to the corresponding Lambda function in Element 20
  - Enforce maximum request body size (API Gateway default: 10 MB; sufficient since files bypass the gateway)
  - Provide TLS termination for all inbound traffic
  - Return structured error responses for malformed or rejected requests
- Interfaces:
  - Incoming:
    - HTTPS requests from anonymous web clients (browsers, CLI tools, etc.)
  - Outgoing:
    - Lambda proxy integration invocations to Element 20 (Conversion API Service)
- Data / state:
  - Stateless. Route definitions and stage configuration are deployment-time artifacts stored in API Gateway's managed control plane.
- Dependencies:
  - Internal:
    - 10.2 (CORS configuration is applied at this layer)
    - 10.3 (throttling settings are configured on this API)
    - 10.4 (custom domain and TLS certificate are attached here)
    - 10.5 (Terraform module provisions this resource)
  - External:
    - Element 20 (Conversion API Service) — Lambda functions that handle routed requests
- Observability / operational considerations:
  - API Gateway automatically emits access logs and execution logs to CloudWatch
  - Metrics: request count, latency (p50/p90/p99), 4xx rate, 5xx rate — all available out of the box
  - CloudWatch alarms on sustained 5xx error rates
  - Access logging enabled with request ID, method, path, status, latency, client IP
- Constraints / notes:
  - HTTP API type has a hard limit of 10 MB payload size (sufficient since file content is not proxied)
  - HTTP API does not support AWS WAF natively (REST API does) — if WAF becomes necessary, consider CloudFront in front or migration to REST API type
  - Lambda proxy integration means the Lambda function receives the full request context and controls the response format
- Principal alternative (optional):
  - AWS API Gateway REST API. Offers built-in WAF integration, request/response transformation, and API keys, but at higher cost and latency. Not chosen because the current scope does not require WAF or request transformation, and fast development favors the simpler HTTP API model.

### 10.2 — CORS Configuration

- Category:
  - API service (configuration layer)
- Purpose:
  - Configures Cross-Origin Resource Sharing headers so that browser-based clients from allowed origins can interact with the API.
- Technology choice:
  - AWS API Gateway HTTP API built-in CORS support. Configured declaratively via the API Gateway CORS settings (no custom Lambda code needed).
- Responsibilities:
  - Define allowed origins (configurable per environment: wildcard for dev, specific domain(s) for production)
  - Define allowed HTTP methods (`GET`, `POST`, `OPTIONS`)
  - Define allowed headers (`Content-Type`, `Accept`, etc.)
  - Define exposed headers (e.g., `X-Request-Id` for client-side correlation)
  - Set `Access-Control-Max-Age` for preflight caching (e.g., 3600 seconds)
  - Handle `OPTIONS` preflight requests automatically (API Gateway HTTP API does this natively)
- Interfaces:
  - Incoming:
    - Browser preflight `OPTIONS` requests
    - CORS headers are appended to all responses passing through the API Gateway
  - Outgoing:
    - CORS response headers (`Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`, etc.)
- Data / state:
  - CORS configuration is a deployment-time artifact. No runtime state.
- Dependencies:
  - Internal:
    - 10.1 (CORS is configured on the HTTP API resource)
    - 10.5 (Terraform module manages the CORS configuration)
  - External:
    - None
- Observability / operational considerations:
  - Misconfigured CORS is a common source of client-side errors — include CORS-related headers in access logs for debugging
  - No dedicated metrics; CORS failures surface as 403 responses in API Gateway metrics
- Constraints / notes:
  - Production origins must be explicitly listed (avoid wildcard `*` in production)
  - CORS settings must be kept in sync with any custom domain configuration (10.4)
- Principal alternative (optional):
  - Handling CORS in the Lambda function code. Not chosen because API Gateway HTTP API handles CORS natively and declaratively, reducing code and eliminating a class of bugs.

### 10.3 — Rate Limiting & Throttling

- Category:
  - API service (policy layer)
- Purpose:
  - Protects the system from abuse and ensures fair usage across anonymous clients by enforcing request rate limits.
- Technology choice:
  - AWS API Gateway built-in throttling (account-level and route-level). Supplemented by a usage plan if fine-grained per-IP throttling is needed (requires REST API or a lightweight Lambda authorizer on HTTP API for IP-based tracking).
- Responsibilities:
  - Enforce a default request rate limit (requests per second) at the API stage level
  - Enforce a burst limit for short traffic spikes
  - Optionally configure per-route throttling overrides (e.g., lower limit on `POST /conversions` to protect compute-intensive downstream paths)
  - Return HTTP `429 Too Many Requests` with a `Retry-After` header when limits are exceeded
- Interfaces:
  - Incoming:
    - All requests passing through API Gateway are subject to throttling
  - Outgoing:
    - `429` responses when throttle limits are breached
    - Throttled request metrics emitted to CloudWatch
- Data / state:
  - Throttle counters are managed internally by API Gateway. No application-owned state.
- Dependencies:
  - Internal:
    - 10.1 (throttling is configured on the HTTP API stage/routes)
    - 10.5 (Terraform module manages throttling settings)
  - External:
    - None
- Observability / operational considerations:
  - CloudWatch metric: `Count` of throttled requests (`429` responses)
  - CloudWatch alarm on sustained throttling (may indicate a legitimate traffic increase requiring limit adjustment)
  - Access logs include whether a request was throttled
- Constraints / notes:
  - API Gateway HTTP API supports account-level and route-level throttling, but does NOT support per-client API keys or usage plans natively (those are REST API features)
  - If per-IP rate limiting becomes a hard requirement, two options: (a) add a lightweight Lambda authorizer that tracks IP-based counters in a fast store (e.g., ElastiCache or DynamoDB), or (b) place CloudFront with AWS WAF rate-based rules in front of the API Gateway
  - For the initial release, account-level and route-level throttling is sufficient — per-IP limiting can be added incrementally
- Principal alternative (optional):
  - AWS WAF rate-based rules (via CloudFront or REST API). Provides true per-IP rate limiting out of the box, but adds cost and complexity. Deferred to a future iteration unless abuse patterns demand it earlier.

### 10.4 — Custom Domain & TLS

- Category:
  - infrastructure / networking
- Purpose:
  - Provides a stable, branded API endpoint (e.g., `api.example.com`) with TLS termination, decoupling clients from the auto-generated API Gateway URL.
- Technology choice:
  - AWS API Gateway custom domain name with AWS Certificate Manager (ACM) for TLS certificates. Optionally fronted by a Route 53 hosted zone for DNS management.
- Responsibilities:
  - Provision and auto-renew a TLS certificate via ACM (DNS-validated)
  - Map the custom domain to the API Gateway HTTP API stage
  - Configure DNS records (CNAME or A-alias) pointing to the API Gateway domain
  - Ensure HTTPS-only access (no HTTP fallback)
- Interfaces:
  - Incoming:
    - HTTPS requests from clients using the custom domain
  - Outgoing:
    - Requests forwarded to API Gateway HTTP API (10.1) under the custom domain mapping
- Data / state:
  - TLS certificate managed by ACM (auto-renewed). DNS records in Route 53 (or external DNS provider).
- Dependencies:
  - Internal:
    - 10.1 (custom domain is mapped to the HTTP API)
    - 10.5 (Terraform module provisions ACM certificate, custom domain name, and DNS records)
  - External:
    - A registered domain name (prerequisite; outside system scope)
- Observability / operational considerations:
  - ACM certificate expiry monitoring (ACM auto-renews, but monitor for renewal failures)
  - DNS propagation verification after changes
- Constraints / notes:
  - ACM certificates for API Gateway must be provisioned in the same region as the API Gateway (unlike CloudFront which requires us-east-1)
  - If CloudFront is added later (e.g., for WAF or caching), the custom domain moves to the CloudFront distribution instead
- Principal alternative (optional):
  - CloudFront distribution in front of API Gateway, with the custom domain on CloudFront. Adds caching and WAF capabilities but increases complexity. Not chosen for the initial release since the API responses are dynamic and per-request, and WAF is not yet required.

### 10.5 — Terraform IaC Module

- Category:
  - infrastructure / IaC
- Purpose:
  - Defines all Element 10 cloud resources as code, enabling reproducible, version-controlled, and reviewable deployments.
- Technology choice:
  - Terraform (HCL) with the AWS provider. Resources managed: `aws_apigatewayv2_api`, `aws_apigatewayv2_stage`, `aws_apigatewayv2_route`, `aws_apigatewayv2_integration`, `aws_acm_certificate`, `aws_apigatewayv2_domain_name`, `aws_route53_record`, CloudWatch log group, and associated IAM roles/policies.
- Responsibilities:
  - Define the API Gateway HTTP API and its stage
  - Define all routes and their Lambda integrations (referencing Lambda ARNs from Element 20's Terraform output)
  - Configure CORS settings declaratively
  - Configure throttling at the stage and route level
  - Provision ACM certificate and custom domain name mapping
  - Create DNS records in Route 53
  - Create CloudWatch log group for access logs and configure log format
  - Create IAM role allowing API Gateway to invoke Element 20 Lambda functions and write to CloudWatch Logs
  - Expose outputs (API endpoint URL, API ID, stage name) for consumption by other Terraform modules
- Interfaces:
  - Incoming:
    - Terraform variables: Lambda function ARNs (from Element 20 module), domain name, Route 53 hosted zone ID, environment name, throttle limits
  - Outgoing:
    - Terraform outputs: API Gateway endpoint URL, API Gateway ID, stage name, custom domain name
- Data / state:
  - Terraform state file (stored in a shared S3 backend with state locking via DynamoDB, per project convention)
- Dependencies:
  - Internal:
    - 10.1, 10.2, 10.3, 10.4 (this module provisions all of them)
  - External:
    - Element 20's Terraform module (provides Lambda ARNs as input)
    - Shared Terraform backend (S3 + DynamoDB for state)
- Observability / operational considerations:
  - Terraform plan output reviewed in CI/CD before apply
  - State drift detection via periodic `terraform plan` in CI
- Constraints / notes:
  - Module should be self-contained: all Element 10 resources in a single module directory, with clear input/output interface
  - Must support multiple environments (dev, staging, production) via variables, not code duplication
  - Lambda integration permissions: requires `aws_lambda_permission` allowing API Gateway to invoke each Lambda (may be defined in Element 20's module, coordinated via Terraform outputs)
- Principal alternative (optional):
  - AWS SAM or AWS CDK. Both provide higher-level abstractions for API Gateway + Lambda patterns. Not chosen because the project has standardized on Terraform for all IaC, and mixing IaC tools adds operational complexity.

## Open questions

- Should CloudFront be placed in front of API Gateway from the start (for WAF and edge caching), or added later only if needed?
- What specific rate limits (requests/second, burst) should be configured for the initial release?
- Is a custom domain required for the initial release, or can the auto-generated API Gateway URL suffice during early development?
- Should API versioning be built into the URL path from the start (e.g., `/v1/conversions`) to support future breaking changes?
