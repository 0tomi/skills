# Interactivity

Inline JS patterns for the standalone HTML. The bar: a control earns its place when it changes something visible the user couldn't feel from prose alone.

## Patterns that pull weight

- **Tabs across alternatives** — when 2–4 versions of the same thing live in the artifact and only one should show at once. Keep semantic markup (`role="tablist"`, `aria-selected`) so keyboard nav works.
- **Sliders / inputs that rerender a preview** — for tuning values (timing, density, weight, count). Mutate CSS variables for purely visual tuning; reach for DOM re-renders only when the change is structural (cards added/removed, layout flipping).
- **Editable fields with a live summary** — fields write to a state object; a readout updates from it so the cumulative effect of choices is visible.
- **Drag-and-drop ordering** — for curation work (prioritize, bucket, triage). Native HTML5 DnD is enough. For short lists, up/down buttons are simpler than drag.

## Copy buttons

Where a control holds state worth carrying out — tuned values, picked options, edited text — give it a small copy button (JSON, CSS variables, or a short line of values). The clipboard is the only channel back to the agent: the user pastes those values when invoking `planificar`. Render what will be copied live, so the user sees it before clicking; `navigator.clipboard.writeText` with a brief "Copied" toggle is enough confirmation. Controls whose state the user can just say in chat ("me quedo con la B") don't need one.

## Architectural constraints

- One state object, one render function. Don't sprawl into framework-shaped architecture for what's essentially a form.
- Vanilla DOM APIs only — no inlined frameworks or large utility libs just to save a few lines.
- `localStorage` only when the artifact is meant to be revisited across sessions; for one-shot iteration, skip persistence.
