---
name: planificar
description: Use this skill whenever the user asks to plan, planificar, draft a plan, design an implementation approach, break a feature down into phases, scope work before coding, or prepare a roadmap for another agent or developer to execute. Triggers on Spanish phrases like "planificá", "armá un plan", "hacé un plan de implementación", "necesito un plan", and English equivalents like "plan this", "draft a plan", "break this down". Also use when the user wants a plan suitable for orchestrating multiple agents in parallel. The skill produces an executable, phased, verifiable implementation plan grounded in the actual repository — not a generic brainstorm.
---

# Planificar — Implementation Planning Skill

## Purpose

Turn a requirement (functional or technical) into an **executable, verifiable, low-risk implementation plan** that another agent or developer can follow as a recipe.

A plan produced with this skill should:

- be grounded in the **actual** state of the repo, not an idealized one;
- name files, components, dependencies and risks concretely;
- break work into phases small enough to validate independently;
- make assumptions, gaps and decisions explicit;
- contain only changes that are technically justified — no cosmetic refactors smuggled in.

If the output reads like a nice brainstorm but cannot be executed step-by-step, it failed. That is the bar.

---

## How to use this skill

The skill has a small core (this file) and four reference files loaded **only when relevant**. Read them on demand:

| Reference | Read when |
|---|---|
| `references/exploration.md` | The repo has not been explored yet, or you are uncertain about its structure. Defines the exploration gate. |
| `references/orchestration.md` | The user mentions orchestrating agents, parallel execution, delegating to multiple models, or splitting work across agents. |
| `references/environments.md` | You know which environment will execute the plan (VSCode, Antigravity, OpenCode, Gemini CLI, Codex CLI) and want environment-specific tool hints. |
| `references/visual-tools.md` | You are unsure which visual representation (table, Mermaid, list, contract block, etc.) fits the section you are writing. |
| `references/examples.md` | You want a worked example of a small, medium, or orchestrated plan before drafting your own. |

Do not load all of them by default. Load what you need.

---

## Step 0 — Decide the plan mode

Before writing anything, decide which mode applies. The mode shapes the rest of the plan.

1. **Sequential mode (default).** The user wants a detailed plan for one executor. Phases run in order.
2. **Orchestration mode.** The user explicitly mentions orchestrating agents, parallelizing, delegating to multiple models, or asks for a plan that can be split across agents. Read `references/orchestration.md` and add the orchestration sections it specifies.

If the user says nothing about orchestration → sequential. Do not invent parallelism the user did not ask for.

If the user explicitly says "no orchestration" or "secuencial" → sequential, and skip orchestration sections entirely.

---

## Step 1 — Exploration gate (skip only if already explored)

A plan written without reading the code is a guess. The exploration gate ensures it is not a guess.

**Skip exploration if all of these are true:**

- The current conversation already shows file reads, greps, repo listings, or the user pasted relevant code.
- The files that the plan will touch have been seen — not just named.
- The user did not ask for a fresh re-exploration.

**Otherwise, explore first.** Read `references/exploration.md` for the checklist and environment-aware command hints. Do not write the plan body until exploration produces enough signal to ground decisions.

If exploration is impossible (no repo access, restricted environment, etc.), state this explicitly in the plan's *Assumptions* section and flag every decision that depends on unverified context.

---

## Step 2 — Write the plan

Use the structure below. **Every section is required**, but the visual representation inside each section is your call — see `references/visual-tools.md` for guidance on what fits where. Do not force a table or a diagram if prose says it better. Do not skip a section because "it's obvious".

### Required structure

```
# Implementation Plan

## 1. Scope

### 1.1 Goal
2–5 lines: what gets built and what the technical end-state looks like.

### 1.2 Non-goals
Bullet list. What is explicitly out of scope. Prevents drift.

### 1.3 Assumptions and constraints
What the plan is taking for granted, and what it cannot change.
Mark assumptions that, if wrong, would invalidate the plan.

### 1.4 Affected surface
A single representation (table, list, or grouped tree) covering every file
or module the plan touches, with state: Create | Modify | Delete | Review.
Do NOT repeat this list per phase — phases reference it.

### 1.5 Initial risks
Bullet list of risks that span multiple phases or the plan as a whole.
Per-phase risks live inside each phase.

### 1.6 Overall size
S | M | L, with a 1–2 line technical justification.
See "Sizing heuristics" below.

## 2. Phases

For each phase, use the phase template below.
Phase count should match the work — do not pad with ceremonial phases,
do not collapse two unrelated changes into one phase to look tidy.

## 3. Execution guidance

### 3.1 Advance rule
Do not start phase N+1 until phase N validates.
If validation fails: fix in place, update the plan if an assumption broke,
do not paper over it.

### 3.2 Per-iteration minimums
What every iteration must verify before being considered done.
Adapt to the project (compile, type-check, test, manual smoke, etc.).

### 3.3 Correction policy
What to do when something unexpected appears mid-implementation.

## 4. Orchestration (only in orchestration mode)
See references/orchestration.md.
Omit this section entirely in sequential mode.
```

