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

## Skills

Organizadas por carpeta; cada una tiene su README con el índice de skills.

| Carpeta | Skills |
|---|---|
| [Agentes/Delegación](./skills/Agentes/Delegación) | `orquestar-planes`, `codex-delegation` |
| [Agentes/Subagents](./skills/Agentes/Subagents) | `claude-cli`, `codex-cli`, `gemini-cli` |
| [Código - Implementaciones/Planificación](./skills/Codigo%20-%20Implementaciones/Planificación) | `prototype-html`, `planificar` |
| [Código - Implementaciones/Reviews](./skills/Codigo%20-%20Implementaciones/Reviews) | `reduce-slop` |
| [Código - Implementaciones/Frameworks](./skills/Codigo%20-%20Implementaciones/Frameworks) | `sveltekit-tauri` |
| [Estudiar](./skills/Estudiar) | `apunte-interactivo` |
| [Varios](./skills/Varios) | `commit` |

## Estructura

```
skills/
├── <Categoría>/
│   ├── README.md             # Índice de la categoría
│   └── <skill>/
│       ├── SKILL.md          # Núcleo: cuándo usar, cómo usar
│       └── references/       # Material complementario, cargado bajo demanda
```

Cada skill sigue el [Agent Skills spec](https://github.com/anthropics/skills): SKILL.md con frontmatter (`name`, `description`) y cuerpo en Markdown. Las referencias se leen on-demand para mantener el contexto liviano.

## Contribuciones

Sugerencias, issues y PRs bienvenidos. Si querés agregar una skill, seguí el formato de las existentes y abrí un PR.

## Licencia

MIT — ver [LICENSE](./LICENSE).
