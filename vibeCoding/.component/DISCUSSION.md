# DISCUSSION

## How to use this file

- This is the **component discussions**.
- Use it to track questions about specific components that require clarification, comparison, investigation, or explicit decision.
- Keep one item per question or decision.
- When a question is resolved, keep the item here for traceability, mark it `RESOLVED`, and summarize the outcome in `HISTORY.md`.
- Any Comp-X.Y with a status `RESOVLED` should not be updated later. If the issue needs to be revisited, favor opening a new Comp-X.Y and reference the previous discussion.

## Question resolution

When a question is resolved:
  - Add a relevant entry to `HISTORY.md`
  - Update this file's status to `RESOLVED` along with any field (notes, etc.)

## Active component issues

### Comp-70.1 — Can we eliminate the dedicated API and Admin components?

- Type: DECISION
- Status: DECISION_REQUIRED
- Related component(s):
  - 70.1 (Format Registry API), 70.2 (Format Data Store), 70.3 (Format Registry Admin)
- Description:
  - The current breakdown introduces two Lambda-based APIs (70.1 for reads, 70.3 for admin writes) on top of the DynamoDB store (70.2). The question is whether both can be eliminated:
    1. **Admin API (70.3)**: format updates are a dev-team activity — devs already manage infrastructure via Terraform and could manage DynamoDB records directly (console, CLI, IaC seed scripts) rather than through a dedicated admin API.
    2. **Read API (70.1)**: consumer services (Elements 20 and 30) are internal micro-services running within the same VPC. They could query DynamoDB directly via the AWS SDK instead of going through an intermediary HTTP API.
- Known options / hypotheses:
  - **Option A — Keep current design (3 components: API + Store + Admin)**
    - Pros: clean separation of concerns, API contract insulates consumers from storage implementation, admin API enables runtime updates without console/CLI access.
    - Cons: two additional Lambda functions to deploy, maintain, and monitor. Over-engineered for a small, rarely-changing dataset accessed only by internal services.
  - **Option B — DynamoDB-only with shared access library (1 component: Store + shared SDK module)**
    - Eliminate both 70.1 and 70.3. Consumers (Elements 20, 30) read DynamoDB directly using a shared internal library/module that encapsulates query logic. Devs manage data via Terraform seed resources, AWS CLI, or console.
    - Pros: drastically simpler — zero additional Lambda functions, no extra API surface, no additional deployments. Aligns with fast-development priority. DynamoDB IAM policies enforce read-only access for consumer roles.
    - Cons: couples consumers to the DynamoDB table schema (if the schema changes, the shared library must be updated and consumers redeployed). No HTTP abstraction layer. Admin changes require Terraform apply or manual CLI/console work.
  - **Option C — Hybrid: eliminate Admin API only, keep read API (2 components: API + Store)**
    - Drop 70.3 (devs manage data directly). Keep 70.1 as a thin read API so consumers are decoupled from the storage layer.
    - Pros: still provides an abstraction for consumers while removing the unnecessary admin API.
    - Cons: still one extra Lambda to deploy and maintain for what amounts to a simple DynamoDB read.
- Required input / evidence:
  - Human decision on acceptable coupling level between consumer services and the Format Registry's storage implementation.
  - Confirmation that the dev team is comfortable managing DynamoDB records directly (via Terraform/CLI/console) for format updates.
- Resolution criteria:
  - A clear decision on the target component count (1, 2, or 3) and the access pattern for both consumers and admins.
- Notes:
  - The architectural description for Element 70 specifies "deployed as an independent serverless function backed by a managed key-value store" and emphasizes independent deployability. Option B would shift "independent deployability" from a separate service to a shared library — the DynamoDB table itself is still independently managed, but the query logic lives inside consumer functions.
  - Given the dataset is small and rarely changes, even Option B's coupling risk is low in practice.

#### Response — Comp-70.1

- Respondent: agent
- Answer:
  - I lean toward **Option B** (DynamoDB-only with shared access module), primarily because:
    1. **Fast development priority**: this is the #1 architectural preference. Two fewer Lambdas means fewer deployments, fewer IAM roles, fewer CloudWatch dashboards, and less Terraform to write and maintain.
    2. **The data is trivially small and stable**: the format registry is a handful of records that change only when the dev team ships new format support. This doesn't warrant a dedicated service.
    3. **All consumers are internal**: there is no external contract to protect. Elements 20 and 30 are our own services in our own VPC. A shared module (e.g., a small `format-registry-client` package) gives us a clean interface without the overhead of an HTTP hop.
    4. **Admin path is clear**: devs manage infra via Terraform. Adding a `aws_dynamodb_table_item` resource (or a seed script) for format data fits naturally into the existing IaC workflow. A dedicated admin Lambda adds ceremony without value.
  - The main trade-off is schema coupling. If the DynamoDB table schema changes, the shared library and its consumers need updating. But given the low change frequency and the fact that a schema change would likely accompany a new conversion capability (requiring Element 30 changes anyway), this coupling is acceptable.
- Additional Details:
  - If Option B is chosen, component 70 would simplify to:
    - **70.1 — Format Data Store** (DynamoDB table, Terraform-managed, seeded via IaC)
    - **70.2 — Format Registry Client Library** (shared internal module consumed by Elements 20 and 30, encapsulating DynamoDB queries)
  - The client library approach preserves a clean interface boundary without the operational cost of a separate service.

## Response template

Use the following template to record a response to any Compo-x.y item.
Append the response block directly under the item it answers.

```
#### Response — Compo-<N.N>

- Respondent: <human | agent>
- Answer:
  - <direct answer, decision, or information provided>
- Additional Details:
  - <Context, caveats etc.>
```
