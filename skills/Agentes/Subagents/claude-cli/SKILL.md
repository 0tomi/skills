---
name: claude-cli
description: "Invocar subagentes de Claude/Claude Code. Usar solo si el usuario lo indica y si no sos Claude."
---

# Claude CLI (headless)

Delegar con `claude -p`: espera a que Claude termine y devuelve la respuesta final por `stdout`.

```bash
claude -p "<PROMPT>" --model sonnet --effort high --permission-mode auto
```

Se ejecuta en el directorio actual; para dar acceso a otro, `--add-dir <RUTA>`.

## Modelos

Los alias `sonnet` y `opus` apuntan siempre a la última versión: no hace falta buscar slugs.

| Modelo | Effort | Uso | Ejemplo |
|---|---|---|---|
| Sonnet | `high` | Por defecto. | `--model sonnet --effort high` |
| Opus | `medium` | Tareas críticas, pesadas o que exigen más criterio. | `--model opus --effort medium` |

## Permisos

Usar siempre `--permission-mode auto`: aprueba ediciones y comandos seguros sin pedir confirmación.

`--dangerously-skip-permissions` solo con permiso explícito del usuario.

## Duración

Una tarea puede tardar mucho: lanzarla en segundo plano o con un timeout de 1 h, nunca con el timeout por defecto de la herramienta.

## Continuar una sesión

Si la sesión se va a retomar, fijar el ID al invocarla con `--session-id <UUID>` (generarlo con `uuidgen`) y retomarla con:

```bash
claude -p "<PROMPT>" --resume <UUID> --model sonnet --effort high --permission-mode auto
```

`-c` (`--continue`) en lugar de `--resume` retoma la última sesión del directorio. Preferir invocaciones independientes salvo que el contexto previo sea necesario.

## Cuota agotada

Es cuota agotada solo si el error lo dice explícitamente (usage limit); un error genérico o de sobrecarga puede ser temporal. En ese caso, dejar de delegar e informar al usuario, citando la hora de restablecimiento si el error la trae.

## Resultado

Transmitir al usuario la respuesta, los cambios y los errores. Verificar cambios y pruebas antes de presentarlos como exitosos.
