# STATE

This file tracks large, "hot topics" that are ongoing. It may not be active all the time.

## State management rules
- Keep this file focused on current execution state. Put rollups and resolved items in `HISTORY.md`.
- You only use this file when major blockers/issues are present.

## Current focus
- Target element: 70
- Revision ID: Comp.70.0.1

## Objective (current breakdown)
Define the component-level design for the Format Registry so runtime consumers can validate conversion support and retrieve option metadata through an independently deployable, read-only service.

## Active assumptions / constraints
- Architecture review is already marked DONE in `vibeCoding/.architecture/STATE.md`, so component design can proceed without gating.
- Registry updates are rare and should flow through controlled publication rather than runtime write APIs.
- The design should preserve independent deployability for Element 70 while remaining lightweight and serverless-friendly.

## Active issues
- (all current issues resolved; component draft for Element 70 is ready for review)
