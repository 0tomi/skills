---
name: prototype-html
description: Prototype proposed changes — or explain a topic you want to learn — as a
  single interactive HTML file you open locally to tantear before planning.
disable-model-invocation: true
---

# Prototype HTML

Explore the repo, formulate a plan of proposed changes, and prototype those changes in a single interactive HTML file the user opens locally to *tantear* them — feel each change before committing to it. This is pre-planning: when the prototypes convince, the user invokes `planificar` for the executable plan.

This skill is a guide, not a template. Use what fits, drop what doesn't — except the two hard rules below.

---

## Hard rules

**1. The output is always a single standalone HTML file.** CSS and JS inline. No external assets beyond what loads from the file alone — no remote fonts, no CDNs, no fetches. The user opens it locally in a browser; this is not a published page.

**2. Lean on `frontend-design` (or a design skill the user named in the prompt) for the visual layer.** Before drafting, check it's available — `<available_skills>`, project skill directories (`.claude/skills/`, `.agents/skills/`, etc.), or wherever skills are surfaced here. If a design skill is reachable, use it. If none is, build with your own design judgement — don't block on it.

Everything else here is suggestion.

---

## Ground it in the repo

This skill usually runs outside any plan mode the CLI offers, so the legwork is yours: explore before drafting. Prototypes of a screen that doesn't match the actual surface, or changes proposed against files that don't exist, kill the artifact's usefulness.

If the conversation already shows file reads, greps, or pasted code covering what the prototypes will touch, skip ahead. If the idea is pure greenfield with no codebase anchor, skip too. Otherwise, see `references/exploration.md` for how to inspect the repo and how deep to go.

---

## The plan of changes structures the HTML

Formulate the changes you propose before writing markup — that list is the artifact's skeleton. Each proposed change gets a section carrying:

- **What and why** in 2–3 lines.
- **The prototype** — the change shown, not described: SVG, HTML/CSS frames, annotated mockups.
- **An approve/adjust affordance** — tabs, buttons, or controls that let the user signal direction (saying it in chat works too).

Open the file with a one-screen overview: goal and context in 2–4 lines, fixed constraints, and the list of proposed changes — the user shouldn't scroll to learn what this is. Close with open questions when something is deliberately unresolved, so the conversation knows where to continue.

---

## Conversation first

The HTML is a living artifact, iterated across turns. Treat each version as a draft worth questioning.

**Ask before drafting when ambiguity would change the artifact's shape** — the surface (mobile? web? which screens?), the direction (one approach or several to compare?), or the scope. One round of sharp questions beats an artifact that guessed.

**Bake cheap assumptions in and move on** — the user corrects them by chat or through the artifact's own controls. **Surface alternatives when meaningfully better** — as a question, not a redirect. If the environment offers structured input pickers, prefer them for option-picking questions.

---

## Handling gaps

When something the prototypes depend on is unknown:

- **Ask** — if the answer would change the artifact's shape.
- **Assume** — when a sensible default works; flag it in an assumptions strip inside the artifact.
- **Block** — when prototyping honestly isn't possible (can't find the surface, can't tell what to preserve). Say what's missing and name the smallest exploration that unblocks you.

---

## Pick a shape

Don't default to one layout out of habit:

- **Exploration grid** — N directions for a change side by side, each with a micro-mockup and a one-line tradeoff. For when the direction isn't decided.
- **Single spec** — one change in depth: flow, key screens, edge cases. For when the call is made and detail needs to land.
- **Decision-first** — alternatives → tradeoffs → recommendation, then depth on the chosen one. For when a call has to land before the detail makes sense.
- **Design playground** — interactive controls the user fiddles with to find the final values. For tuning, not deciding.
- **Research / explainer** — diagram-led synthesis the user learns from. For when they don't know the territory yet and want to understand it before deciding anything.

Mix shapes when it helps (decision-first → single spec is common). See `references/patterns.md` for skeletons.

---

## Interactivity

Inline JS earns its place when it lets the user *feel* a change — tabs across A/B/C versions, sliders rerendering a preview, editable fields with a live summary. Controls that don't visibly change anything are noise.

Where a control holds state worth carrying out (tuned values, picked options), give it a small copy button: the clipboard is the only channel back to the agent, and the user pastes those values when invoking `planificar`. See `references/interactivity.md` for patterns.

---

## Smell tests

An artifact worth shipping avoids these:

- Decorative interactivity (a control that changes nothing visible).
- Lorem-ipsum prototypes too generic to be for anything specific.
- Padding to look thorough — ceremonial sections, repeated info across blocks.
- A 4000-line HTML where 400 would say more.

---

## References

Load only when needed:

- `references/exploration.md` — inspecting the repo before prototyping, when it hasn't been explored.
- `references/patterns.md` — skeletons for the five shapes.
- `references/interactivity.md` — JS patterns for controls, live previews, and copy buttons.
