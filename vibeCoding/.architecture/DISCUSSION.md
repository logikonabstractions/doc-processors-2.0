# PLAN

## How to use this file

- This is the **architecture discussions**.
- Use it to track architecture questions that require clarification, comparison, investigation, or explicit decision.
- Keep one item per question or decision.
- When a question is resolved, keep the item here for traceability, mark it `RESOLVED`, and summarize the outcome in `HISTORY.md`.
- Any Arch-X.Y with a status `RESOVLED` should not be updated later. If the issue needs to be revisited, favor opening a new Arch-X.Y and reference the previous discussion.

## Question resolution

When a question is resolved:
  - Add a relevant entry to `HISTORY.md`
  - Update this file's status to `DONE` along with any field (notes, etc.)

## Active architecture issues

### Arch-0.0 — <short title>

- Type: <CLARIFICATION | DECISION | INVESTIGATION | ASSUMPTION_VALIDATION | RISK_REVIEW>
- Status: <OPEN | IN_PROGRESS | BLOCKED | DECISION_REQUIRED | RESOLVED>
- Related element(s):
  - <Archiecture elements name & ID>
- Description:
  - <why this question materially affects the architecture>
- Known options / hypotheses:
  - <option or hypothesis>
- Required input / evidence:
  - <what information, analysis, or human decision is needed>
- Resolution criteria:
  - <what must be true for this item to be considered resolved>
- Notes:
  - <optional>

## Response template

Use the following template to record a response to any Arch item.
Append the response block directly under the item it answers.

```
#### Response — Arch-<N.N>

- Respondent: <human | agent>
- Answer:
  - <direct answer, decision, or information provided>
- Additional Details:
  - <Context, caveats etc.>
```
