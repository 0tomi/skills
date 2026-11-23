---
name: gemini-cli
description: "Invocar subagentes de Gemini/Antigravity/Agy. Usar solo si el usuario lo indica y si no sos Gemini."
---

# Gemini CLI (headless)

Delegar con `agy -p`: espera a que Gemini termine, muestra diagnósticos en `stderr` y la respuesta final por `stdout`.

```bash
agy -p "<PROMPT>" --model gemini-4-hydrogen-high --mode auto --print-timeout 1h
```

## Modelos

Usar la última versión de cada familia. Slug: `gemini-<VERSIÓN>-<familia>-<effort>` (el effort va en el slug); consultarlos con `agy models`. Si un slug falla por inválido, informarlo sin sustituirlo.

| Familia | Effort | Uso | Ejemplo |
|---|---|---|---|
| Hydrogen | `high` | Por defecto. | `--model gemini-4-hydrogen-high` |
| Neon | `medium` | Solo si el usuario quiere delegar trabajo crítico, pesado o que exige más criterio. | `--model gemini-4-neon-medium` |

## Permisos

Usar siempre `--mode auto`: aprueba ediciones y comandos seguros sin pedir confirmación. Agy puede mostrar `--mode auto is not available for this account` al iniciar, aunque el modo auto se aplica igual. No usar `--mode accept-edits`: deniega los comandos de terminal.

## Duración

Una tarea puede tardar mucho: lanzarla en segundo plano o con un timeout de 1 h, nunca con el timeout por defecto de la herramienta.

## Continuar una conversación

Agy solo expone el ID con `--output-format json` (campo `conversation_id`): agregarlo únicamente en la invocación que se vaya a retomar, y retomarla con:

```bash
agy -p "<PROMPT>" --model gemini-4-hydrogen-high --mode auto --conversation <CONVERSATION_ID>
```

`-c` (`--continue`) en lugar de `--conversation` retoma la última conversación. Preferir invocaciones independientes salvo que el contexto previo sea necesario.

## Cuota agotada

Consultar la cuota con `agy -p "/usage"` antes de la primera invocación y cada vez que una falle (código de salida distinto de cero, timeout o respuesta vacía o inútil).

- Si `/usage` confirma la cuota agotada: dejar de usar Gemini por el resto de la orquestación, sin reintentar ni volver a sondear, e informar al usuario.
- Si queda cuota: tratarlo como un error común (modelo, permisos, timeout, configuración o servicio), sin deshabilitar Gemini.

## Resultado

Transmitir al usuario la respuesta, los cambios y los errores. Verificar cambios y pruebas antes de presentarlos como exitosos.
