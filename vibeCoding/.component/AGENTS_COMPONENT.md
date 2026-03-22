# COMPONENT Workflow Contract

## Purpose

Translate **one architectural element** into a **component-level design**.

## Instruction precedence & read order
1. As specified by `AGENTS.md`
2. This file
3. `.component/COMPONENTS_DESCRIPTIONS.md`
4. `.component/STATE.md`
5. `.component/DISCUSSION.md`
6. `.component/HISTORY.md`
7. `.architecture/STATE.md` — specifically the `Key Architecture Decisions` section (read-only reference)
8. `.architecture/ARCHITECTURE_DESCRIPTION.md` (read-only reference)

## Meta-templates

Found under `/meta_templates/.component`

| File | Role |
|------|------|
| `/meta_templates/.component/components_description_tplt.md` | Output template and deliverable for the component design |
| `/meta_templates/.component/state_tplt.md` | Current focus, active blockers, work log |
| `/meta_templates/.component/history_tplt.md` | Resolved questions and completed component reviews |

## Scope

This mode receives **exactly one architectural element** and breaks it down into the concrete components required to implement it.

## Core output

The deliverable is a complete list of components, conform to the format in `COMPONENTS_DESCRIPTIONS.md`

## Input requirements

The prompt must provide or reference:
- the target architectural element number (e.g. "element 10")
- access to the current `.architecture/ARCHITECTURE_DESCRIPTION.md` (or its relevant section)

If the architecture has not been frozen (status ≠ FREEZE in `.architecture/STATE.md`), log a warning in `.component/STATE.md` but proceed unless explicitly told to stop.

**Architecture context**: before starting work, read the `Key Architecture Decisions` section in `.architecture/STATE.md`. This section contains the major orientations chosen during the architecture review (cloud provider, core patterns, technology families, key constraints). All component-level decisions must be consistent with these architecture-level decisions.

## Abstraction rules

Describe each component by **concrete role and technology**. A component is the unit of work required to implement part of the target architectural element.

The correct level of abstraction for a component (10.1, 10.2, ...) is one where:
- A specific technology (or set of technologies) is identified for implementation
- The component has a clear objective and coherent set of responsibilities
- It maps to a recognizable deliverable (a service, a schema, a configured runtime, a UI module, ...)
- It remains large enough to require multiple checkpoints to deliver (roughly the size of one or a few sprints)

For example, a component could be: "Authentication mechanism", with chosen technologies (e.g. OAuth 2.0 implemented with Passport.js, hashing with bcrypt, ...). It should not be "A social media app" (too broad) nor should it be "A sign-in form" (that would be a checkpoint to implement).

## Numbering rules

Components are numbered as sub-elements of their parent architectural element, appending a component_id to each (.1, 2., .3, ...)

- Architectural element 10 → components 10.1, 10.2, ..., 10.14...
- Architectural element 20 → components 20.1, 20.2, 20.3 ...

There is no fixed upper bound on component count — use as many as relevant.


## Question lifecycle

1. Create a `Comp-N.N` item in `.component/DISCUSSION.md`, following instructions provided there.