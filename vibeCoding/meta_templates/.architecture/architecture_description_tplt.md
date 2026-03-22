**Template file, copy but do not edit**

## Architectural elements

Repeat & fill this template as needed. Follow the numbering convention.

### 10 — <Element name>

- Category:
  - <client / domain service / orchestration / identity and access / data persistence / messaging / external integration / observability / other>
- Purpose:
  - <why this exists>
- Responsibilities:
  - <responsibility>
- Interfaces:
  - Incoming:
    - <requests / commands / events / user actions / upstream inputs>
  - Outgoing:
    - <responses / commands / events / downstream outputs>
- Data / state:
  - <what data or state this element owns, reads, writes, persists, or exposes>
- Interactions:
  - User-facing:
    - <if applicable>
  - Internal synchronous:
    - <if applicable>
  - Internal asynchronous:
    - <if applicable>
- Security / access considerations:
  - <authentication / authorization / trust boundary / sensitive data notes>
- Observability / operational considerations:
  - <logging / monitoring / auditability / failure visibility / admin concerns>
- Dependencies:
  - <Element IDs>
- Constraints / notes:
  - <important remarks>
- Principal alternative (optional)
  - 1-2 sentences. Do not systematically include with all elements. Only when there are strong reasons to consider 2 options as very close, describe your 2nd best choice here.

## System interaction summary

- Primary request / control paths:
  - <summary>
- Primary data flows:
  - <summary>
- Primary event flows:
  - <summary>

## System-wide concerns

- Security and access control:
  - <items>
- Reliability and recovery:
  - <items>
- Observability and operations:
  - <items>
- Performance and scalability:
  - <items>
- Compliance / audit / governance:
  - <items if relevant>

## Open questions
- <question requiring architectural decision>