### Phase template

Each phase **must** include all of these. The visual format inside each is flexible.

```
### Phase N — <Short name>

**Goal.** One sentence. The concrete result that exists when this phase ends.

**Preconditions.** What must be true before starting (prior phases done, env ready, etc.).

**Inputs / Outputs (contract).**
- Inputs: what this phase consumes (files, types, data, decisions from earlier).
- Outputs: what this phase produces (new files, modified contracts, migrations).
This block is the contract another agent will use to consume the phase. Take it seriously.

**Tasks.** Atomic, verb-led, executable steps. Avoid "improve", "adjust", "refactor"
without a target. If a task is not executable as written, split it.

**Touched files.** Reference §1.4 by listing only the files relevant to this phase.
If the action differs from §1.4 (e.g. a Review file becomes a Modify), say so here.

**Validation.** Concrete acceptance criteria. Examples:
- compiles / type-checks
- specific test passes (name it)
- happy path produces X
- known edge case Y handled
- no regression in Z

**Risks.** Things that can break specifically inside this phase.

**Complexity.** S | M | L, plus 1 line of why.
This per-phase complexity matters: it informs which agent or model handles the phase
in orchestration mode, and surfaces hot spots in sequential mode.

**Result.** One line: what exists or works at phase end.
```

---

## Sizing heuristics

Used both for the overall plan (§1.6) and per phase.

**Small (S).** Few files, local change, no contract or architecture impact, validation is direct.
*Examples:* a targeted validation, a single endpoint, a query fix, a localized bug.

**Medium (M).** Touches multiple layers, some cross-module coordination, meaningful business logic, broader tests.
*Examples:* a new business flow, a frontend↔backend contract change, adding internal persistence.

**Large (L).** Multiple domains/services, migrations, rollout or backward-compat concerns, high regression risk.
*Examples:* replacing a core component, redesigning auth, introducing queues or eventing.

If a single phase is L, consider splitting it. A plan full of L phases is a plan that has not been broken down.

---

## Quality bar

A plan from this skill must be:

- **Specific** — names files, modules, functions, not "the relevant code".
- **Grounded** — references things that actually exist (or explicitly marks them as new).
- **Incremental** — each phase ends in a verifiable state.
- **Honest** — surfaces risks, assumptions, and gaps instead of hiding them.
- **Sober** — no unrequested refactors, no padding, no decorative diagrams.
- **Right-sized visually** — uses tables, diagrams, lists, or prose **only where they communicate better than the alternative**.

Smell tests for a bad plan:

- Vague tasks ("adapt the service").
- Phases that span unrelated concerns.
- No mention of validation.
- A diagram per phase whether or not it adds anything.
- File names that the agent never actually opened.
- Claims of "no risk" on changes that obviously have risk.

---

## Ambiguity handling

If the requirement has gaps, pick **one** of these explicitly:

- **Plan with explicit assumptions.** Use when work can still be structured reasonably. List the assumptions in §1.3 and flag the ones that, if wrong, change the plan.
- **Plan with alternatives.** Use when there are 2–3 plausible technical approaches. Lay them out, recommend one, give the reason. Do not produce parallel plans for each — pick a recommendation.
- **Block.** Use only when there is no honest way to plan: stack unknown, no entry point, no way to know what compatibility must be preserved. Say what is missing, name the smallest exploration that would unblock you, then stop.

Do not silently fill gaps with plausible-sounding guesses. That is the failure mode this skill exists to prevent.

---

## Output checklist (run before delivering the plan)

- [ ] Mode chosen (sequential or orchestration) and matches what the user asked for.
- [ ] Exploration done, or explicitly marked as skipped with reason.
- [ ] §1.4 lists every file the plan touches, exactly once.
- [ ] Every phase has Goal, Preconditions, Contract, Tasks, Validation, Risks, Complexity, Result.
- [ ] No phase contains tasks that cannot be executed as written.
- [ ] No diagrams, tables, or sections added "for show" — each one earns its place.
- [ ] Assumptions are flagged when wrong-assumption invalidates the plan.
- [ ] Sizing (overall and per phase) is justified, not just labeled.
- [ ] Orchestration section present iff in orchestration mode.

If any item fails, fix before delivering.
