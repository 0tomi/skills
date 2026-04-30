# Operaciones Engram para Orquestación

> Engram es un sistema de memoria persistente con SQLite + FTS5.
> No es un key-value store. Las memorias se guardan como **observaciones estructuradas**
> y se recuperan por **búsqueda full-text** o por **ID de observación**.

## Tabla de Contenidos

0. [Regla crítica: escrituras secuenciales, nunca en paralelo](#serializar)
1. [Los 10 tools MCP disponibles](#tools)
2. [Cómo mapear el plan a observaciones Engram](#mapeo)
3. [Operaciones en cada etapa del ciclo de orquestación](#operaciones)
4. [Plantillas de contenido por tipo de observación](#plantillas)
5. [Patrón de recuperación en 3 capas](#recovery)
6. [Recuperación ante compactación o nueva sesión](#compactacion)
7. [Reglas de uso para sub-agentes](#subagentes)
8. [Errores comunes y cómo evitarlos](#errores)

---

## 0. Regla Crítica: Escrituras Secuenciales, Nunca en Paralelo {#serializar}

**Engram usa SQLite. SQLite no soporta escrituras concurrentes. Disparar varios `mem_save` en paralelo rompe la base de datos o devuelve errores de bloqueo.**

### Reglas obligatorias

1. **Una escritura a la vez.** Nunca disparar dos `mem_save` en paralelo (ni con `Promise.all`, ni encadenando tool calls simultáneos en un mismo turno, ni de cualquier otra manera).
2. **Esperar respuesta antes de la siguiente.** Cada `mem_save` debe completar y devolver su ID antes de iniciar el siguiente. Si una llamada falla, reintentar esa antes de continuar.
3. **Las lecturas (`mem_get_observation`, `mem_search`, `mem_context`, `mem_timeline`) sí pueden hacerse en paralelo entre sí**, pero NO en paralelo con escrituras.
4. **Mientras hay un `mem_save` en curso, no disparar otra operación de escritura** (`mem_save`, `mem_session_summary`, `mem_save_prompt`).

### Patrones correctos vs incorrectos

```
INCORRECTO — disparar varias escrituras en el mismo turno:
  mem_save(estado_fase_3)
  mem_save(estado_global_actualizado)
  → Engram puede fallar por lock. La segunda llamada puede no escribir.

CORRECTO — secuencial, una termina antes que arranque la siguiente:
  resultado_1 = mem_save(estado_fase_3)        ← esperar a que termine
  resultado_2 = mem_save(estado_global)        ← recién después, la siguiente

INCORRECTO — indexación inicial en paralelo:
  En un solo turno, disparar simultáneamente:
    mem_save(meta), mem_save(restricciones), mem_save(fase_1), mem_save(fase_2)...
  → SQLite lock garantizado.

CORRECTO — indexación inicial secuencial:
  Turno 1: mem_save(meta)                  → esperar respuesta, guardar ID
  Turno 2: mem_save(restricciones)         → esperar respuesta, guardar ID
  Turno 3: mem_save(fase_1)                → esperar respuesta, guardar ID
  ...
```

### Consecuencia para el cierre de fase

Cuando cierra una fase, el orquestador escribe DOS observaciones (estado de fase + estado global). DEBEN ir secuenciales:

```
1. mem_save(estado_fase_N)        → esperar respuesta
2. mem_save(estado_global)        → recién ahora
```

No usar `Promise.all` ni equivalentes. No disparar la segunda llamada antes de que vuelva la primera.

### Consecuencia para la auditoría final

La auditoría escribe múltiples observaciones (auditoría + cierre + session_summary). Todas secuenciales:

```
1. mem_save(auditoría_final)      → esperar
2. mem_save(cierre_final)         → esperar
3. mem_session_summary            → esperar
4. mem_session_end                → esperar
```

### Si el orquestador delega fases en paralelo

Aunque varias fases puedan ejecutarse en paralelo a nivel de sub-agentes, sus **escrituras de cierre a Engram deben serializarse en el orquestador**. Cuando dos sub-agentes terminen casi simultáneamente, el orquestador procesa los cierres de a uno: valida el primero, escribe a Engram (esperando respuesta), valida el segundo, escribe a Engram. Nunca dispara las dos escrituras en paralelo.

---

## 1. Los 10 Tools MCP Disponibles {#tools}

| Tool | Cuándo usar en orquestación |
|---|---|
| `mem_save` | Persistir el plan, fases, estados de cierre, decisiones clave |
| `mem_search` | Recuperar observaciones por contenido (query full-text) |
| `mem_get_observation` | Obtener el contenido completo de una observación por ID |
| `mem_timeline` | Ver contexto cronológico alrededor de una observación |
| `mem_context` | Cargar contexto de sesiones anteriores al iniciar |
| `mem_session_summary` | Guardar resumen de sesión al cerrar (obligatorio) |
| `mem_session_start` | Registrar inicio de sesión de orquestación |
| `mem_session_end` | Marcar sesión de orquestación como completada |
| `mem_save_prompt` | Guardar el prompt/instrucción del plan original si es muy largo |
| `mem_stats` | Verificar estado del sistema de memoria |

---

## 2. Cómo Mapear el Plan a Observaciones Engram {#mapeo}

Engram almacena **observaciones** con título, tipo y contenido libre.
El orquestador crea una observación por cada elemento del plan que necesite persistencia.

### Convención de tipos recomendada

| Tipo de observación | Contenido |
|---|---|
| `"plan"` | Metadatos del plan y estado global de progreso |
| `"architecture"` | Restricciones globales, convenciones, contratos inamovibles |
| `"decision"` | Contenido operativo de cada fase (una observación por fase) |
| `"bugfix"` | Estado de cierre de una fase (qué se hizo, qué no, deuda) |
| `"session_summary"` | Resumen de sesión (generado por `mem_session_summary`) |

### Convención de títulos para búsqueda predecible

Usar el prefijo `[PLAN:{nombre}]` en todos los títulos. Permite filtrar todas las memorias de un plan con una sola búsqueda:

```
mem_search "[PLAN:suite_api_v2]"   → devuelve todo lo del plan
```

Títulos recomendados:

```
"[PLAN:{nombre}] Meta y objetivo general"
"[PLAN:{nombre}] Restricciones globales del proyecto"
"[PLAN:{nombre}] Estado global"
"[PLAN:{nombre}] Fase 1: {nombre de la fase}"
"[PLAN:{nombre}] Fase 2: {nombre de la fase}"
"[PLAN:{nombre}] Estado Fase 1 - COMPLETADA"
"[PLAN:{nombre}] Estado Fase 2 - BLOQUEADA"
"[PLAN:{nombre}] Cierre final"
```

---

## 3. Operaciones en Cada Etapa del Ciclo {#operaciones}

### FASE 0 — Inicio de sesión + Indexación del plan

```
1. mem_session_start
   → registrar sesión de orquestación

2. mem_context
   → verificar si hay contexto de sesiones anteriores
   → si ya existe el plan indexado, no re-crear; recuperar IDs via mem_search

3. mem_save  tipo:"plan"
   título: "[PLAN:{nombre}] Meta y objetivo general"
   → retorna observation_id → guardarlo

4. mem_save  tipo:"architecture"
   título: "[PLAN:{nombre}] Restricciones globales del proyecto"
   → retorna observation_id → guardarlo

5. PARA CADA FASE N:
   mem_save  tipo:"decision"
   título: "[PLAN:{nombre}] Fase {N}: {nombre}"
   → retorna observation_id → guardarlo como ID_fase_N

6. mem_save  tipo:"plan"
   título: "[PLAN:{nombre}] Estado global"
   contenido: mapa de todas las fases en estado "pendiente"
   → retorna observation_id → guardarlo como ID_estado_global

IMPORTANTE: guardar los IDs devueltos durante la sesión activa.
Si se pierde el contexto, recuperarlos con mem_search "[PLAN:{nombre}]".
```

### ANTES DE CADA DELEGACIÓN

```
1. mem_get_observation id={ID_estado_global}
   → leer estado actual para verificar que precondiciones están completadas

2. mem_get_observation id={ID_fase_N}
   → obtener contenido completo y vigente de la fase a delegar

3. mem_save  tipo:"plan"
   título: "[PLAN:{nombre}] Fase {N}: {nombre} - EN CURSO"
   contenido: timestamp de inicio, sub-agente asignado
```

### DESPUÉS DE VALIDAR ENTREGABLE

```
1. mem_save  tipo:"bugfix"
   título: "[PLAN:{nombre}] Estado Fase {N} - {COMPLETADA|BLOQUEADA|RECHAZADA}"
   contenido: resultado real (ver plantilla §4)

2. mem_get_observation id={ID_estado_global}
   → leer estado actual completo

3. mem_save  tipo:"plan"
   título: "[PLAN:{nombre}] Estado global"
   contenido: mapa actualizado con nuevo estado de fase N
   → guardar nuevo ID_estado_global (el anterior queda en historial)
```

### AL FINALIZAR TODAS LAS FASES

```
1. mem_save  tipo:"plan"
   título: "[PLAN:{nombre}] Cierre final"
   contenido: resumen completo (ver plantilla §4)

2. mem_session_summary
   formato obligatorio:
     Goal: implementar {plan}
     Discoveries: {hallazgos durante la ejecución}
     Accomplished: {fases completadas, archivos afectados}
     Files: {lista consolidada de archivos modificados}

3. mem_session_end
   → marcar sesión como completada
```

### ANTE COMPACTACIÓN INMINENTE

```
→ Antes de que el contexto se comprima, ejecutar:

1. mem_save  tipo:"bugfix"
   título: "[PLAN:{nombre}] Checkpoint pre-compactación"
   contenido: estado real actual, fase activa, qué se hizo, qué falta

2. mem_session_summary
   → guardar estado de sesión hasta este momento
   → la próxima sesión arranca con este contexto via mem_context
```

---

## 4. Plantillas de Contenido por Tipo de Observación {#plantillas}

### `[PLAN:{nombre}] Meta y objetivo general`

```
PLAN: {nombre completo}
VERSION: {v1.0 | fecha}
OBJETIVO: {qué resuelve este plan}
DOMINIO TÉCNICO: {Laravel / Qt / React / etc}
FECHA INICIO: {timestamp}
TOTAL FASES: {N}
FASES: [1, 2, 3, ..., N]
ORQUESTADOR: {agente o sesión que inició}
```

### `[PLAN:{nombre}] Restricciones globales del proyecto`

```
CONVENCIONES DE CÓDIGO:
  - {convención 1}
  - {convención 2}

CONTRATOS INAMOVIBLES:
  - {contrato 1: descripción}

SUPUESTOS ARQUITECTÓNICOS:
  - {supuesto 1}

LIBRERÍAS PROHIBIDAS:
  - {librería y razón}

SEPARACIÓN DE CAPAS:
  - backend no toca frontend
  - {otras separaciones}

ARCHIVOS CRÍTICOS (no tocar sin autorización):
  - {ruta/archivo}
```

### `[PLAN:{nombre}] Fase {N}: {nombre}`

```
FASE: {N}
NOMBRE: {nombre de la fase}
DOMINIO: backend | frontend | datos | infra | qa

OBJETIVO DE FASE: {qué habilita dentro del plan general}
OBJETIVO TÉCNICO: {qué se implementa concretamente}

PRECONDICIONES:
  - Fase {M} completada
  - {otra precondición}

CAMBIOS CONTEMPLADOS:
  - {cambio 1}
  - {cambio 2}

ARCHIVOS PERMITIDOS:
  - {ruta/archivo_1}
  - {ruta/archivo_2}

ARCHIVOS PROHIBIDOS:
  - {ruta/archivo_x} — razón

RESTRICCIONES DE FASE:
  - {restricción específica}

CRITERIO DE FINALIZACIÓN: {condición verificable}

RESULTADO PARA FASE SIGUIENTE: {qué habilita}
```

### `[PLAN:{nombre}] Estado global`

```
ÚLTIMA ACTUALIZACIÓN: {timestamp}
FASE ACTIVA: {N | ninguna}

fase_1: pendiente | en_curso | completada | parcial | bloqueada | rechazada
fase_2: pendiente | en_curso | completada | parcial | bloqueada | rechazada
...
```

### `[PLAN:{nombre}] Estado Fase {N} - {ESTADO}`

```
FASE: {N}
ESTADO: COMPLETADA | PARCIAL | BLOQUEADA | RECHAZADA
FECHA CIERRE: {timestamp}

ARCHIVOS TOCADOS:
  - {ruta/archivo_1} — {descripción del cambio}

SUPUESTOS USADOS:
  - {supuesto que el sub-agente documentó}

DESVIACIONES DETECTADAS:
  - ninguna | {descripción}

DEUDA TÉCNICA:
  - ninguna | {observación para fases futuras}

CONTEXTO FUE SUFICIENTE: sí | no | parcial — {detalle}

NOTAS PARA SIGUIENTE AGENTE:
  - {qué debe saber quien continúe}

MOTIVO DE BLOQUEO: {solo si BLOQUEADA}
REQUIERE PARA DESBLOQUEAR: {solo si BLOQUEADA}
```

### `[PLAN:{nombre}] Cierre final`

```
ESTADO: COMPLETADO
FECHA CIERRE: {timestamp}

FASES EJECUTADAS: [1, 2, 3]
FASES PARCIALES: [4] — razón

SUB-AGENTES INVOLUCRADOS:
  - backend: fases 1, 3
  - frontend: fase 2

ARCHIVOS AFECTADOS (consolidado):
  - {ruta/archivo_1}

DESVIACIONES CORREGIDAS: {descripción | ninguna}
DEUDA TÉCNICA REMANENTE: {descripción | ninguna}
RIESGOS DETECTADOS: {descripción | ninguno}
FALLAS DE CONTEXTO: {descripción | ninguna}
```

---

## 5. Patrón de Recuperación en 3 Capas (Progressive Disclosure) {#recovery}

Engram está diseñado para recuperación eficiente en tokens. Nunca volcar todo: buscar, identificar, luego obtener detalle.

```
CAPA 1 — Búsqueda compacta (~100 tokens por resultado)
  mem_search "[PLAN:{nombre}]"
  → lista de observaciones con IDs

CAPA 2 — Contexto cronológico
  mem_timeline observation_id={ID}
  → qué pasó antes y después de esa observación en la sesión

CAPA 3 — Contenido completo
  mem_get_observation id={ID}
  → contenido sin truncar de la observación específica
```

**Regla**: nunca hacer `mem_get_observation` sobre todos los IDs a ciegas. Buscar primero, identificar el relevante, luego obtener el detalle.

---

## 6. Recuperación ante Compactación o Nueva Sesión {#compactacion}

```
1. mem_context
   → inyecta contexto de sesión anterior automáticamente

2. mem_search "[PLAN:{nombre}] Estado global"
   → encontrar la observación de estado más reciente

3. mem_get_observation id={ID_estado_global}
   → leer qué fases están pendientes, en curso, bloqueadas

4. Si hay fases "en_curso":
   mem_search "[PLAN:{nombre}] Fase {N}"
   mem_get_observation id={ID_fase_N}
   → determinar si se completó o quedó a medias

5. mem_search "[PLAN:{nombre}] Restricciones"
   mem_get_observation id={ID_restricciones}
   → cargar convenciones globales

6. Continuar desde la primera fase no completada
```

**Regla crítica**: el estado real vive en Engram, no en la memoria activa. Nunca asumir que una fase está completa sin leerlo desde Engram.

### Instrucción para CLAUDE.md / system prompt del agente

```
## Memoria con Engram
Tenés acceso a memoria persistente via MCP (mem_save, mem_search, mem_context, etc.).
- Guardá el plan completo en Engram antes de delegar la primera fase.
- Actualizá el estado en Engram inmediatamente tras aprobar o rechazar cada entregable.
- Después de cualquier compactación o reset de contexto, llamá mem_context y luego
  mem_search "[PLAN:{nombre}]" para recuperar el estado del plan antes de continuar.
- Antes de cerrar la sesión, siempre llamá mem_session_summary.
```

---

## 7. Reglas de Uso para Sub-agentes {#subagentes}

| Operación | Orquestador | Sub-agente |
|---|---|---|
| `mem_save` | ✅ Sí | ❌ Solo si el orquestador lo autoriza explícitamente |
| `mem_search` | ✅ Sí | ✅ Solo con query específica autorizada |
| `mem_get_observation` | ✅ Sí | ✅ Solo con ID específico provisto por el orquestador |
| `mem_session_summary` | ✅ Sí | ❌ No |
| `mem_context` | ✅ Sí | ❌ No (contexto global, no pertinente) |

### Cómo incluir acceso Engram en una delegación

```
## Acceso a Engram autorizado para esta fase

Si necesitás contexto adicional, podés consultar:
  mem_get_observation id={ID_fase_N}        ← contenido completo de esta fase
  mem_get_observation id={ID_restricciones} ← convenciones del proyecto

NO hagas mem_search genérico.
NO accedas a otras observaciones del plan.
NO guardes nada en Engram — el orquestador registra el cierre.
Si encontrás algo en Engram que contradice lo que te pasé en este mensaje,
reportá la discrepancia; no resolvás por tu cuenta.
```

---

## 8. Errores Comunes y Cómo Evitarlos {#errores}

| Error | Consecuencia | Prevención |
|---|---|---|
| No llamar `mem_session_start` | Sesión no registrada, `mem_context` no la recupera | Primer llamado siempre |
| No llamar `mem_session_summary` al cerrar | Próxima sesión empieza ciega | Siempre antes de terminar |
| No guardar IDs tras `mem_save` | Imposible hacer `mem_get_observation` directo | Registrar IDs devueltos |
| `mem_search` sin prefijo `[PLAN:{nombre}]` | Resultados de otros proyectos contaminan | Siempre incluir el prefijo |
| Sub-agente hace `mem_search` libre | Lee contexto de otras fases o proyectos | Proveer solo IDs específicos |
| Actualizar estado solo al final | Si hay compactación, estado queda desactualizado | Actualizar tras cada fase |
| No llamar `mem_context` al inicio | Empezar desde cero aunque haya contexto guardado | Primer llamado tras compactación |
| Sub-agente llama `mem_save` sin autorización | Ruido o corrupción en las memorias del plan | Prohibición explícita en la delegación |
