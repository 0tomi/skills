# Subagents

Skills para delegar tareas a modelos de otras suscripciones, ejecutados en su propio harness, desde cualquier agente.

Sirven para:

- **Usar suscripciones restringidas a su propio harness** (Claude Code, Antigravity) desde otros harnesses como OpenCode, Pi o Codex.
- **Aprovechar modelos que rinden mejor en su harness nativo**, como los de Codex, cuando se invocan desde otro agente.

| Skill | Modelos | Empresa | Suscripción | CLI |
|---|---|---|---|---|
| `claude-cli` | Claude | Anthropic | Claude Code | `claude` |
| `codex-cli` | GPT | OpenAI | ChatGPT / Codex | `codex` |
| `gemini-cli` | Gemini | Google | Antigravity | `agy` |
