---
name: codex-cli
description: "Invocar subagentes de Codex o ChatGPT. Usar solo si el usuario lo indica y si no sos ChatGPT o Codex."
---

# Codex CLI (headless)

Delegar con `codex exec`: muestra el progreso en `stderr` y la respuesta final por `stdout`.

```bash
codex exec -C /ruta/proyecto -m gpt-6.1-terra -c model_reasoning_effort=high --approve-for-me \
  "Corregí el error, ejecutá las pruebas relevantes y resumí los cambios."
```

## Modelos

Usar la última versión de cada familia. Slug: `gpt-<VERSIÓN>-<familia>`; consultarlos en https://learn.chatgpt.com/docs/models. Si un slug falla por inválido, informarlo sin sustituirlo.

| Familia | Effort | Uso | Ejemplo |
|---|---|---|---|
| Terra | `high` | Por defecto. | `-m gpt-6.1-terra -c model_reasoning_effort=high` |
| Sol | `medium` | Tareas críticas, pesadas o que exigen más criterio. | `-m gpt-6.2-sol -c model_reasoning_effort=medium` |

## Permisos

Usar siempre `--approve-for-me` (modo *auto*): una revisión automática resuelve las aprobaciones y la ejecución queda en el sandbox `workspace-write`, así la delegación no se bloquea esperando confirmación.

- No combinarlo con `-s`: el modo ya fija el sandbox.

## Otros parámetros

| Parámetro | Función |
|---|---|
| `-` | Leer el prompt desde `stdin` (`codex exec -C /ruta - < tarea.md`). |
| `-o RUTA` | Guardar también el mensaje final en un archivo. |
| `-i RUTA` | Adjuntar una imagen. |
| `--skip-git-repo-check` | Ejecutar fuera de un repositorio Git. |

## Duración

Una tarea puede tardar mucho: lanzarla en segundo plano o con un timeout de 1 h, nunca con el timeout por defecto de la herramienta.

## Continuar una sesión

Codex imprime el `<SESSION_ID>` al terminar el turno. `resume` no acepta `--approve-for-me` ni `-C`: pasar el modo auto con `-c` y ejecutarlo desde el directorio del proyecto.

```bash
codex exec resume <SESSION_ID> \
  -c approval_policy=on-request -c sandbox_mode=workspace-write -c approvals_reviewer=auto_review \
  "Continuá con las correcciones pendientes."
```

`--last` en lugar del ID retoma la última sesión del directorio actual; preferir el ID si hay varias tareas. No usar `--ephemeral` en ejecuciones que deban retomarse.

## Cuota agotada

Es `QUOTA_EXHAUSTED` solo si `stderr` lo dice explícitamente (`You've hit your usage limit`, `usage limit has been reached`); un `429` genérico puede ser un límite temporal. En ese caso, dejar de delegar e informar al usuario, citando la fecha de restablecimiento si el error la trae. Límites: https://chatgpt.com/codex/settings/usage

## Resultado

Transmitir al usuario el texto final de `stdout` (o de `-o`), los cambios y los errores. Verificar cambios y pruebas antes de presentarlos como exitosos.
