---
name: prototype-html
description: Prototype proposed changes — or explain a topic you want to learn — as a
  single interactive HTML file you open locally to tantear before planning.
disable-model-invocation: true
---

# Prototype HTML

Explore the repo, formulate the changes you propose, and prototype them in a single interactive HTML file the user opens locally to *tantear* — feel each change before committing to it. This is pre-planning: when the prototypes convince, the user invokes `planificar`.

A guide, not a template: use what fits, except the hard rules.

## Hard rules

1. **One standalone HTML file.** CSS and JS inline; no remote fonts, CDNs or fetches. It opens locally, it is not a published page.
2. **Visual layer from `frontend-design`** (or the design skill the user named). Check it's reachable — `<available_skills>`, `.claude/skills/`, `.agents/skills/` — and use it. If none is, use your own design judgement.
3. **The layout does the explaining.** Every label, tag, badge, subtitle or caption must carry information the layout doesn't already give; one that restates a heading or names the obvious goes. The prototype shows, it doesn't annotate itself.

## Ground it in the repo

The legwork is yours: explore before drafting. A mockup of a screen that doesn't match the real surface, or changes against files that don't exist, kills the artifact. Skip when the conversation already covers what the prototypes touch, or the idea is pure greenfield. Otherwise follow `references/exploration.md`.

## Ask, assume, or block

When something the prototypes depend on is unknown:

- **Ask** when the answer changes the artifact's shape — surface (mobile? web? which screens?), direction (one approach or several?), scope. One round of sharp questions, structured pickers if the environment has them.
- **Assume** when a sensible default works; note it in an assumptions strip. The user corrects by chat or through the artifact's controls.
- **Block** when prototyping honestly isn't possible. Say what's missing and the smallest exploration that unblocks it.

Surface a meaningfully better alternative as a question, not a redirect. The HTML is a living draft, iterated across turns.

## The plan of changes is the skeleton

Formulate the proposed changes before writing markup. The file:

- **Opens with a one-screen overview:** goal in 2–4 lines, fixed constraints, the list of changes.
- **One section per change:** what and why in 2–3 lines, then the prototype — SVG, HTML/CSS frames, annotated mockups. Shown, not described.
- **Closes with open questions** when something is deliberately unresolved.

## Pick a shape

- **Exploration grid** — N directions side by side, for when the direction isn't decided.
- **Single spec** — one change in depth, for when the call is made.
- **Decision-first** — alternatives → tradeoffs → recommendation, for when a call must land before detail.
- **Design playground** — controls to tune final values.
- **Research / explainer** — diagram-led synthesis, for learning the territory first.

Mix freely (decision-first → single spec is common). Skeletons in `references/patterns.md`.

## Interactivity

A control earns its place when it lets the user *feel* a change — tabs across versions, sliders rerendering a preview, editable fields with a live summary. A control that changes nothing visible is noise. Patterns and copy buttons (the clipboard is the only channel back to `planificar`) in `references/interactivity.md`.

## Done when

- Every proposed change has a section the user can react to.
- Every control visibly changes something.
- No ceremonial sections, repeated info or generic lorem-ipsum mockups — 400 lines that say it beat 4000 that pad it.
