# agents.md — Overriding workflow contract

## Purpose

Use this repository as a **lightweight, human-in-the-loop planning system**. It supports 3 modes: **architecture**, **component** and **vibe** (which has two sub-modes: **vibe-draft** and **vibe-implement**). Each mode can be used in isolation. A logical flow can also be used, defining **architectural elements**, from each of which we can derive **components**. From each component, **vibe-draft** can be used to write Stages & Checkpoints, which can then be coded using **vibe-implement**. Or, one can directly write Stages & Checkpoints to **vibe-implement** them.

This file is the overriding, baseline workflow contract. Each mode-specific `AGENTS_<mode>.md` file extends this contract with mode-specific rules.

## Generic instructions
	- The mode (**architecture**, **component** or **vibe**) must be clearly specified. Either earlier in the conversation (e.g. it is clear from conversation history which mode is expected) or explicitly in the prompt. If you are unsure, you MUST ask to confirm.
	- Treat repository paths as **repo-root relative** unless a document says otherwise.

## Instruction precedence & read order

	1. User instructions in chat
	2. This file
	3. The mode-specific workflow contract for your task will detail the rest: `AGENTS_<mode>.md`

**Important**: you only need to read & consider this file and the files contained in your workflow's folder (`.vibe`, `.component`, `.architecture`). Ignore the rest unless specifically mentioned. (For instance, a **component** may refer to a specific **architectural element** from `.architecture`)

## History management

For all modes, `.<mode>/history.md` tracks advancement of the task. Append an entry when:
	- A status moves to `DONE`
	- A review round is completed
	- An important decision is made

Do **not** remove or rewrite earlier entries — history.md is append-only.

## State management

For all modes, `.<mode>/state.md` tracks current focus, active blockers, and work in progress. Update it whenever focus shifts or a blocker is added/resolved.

## plan.md conventions

Each mode uses a `plan.md` file, but its role differs by mode:
- **architecture** and **component**: `plan.md` is a **question & investigation backlog** — not an implementation plan.
- **vibe**: `plan.md` is an **ordered checkpoint plan** — the implementation backlog.

See each mode's `AGENTS_<mode>.md` for the specific plan.md semantics and templates.

## Meta-templates

All templates are found under `/meta_templates`. When inserting new sections (checkpoints, components, architectural element descriptions, history points, etc.) always start from the corresponding template for the mode of operation (e.g. `/meta_templates/.component` for component mode). Each mode file lists its specific templates.