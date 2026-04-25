# Environment Notes

Read this when you know which environment will execute the plan, or when you want environment-aware exploration and validation hints. If the environment is unknown or mixed, use the *Generic* section.

The goal of this file is **not** to teach the planner each environment's tooling — it is to nudge the plan toward commands and validation steps that fit the place where it will run.

---

## Generic / unknown environment

When you don't know where the plan will execute:

- Avoid commands tied to a specific shell or OS unless the repo signals one (e.g. `package.json` scripts, a `Makefile`, a `composer.json`).
- Prefer language-native commands (`npm test`, `pytest`, `php artisan test`, `cargo test`, `dotnet test`) over environment-specific wrappers.
- For exploration, prefer `grep`/`find` patterns that work in most POSIX shells.
- Mark validation steps as "run the project's standard test command" rather than baking in a specific runner you can't verify exists.
- In §1.3 Assumptions, note that the plan was written without knowing the executing environment.

---

## VSCode (with Claude, Copilot, Cursor, or similar in-editor agents)

**Strengths to lean on:**

- Workspace-wide search and references — the agent can verify symbol usage cheaply.
- Inline file editing with diff preview — favors small, reviewable patches.
- Integrated terminal — full shell access if the user grants it.
- Problems panel surfaces type / lint errors immediately after edits.

**Plan adaptations:**

- Validation steps can reference the Problems panel ("after the change, Problems panel should show no new errors in `<files>`").
- Phases can be sized to fit a single editor session — avoid phases that would require dozens of file switches.
- When in doubt about a symbol's usage, the plan can include a "Verify with workspace search" task instead of pre-deciding.

---

## Antigravity

**Strengths to lean on:**

- Multi-agent orchestration is a first-class concept — plans in *orchestration mode* land naturally here.
- Agents can run long, structured workflows with checkpoints.
- File and shell tools are available; agents can verify their own work.

**Plan adaptations:**

- Lean into orchestration mode when the work has independent branches; this is Antigravity's sweet spot.
- Phases that would benefit from isolated agent contexts (clean-room implementation, second-opinion review) should say so in §4.5 Orchestrator notes.
- Validation can include "spawn a verification agent against the contract from §4.3" when the contract is well-defined.

---

## OpenCode

**Strengths to lean on:**

- Terminal-first agent with full shell, edit, grep, and file tools.
- Works against a real working directory; the repo state is concrete, not simulated.
- Comfortable with multi-step plans executed in sequence.

**Plan adaptations:**

- Phases can include shell-level validation directly (`npm run build`, `pytest -k <name>`, etc.).
- Exploration can rely on `rg` / `grep` / `find` without hedging.
- For long plans, include explicit checkpoints — OpenCode handles them well, and they reduce blast radius if a phase goes sideways.

---

## Gemini CLI

**Strengths to lean on:**

- CLI agent with file and shell access.
- Strong at reading and synthesizing across files when given the right pointers.

**Plan adaptations:**

- Be explicit about which files each phase reads; do not rely on the agent re-discovering surface area mid-phase.
- Validation steps should be concrete shell commands, not "verify it works".
- Long-form per-phase prose is fine; this CLI handles it without losing track.

---

## Codex CLI

**Strengths to lean on:**

- CLI agent with file editing and shell execution.
- Works well against well-scoped, small-diff phases.

**Plan adaptations:**

- Prefer phases that produce small, atomic diffs over phases that touch many files.
- Surface validation hooks (lint, type-check, test) explicitly per phase — Codex does best when each phase ends with a verifiable command.
- For exploration, list the specific files to read; broad "explore the repo" tasks are less efficient here.

---

## Cross-environment notes

Regardless of environment:

- Do not assume a particular OS unless the repo proves it (e.g. PowerShell-specific scripts, `.bat`/`.cmd` files, Inno Setup → Windows).
- If the repo includes Docker, devcontainers, or CI definitions, validation steps should match those rather than inventing local-only commands.
- When the plan will be handed to an agent in an environment that *isn't* in this list, default to the *Generic* section and flag the assumption in §1.3.
