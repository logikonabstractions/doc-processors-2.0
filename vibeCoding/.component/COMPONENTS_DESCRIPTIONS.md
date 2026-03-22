# COMPONENTS DESCRIPTIONS

The response must provide a component-level design for **one parent architectural element**.

# PROBLEM STATEMENT

## Parent architectural element

- Parent ID:
  - <10 | 20 | 30 | ...>
- Parent name:
  - <name from architecture description>
- Parent purpose:
  - <short summary of the parent architectural role>

## Assumptions
- <assumptions>

## Constraints
- <important constraints: inherited from the architecture, inherent to tech choices, etc.>

## Component breakdown

Repeat & fill this template as needed. Follow the numbering convention.

### <ARCHITECTURAL_ELEMENT_ID><COMPO_ID> — <Component name>

- Category:
  - <Frontend system / API service / worker / database / queue / adapter / storage / external integration / observability / other>
- Purpose:
  - <why this component exists>
- Technology choice:
  - <specific framework / product / runtime / service / library>
- Responsibilities:
  - <responsibilities>
- Interfaces:
  - Incoming:
    - <requests / commands / events / data>
  - Outgoing:
    - <requests / commands / events / data>
- Data / state:
  - <what this component owns, persists, caches, or reads>
- Dependencies:
  - Internal:
    - <other x.y components>
  - External:
    - <architectural elements outside the parent scope, if relevant>
- Observability / operational considerations:
  - <logs / metrics / tracing / admin / scaling / failure visibility>
- Constraints / notes:
  - <important implementation constraints>
- Principal alternative (optional):
  - <close second option and why it was not chosen>

## Open questions

- <question requiring human/component-design decision>
