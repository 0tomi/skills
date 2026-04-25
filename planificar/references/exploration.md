# Exploration Gate

Read this when the planner has not yet explored the repo, or when uncertainty about repo state would make the plan a guess.

## When to skip exploration

Skip if **all** of these hold:

- The conversation already contains file reads, greps, listings, or pasted code covering the surface the plan will touch.
- The relevant files have been *seen*, not just *named*.
- The user did not ask for fresh re-exploration.

When in doubt, do a minimal exploration pass anyway. A 30-second listing is cheaper than a wrong plan.

## Exploration goal

Gather enough signal to answer these before writing the plan body:

1. **Where does the change live?** Concrete entry points, modules, files.
2. **What does it touch?** Direct dependencies and consumers of the code under change.
3. **What conventions exist?** Naming, layering, test patterns, error handling style — so the plan does not fight the codebase.
4. **What is already there?** Avoid proposing to "create" things that exist, or to "modify" things that don't.
5. **What is the test/validation surface?** What will be used to verify each phase.

If after exploration any of these is still unknown, say so in §1.3 (Assumptions) of the plan.

## Exploration checklist

Adapt to the project. Not every item applies every time.

- [ ] Top-level layout: identify project type, framework, languages, build tools.
- [ ] Locate the entry points relevant to the requirement.
- [ ] Read (not just list) the files that will be modified.
- [ ] Find direct callers / consumers of the code under change.
- [ ] Find existing tests covering the area.
- [ ] Check for related migrations, config, env vars, feature flags.
- [ ] Note conventions: how similar features are structured today.
- [ ] Note deviations or tech debt that constrain the plan.

## Command hints by environment

These are *suggestions*, not contracts. Use whatever the environment makes natural.

### Generic / unknown environment

If you have a shell:

```bash
# Layout
ls -la
find . -type f -name "*.<ext>" | head -50

# Locate symbols / usages
grep -rn "SymbolName" --include="*.<ext>"
grep -rn "function_name(" --include="*.<ext>"

# Read files in full before planning changes
cat path/to/file
```

If you only have a file-read tool, prioritize: project root listing → main config (package.json, composer.json, .csproj, etc.) → the most-likely-to-be-touched file.

### VSCode (with Claude / Copilot / similar)

- Use the workspace search across files for symbols and patterns.
- Open and read full files, not just the snippet around a match.
- Check the Problems panel for existing errors before promising a clean compile.

### Antigravity, OpenCode, Codex CLI, Gemini CLI

These environments expose tool primitives (read, glob, grep, bash). Use them directly:

- Glob the project to find candidates.
- Grep to confirm where a symbol is used and defined.
- Read full files; do not extrapolate from snippets.
- If running commands is allowed, prefer non-mutating commands during exploration (`ls`, `cat`, `grep`, language-specific *list* or *check* commands). Defer anything that writes or installs to the actual implementation phase.

For environment-specific notes (test runners, build commands, idioms), read `references/environments.md`.

## Exit criterion

Exploration ends when you can fill §1.4 (Affected surface) of the plan **with file paths you have actually opened**, and §1.3 (Assumptions) reflects what you still don't know. If §1.3 ends up listing critical unknowns that block the plan, do not write the plan — escalate as a *Block* per the ambiguity handling section in `SKILL.md`.
