# skills

Skills propias para agentes de código (Claude Code, OpenCode, Codex, Cursor, Antigravity, y otros). Pensadas, usadas y curadas en el día a día.

## Instalación

Cualquiera de los siguientes instaladores funciona. El más establecido es `skills` de Vercel Labs.

### Comando de instalación estándar

```bash
npx skills add 0tomi/skills
```

### Usos puntuales 

```bash
# Instalar una skill particular
npx skills add 0tomi/skills --skill planificar

# Instalar a un agente específico
npx skills add 0tomi/skills --skill planificar -a claude-code
npx skills add 0tomi/skills --skill planificar -a opencode
```
### Alternativa: openskills

```bash
npx openskills install 0tomi/skills
```

## Skills principales incluidas

| Skill | Descripción |
|---|---|
| [`planificar`](./skills/planificar) | Convierte un requerimiento en un plan de implementación ejecutable. Soporta modo secuencial y orquestación entre agentes. Conversacional: pregunta antes de comprometerse cuando hay ambigüedad real. |
| [`orquestacion-especializada-planes`](./skills/orquestacion-especializada-planes) | Explica al agente como orquestar de forma correcta subagentes a partir de un plan detallado. |
| [`gemini-cli`](./skills/gemini-cli) | Permite utilizar como subagentes los modelos de Antigravity en cualquier plataforma (Opencode, Codex, etc), siempre y cuando tengas agy instalado. |
| [`reduce-slop`](./skills/reduce-slop) | Skill para un revisor, provee patrones convencionales para reducir el slop del codigo generado por los agentes. |
| [`sveltekit-tauri`](./skills/sveltekit-tauri) | Da normativas para utilizar Svelte con Tauri, armado según la documentación oficial. |

## Estructura

```
skills/
├── <skill>/
│   ├── SKILL.md          # Núcleo: cuándo usar, cómo usar
│   └── references/       # Material complementario, cargado bajo demanda
```

Cada skill sigue el [Agent Skills spec](https://github.com/anthropics/skills): SKILL.md con frontmatter (`name`, `description`) y cuerpo en Markdown. Las referencias se leen on-demand para mantener el contexto liviano.

## Contribuciones

Sugerencias, issues y PRs bienvenidos. Si querés agregar una skill, seguí el formato de las existentes y abrí un PR.

## Licencia

MIT — ver [LICENSE](./LICENSE).
