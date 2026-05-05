# my-agents-skills

Skills propias para agentes de código (Claude Code, OpenCode, Codex, Cursor, Antigravity, Gemini CLI y otros). Pensadas, probadas y curadas en el día a día.

## Instalación

Cualquiera de los siguientes instaladores funciona. El más establecido es `skills` de Vercel Labs.

### Una skill puntual

```bash
# Instalar planificar
npx skills add 0tomi/my-agents-skills --skill planificar

# Instalar a un agente específico
npx skills add 0tomi/my-agents-skills --skill planificar -a claude-code
npx skills add 0tomi/my-agents-skills --skill planificar -a opencode
```

### Todas las skills del repo

```bash
npx skills add 0tomi/my-agents-skills --all
```

### Listar lo que hay sin instalar

```bash
npx skills add 0tomi/my-agents-skills --list
```

### Alternativa: openskills

```bash
npx openskills install 0tomi/my-agents-skills
```

## Skills incluidas

| Skill | Descripción |
|---|---|
| [`planificar`](./skills/planificar) | Convierte un requerimiento en un plan de implementación ejecutable. Soporta modo secuencial y orquestación entre agentes. Conversacional: pregunta antes de comprometerse cuando hay ambigüedad real. |
| [`orquestacion-especializada-planes`](./skills/orquestacion-especializada-planes) | (Descripción acá.) |
| [`my-agents`](./skills/my-agents) | (Descripción acá.) |
| [`commit`](./skills/commit) | (Descripción acá.) |

## Estructura

```
skills/
├── <skill>/
│   ├── SKILL.md          # Núcleo: cuándo usar, cómo usar
│   └── references/       # Material complementario, cargado bajo demanda
```

Cada skill sigue el [Agent Skills spec](https://github.com/anthropics/skills): SKILL.md con frontmatter (`name`, `description`) y cuerpo en Markdown. Las referencias se leen on-demand para mantener el contexto liviano.

## Compatibilidad

Probado o compatible con:

- Claude Code
- OpenCode
- Codex CLI
- Cursor
- Antigravity
- Gemini CLI
- VSCode con extensiones de agente
- Cualquier agente que respete el spec de Agent Skills

## Contribuciones

Sugerencias, issues y PRs bienvenidos. Si querés agregar una skill, seguí el formato de las existentes y abrí un PR.

## Licencia

MIT — ver [LICENSE](./LICENSE).
