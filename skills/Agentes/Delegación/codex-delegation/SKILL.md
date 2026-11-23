---
name: codex-delegation
description: Protocolo de delegación para que Codex elija qué modelo usar en cada subagente.
---

Al delegar una tarea del usuario, elegir el subagente así:

| Subagente | Cuándo |
|---|---|
| GPT Terra, effort `high` | Por defecto: tareas comunes, implementaciones medianas o largas de poca complejidad, exploraciones. |
| GPT Sol, effort `medium` | Tareas críticas o largas, y reviews de tareas críticas, complejas o muy largas. También cuando un Terra falló de forma estrepitosa: la tarea estaba subestimada. |
| Gemini, vía la skill `gemini-cli` | Revisiones de fases medianas o largas de poca complejidad. Equivale a un Terra, pero solo se usa para revisiones. |

Para revisiones de poca complejidad, Gemini va primero; Terra queda como fallback si Gemini falla o no tiene cuota.
