# Visual Tools — When to Use What

A plan should use the representation that **communicates fastest**, not the one that looks most thorough. This file is a cheat sheet for choosing.

The default is prose. Reach for a structured representation only when prose would be longer, vaguer, or harder to scan.

---

## Tables

**Use when:** comparing N items across the same dimensions. Files × state, phases × parallel group, alternatives × tradeoffs.

**Skip when:** the "table" would have one row, or rows whose cells are mostly empty, or where each row is a paragraph (use a list with sub-bullets instead).

**Examples that earn a table:**

| File | State | Reason |
|---|---|---|
| `src/auth/login.ts` | Modify | Add MFA branch |
| `src/auth/mfa.ts` | Create | New TOTP verifier |
| `tests/auth/login.test.ts` | Modify | Cover MFA path |

| Phase | Group | Depends on | Profile |
|---|---|---|---|
| 1 | G1 | none | standard |
| 2 | G2 | 1 | light |
| 3 | G2 | 1 | standard |

**Examples that should not be a table:**

A 2-row table with one column that says "yes" twice. Just write a sentence.

---

## Mermaid diagrams

**Use when:** the relationship is genuinely graph-shaped *and* the reader would have to mentally reconstruct it from prose.

**Skip when:** the flow is linear (use a numbered list), the diagram has 3 nodes (use a sentence), or the diagram is decorative.

**The honest test:** if you removed the diagram, would the reader miss something they needed? If no, remove it.

**Earns a Mermaid diagram:**

- A dependency graph with branching and merging (orchestration §4.1).
- A non-obvious data flow across 4+ components.
- A state machine with transitions worth seeing.

**Does not earn a Mermaid diagram:**

- "Input → Service → Database". That's a sentence.
- One per phase, just because the template suggested it.
- A flowchart of a function that already reads top-to-bottom.

---

## Bullet / numbered lists

**Use when:** sequence or enumeration of short items. Tasks, risks, validation criteria, assumptions.

**Tasks** — numbered if order within a phase matters; bulleted if not. Each item should be:

- a verb-led action (`Add validation for empty email in LoginForm`)
- atomic (one mental unit)
- verifiable (when it's done, you can tell)

**Bad task list:**
- Improve auth
- Adjust the service
- Make sure things work

**Good task list:**
- Add `validateEmail()` to `LoginForm.tsx`, returning `{ ok: boolean, reason?: string }`.
- Wire `validateEmail` into the submit handler before the API call.
- Add unit test for empty, malformed, and valid inputs in `LoginForm.test.tsx`.

---

## Contract blocks

**Use when:** describing what a phase consumes / produces, especially in orchestration mode. Prefer a small fenced block over a paragraph.

```
Phase 2 contract
  Inputs:
    - Order model from Phase 1 (fields: id, total, status)
    - Migration 2026_xx_create_orders.sql applied
  Outputs:
    - POST /orders endpoint, body { items[], customerId }, response { id, status }
    - Error shape { code, message }
```

This format works because the next agent can copy it verbatim and check each line.

---

## Prose

**Use when:** the content is reasoning, justification, nuance, or a single decision.

The §1.1 Goal, §1.3 Assumptions narrative, sizing justification, and orchestrator notes are usually best as prose. Do not bullet-point a paragraph just to look "structured".

---

## Code / config snippets

**Use when:** the plan needs to fix a contract, type, schema, or config exactly. A 4-line type definition is clearer than a paragraph describing it.

**Skip when:** the snippet is illustrative rather than prescriptive. If the implementer should write it themselves, describe the shape instead of the code.

---

## Formatting hygiene

- Headings only at section boundaries the template defines. Do not invent extra heading levels for emphasis.
- Bold sparingly — for the field labels (`**Goal.**`, `**Risks.**`) and nothing else inside prose.
- One blank line between sections; do not stack multiple blank lines.
- File paths in backticks. Symbol names in backticks. Plain prose in plain text.

---

## Decision flow

When you're about to add a visual element, ask in order:

1. Would prose say this just as clearly in fewer lines? → use prose.
2. Is it a list of similar short items? → bullet or numbered list.
3. Is it N items × M dimensions, dense enough to fill a grid? → table.
4. Is it genuinely graph-shaped *and* hard to read as prose? → Mermaid.
5. Is it a contract another agent will consume? → fenced contract block.
6. Otherwise → prose.

If the answer to all of 2–5 is "no", the visual was decorative. Drop it.
