# HISTORY

## Rules

- This file is non-authoritative.
- Use it for architecture drafts, resolved issues, major review outcomes, and durable architectural decisions.
- Prefer pointers to files, commits, or review notes instead of copying large blocks.

## Entry Templates

Use the most relevant template below when adding an entry. All entries must conform to one such template.


## Resolved issues
- Arch-0.1: File upload pattern — direct-to-storage vs API-proxied
  - 2026-03-22
  - Resolution: Option A — Direct-to-storage via pre-signed URLs. Serverless compute is too constrained to proxy file uploads. Conversion API Service issues pre-signed URLs; clients upload/download directly to/from Temporary File Storage.
  - Notes: See DISCUSSION.md Arch-0.1 response.

- Arch-0.2: Worker deployment model — container-based vs serverless
  - 2026-03-22
  - Resolution: Option B — Serverless functions triggered by queue events. Conversion durations are seconds-to-minutes (within serverless limits). Unpredictable traffic and no reserved capacity make scale-to-zero essential. Container-based workers remain a future option via the same queue.
  - Notes: See DISCUSSION.md Arch-0.2 response.

- Arch-0.3: Format Registry — embedded configuration vs standalone service
  - 2026-03-22
  - Resolution: Option B — Standalone service. A lightweight serverless function backed by a managed key-value store. Achieves full deployment independence: format updates don't require redeploying API or Worker services.
  - Notes: See DISCUSSION.md Arch-0.3 response.

## Architectural decisions
- 2026-03-22: Initial architecture draft (Arch.0.1 revision) completed with 7 elements
  - Architectural element IDs (if relevant): [10, 20, 30, 40, 50, 60, 70]
  - Related Discussions if any: Arch-0.1, Arch-0.2, Arch-0.3
  - Rationale: Established the core architectural breakdown for the file conversion system based on the problem statement. Chose an async, queue-based processing model to decouple API from conversion workloads.
  - Impact: Defines the full system structure.

- 2026-03-22: Adopted serverless-first, maximally-decoupled architectural preferences (Arch.0.2 revision)
  - Architectural element IDs (if relevant): [10, 20, 30, 40, 50, 60, 70]
  - Related Discussions if any: Arch-0.1, Arch-0.2, Arch-0.3
  - Rationale: Human specified 4 preferences — maximal decoupling (atomic deployments), serverless-first (scale-to-zero, no reserved capacity), minimal storage (ephemeral only), unpredictable traffic handling. All 3 open discussion items resolved accordingly.
  - Impact: All compute elements are now serverless functions. Upload/download is direct-to-storage via pre-signed URLs. Format Registry is a standalone service. All elements independently deployable. Architecture is extensible to container-based workers for future long-running conversions.

## Superseded assumptions / changes
<!-- Optional. Use when previous assumptions were later invalidated. -->
- 2026-03-22: Element 20 previously allowed API-proxied file upload as an option
  - Related Discussions if any: Arch-0.1
  - Replaced by: Strictly direct-to-storage via pre-signed URLs; Element 20 never handles file content
  - Reason: Serverless-first architecture makes API-proxied uploads impractical (ephemeral compute, memory constraints)

- 2026-03-22: Element 30 previously described as potentially container-based or serverless
  - Related Discussions if any: Arch-0.2
  - Replaced by: Serverless functions for initial release; container option preserved as future extensibility path
  - Reason: Conversions are seconds-to-minutes; unpredictable traffic and no reserved capacity favor serverless

- 2026-03-22: Element 70 previously described as potentially embedded configuration
  - Related Discussions if any: Arch-0.3
  - Replaced by: Standalone serverless service backed by managed key-value store
  - Reason: Maximal decoupling preference — format changes must be deployable independently of API and Worker services
