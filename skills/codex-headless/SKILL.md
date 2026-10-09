---
name: codex-headless
description: Invocar subagentes de Codex. Usar si el usuario lo indica. Permite usar `--dangerously-bypass-approvals-and-sandbox` solo con permiso explícito del usuario.
---

# Codex CLI (headless)

Para delegar una tarea sin abrir la interfaz interactiva, usar `codex exec` (alias `codex e`). El proceso ejecuta la tarea, muestra el progreso en `stderr` y entrega normalmente la respuesta final en texto plano por `stdout`.

## Modelo recomendado

- **Modelo:** GPT-6.1 Sol (`gpt-6.1-sol`)
- **Effort:** `medium`

## Ejecución

```bash
codex exec -C /ruta/proyecto -m gpt-6.1-sol -c model_reasoning_effort=medium \
  "Analizá el módulo de autenticación y resumí los problemas encontrados."
```

Para permitir modificaciones en el proyecto:

```bash
codex exec -C /ruta/proyecto -s workspace-write \
  "Corregí el error, ejecutá las pruebas relevantes y resumí los cambios."
```

Parámetros principales:

| Parámetro | Función |
|---|---|
| `PROMPT` | Instrucción a ejecutar. |
| `-` | Leer el prompt desde `stdin`. |
| `-C RUTA` | Directorio del proyecto. |
| `-s MODO` | Sandbox: `read-only`, `workspace-write` o `danger-full-access`. |
| `-m MODELO` | Elegir modelo. |
| `-c clave=valor` | Sobrescribir una opción de configuración. |
| `-o RUTA` | Guardar el mensaje final en un archivo. |
| `--dangerously-bypass-approvals-and-sandbox` (`--yolo`) | Omitir aprobaciones y sandbox; solo con permiso explícito del usuario. |
| `--skip-git-repo-check` | Ejecutar fuera de un repositorio Git. |
| `-i RUTA` | Adjuntar una imagen al prompt. |

Para pasar un prompt largo desde un archivo:

```bash
codex exec -C /ruta/proyecto - < tarea.md
```

Para conservar el resumen final:

```bash
codex exec -C /ruta/proyecto -o /tmp/codex-resumen.txt \
  "Revisá los cambios y detallá los hallazgos."
```

Usar `read-only` para consultas y `workspace-write` para tareas que requieran edición. Evitar `danger-full-access` y `--dangerously-bypass-approvals-and-sandbox` salvo autorización explícita del usuario; el último también omite las confirmaciones.

## Continuar una sesión

```bash
codex exec resume <SESSION_ID> "Continuá con las correcciones pendientes."
```

Para retomar la última sesión:

```bash
codex exec resume --last "Continuá la tarea anterior."
```

Preferir el ID explícito si hay varias tareas o proyectos. No utilizar `--ephemeral` en ejecuciones que deban retomarse.

## Cuota agotada

Si Codex falla, revisar el código de salida y `stderr`. Clasificar como **`QUOTA_EXHAUSTED`** únicamente cuando el error indique explícitamente que se agotó la cuota, por ejemplo `You've hit your usage limit` o `usage limit has been reached`.

Ante **`QUOTA_EXHAUSTED`**, detener nuevas delegaciones con esas credenciales e informar al usuario. Si el error incluye una fecha de restablecimiento, reproducirla sin inventar plazos. Un `429` genérico no confirma cuota agotada: también puede ser una limitación temporal de solicitudes.

Consulta manual de límites: https://chatgpt.com/codex/settings/usage

## Resultado de la delegación

Tomar el texto final de `stdout` (o del archivo indicado con `-o`) y transmitir al usuario el resultado, los cambios realizados y cualquier error. Verificar los cambios y pruebas relevantes antes de presentarlos como exitosos.
