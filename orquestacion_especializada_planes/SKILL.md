---
name: orquestacion_especializada_planes
description: Skill para gestionar, delegar, supervisar y validar la ejecución de sub-agentes especializados durante la implementación técnica de un plan. Incluye uso de memoria persistente con Engram para indexar el plan por fases, registrar progreso en tiempo real, y proveer a cada sub-agente contexto completo y vigente sin depender de la ventana de contexto activa. Activar siempre que haya un plan de implementación técnico con múltiples fases o dominios, haya riesgo de pérdida de contexto entre agentes, se necesite orquestar sub-agentes especializados, o cuando el trabajo pueda compactarse o interrumpirse entre sesiones. También usar cuando el usuario mencione "orquestar agentes", "plan de fases", "delegación técnica", "sub-agentes", "memoria de plan", o cualquier variante de ejecución por etapas con múltiples especialistas.
---

# SKILL: Orquestación Especializada de Planes con Memoria Persistente

> Para referencias detalladas, leer:
> - `references/protocolo_delegacion.md` — protocolo completo de delegación, plantilla y reglas por dominio
> - `references/engram_ops.md` — operaciones Engram: cómo indexar, actualizar y consultar memoria de plan

---

## 1. Propósito

Esta skill define cómo un agente orquestador coordina sub-agentes especializados durante la ejecución de un **Plan de Implementación** usando **memoria persistente con Engram** como columna vertebral del estado compartido.

Objetivos:

- convertir fases del plan en tareas ejecutables con contexto autosuficiente;
- persistir el plan completo en Engram antes de comenzar, indexado por fases;
- proveer a cada sub-agente la clave Engram exacta desde donde recuperar su contexto;
- actualizar Engram al cierre de cada fase con estado real (completado, parcial, bloqueado);
- garantizar que ante compactación, nueva sesión o cambio de agente, el plan nunca se pierda;
- evitar que sub-agentes reconstruyan contexto por inferencia o planes ambiguos.

Esta skill **no reemplaza** al plan de implementación: lo persiste, lo distribuye y lo actualiza.

---

## 2. Cuándo se Activa

Condiciones de activación (todas deben cumplirse):

1. existe un Plan de Implementación explícito con fases definidas;
2. el trabajo requiere ejecución técnica en una o más capas del sistema;
3. hay riesgo de desalineación entre diseño, código e integración.

No activar para:

- brainstorming o diseño inicial sin fases definidas;
- tareas triviales de una sola capa sin delegación;
- cambios menores que no requieran sub-agentes ni supervisión multi-dominio.

---

## 3. Flujo Maestro del Orquestador

```
INICIO
  │
  ▼
[FASE 0] ── Indexar plan en Engram ──────────────────────────────────────────┐
  │           • mem_session_start                                             │
  │           • mem_save: meta, restricciones, una obs. por fase, estado     │
  │           • guardar IDs devueltos en contexto activo                     │
  │                                                                           │
  ▼                                                                           │
[FASE N] ── Preparar delegación ─────────────────────────────────────────────┤
  │           • mem_get_observation ID_fase_N  → verificar precondiciones    │
  │           • Construir contexto curado (extracto de la fase)              │
  │           • Determinar skills disponibles que aplican → sugerirlas       │
  │           • Incluir IDs Engram para consulta proactiva del sub-agente    │
  │                                                                           │
  ▼                                                                           │
[EJECUCIÓN] ── Sub-agente opera ────────────────────────────────────────────┤
  │           • Consulta Engram proactivamente al inicio (si tiene acceso)   │
  │           • Consulta skills sugeridas antes de implementar               │
  │           • Devuelve entregable + reporte de estado                      │
  │                                                                           │
  ▼                                                                           │
[VALIDACIÓN] ── Orquestador valida ─────────────────────────────────────────┤
  │           • Contrasta contra mem_get_observation ID_fase_N               │
  │           • Decisión: Aprobada / Observaciones / Rechazada / Bloqueada   │
  │                                                                           │
  ▼                                                                           │
[ACTUALIZAR ENGRAM] ── Registrar resultado real ─────────────────────────────┤
  │           • mem_save: estado de fase (completada/bloqueada/rechazada)    │
  │           • mem_save: estado global actualizado                          │
  │           • Registrar archivos tocados, desvíos, deuda técnica           │
  │                                                                           │
  ▼                                                                           │
¿Más fases? ── Sí → volver a [FASE N+1]                                      │
             └─ No → [CIERRE FINAL]                                          │
                      • mem_save: cierre final                               │
                      • mem_session_summary + mem_session_end                │
```

---

## 4. Fase 0: Indexación del Plan en Engram

**Esta fase es obligatoria antes de delegar cualquier sub-agente.**

Engram no es un key-value store: persiste **observaciones estructuradas** recuperables por ID o por búsqueda full-text. El orquestador crea una observación por cada elemento del plan que necesite persistencia.

### 4.1 Observaciones requeridas al inicio

| Título de observación | Tipo | Contenido |
|---|---|---|
| `[PLAN:{nombre}] Meta y objetivo general` | `"plan"` | Nombre, versión, objetivo, dominio, total de fases |
| `[PLAN:{nombre}] Restricciones globales` | `"architecture"` | Convenciones, contratos, separación de capas |
| `[PLAN:{nombre}] Fase {N}: {nombre}` | `"decision"` | Contenido operativo completo de la fase (una obs. por fase) |
| `[PLAN:{nombre}] Estado global` | `"plan"` | Mapa de progreso: todas las fases en `"pendiente"` |

### 4.2 Convención de títulos

Usar el prefijo `[PLAN:{nombre}]` en todos los títulos. Permite recuperar todas las memorias del plan con una sola búsqueda:

```
mem_search "[PLAN:suite_api_v2]"   → lista todas las observaciones del plan
```

### 4.3 Secuencia de inicio

```
1. mem_session_start
2. mem_context                        ← verificar si el plan ya existe de sesión anterior
3. mem_save para cada observación     ← guardar los IDs devueltos
4. Retener IDs en contexto activo     ← para acceso directo durante la sesión
```

Los IDs devueltos por `mem_save` permiten `mem_get_observation id={ID}` directo sin búsqueda. Guardarlos es crítico mientras el contexto activo esté disponible.

Ver plantillas completas de contenido en `references/engram_ops.md` → §4.

---

## 5. Preparación de la Delegación

Antes de invocar al sub-agente, el orquestador debe preparar tres elementos además del contexto de la fase: el acceso a Engram, las skills sugeridas, y el alcance.

### 5.1 Acceso a Engram para el sub-agente

Si el sub-agente tiene acceso a Engram, el orquestador debe **siempre** incluir los IDs relevantes e instruir al sub-agente a consultarlos **al inicio de su tarea**, no solo si tiene dudas.

La lógica es: Engram tiene el contenido completo y persistido de la fase; el mensaje de delegación puede ser un extracto. El sub-agente debería leer la observación completa antes de arrancar.

| Situación | Acción |
|---|---|
| Sub-agente tiene acceso a Engram | Incluir IDs y sugerir consulta proactiva al inicio |
| Sub-agente NO tiene acceso a Engram | Pasar todo el contexto necesario embebido en el mensaje |

**Instrucción estándar para sub-agente con acceso Engram:**

```
Antes de comenzar, consultá Engram para obtener el contexto completo de esta fase:
  mem_get_observation id={ID_fase_N}          ← contenido completo de la fase
  mem_get_observation id={ID_restricciones}   ← convenciones y contratos del proyecto

El mensaje que te pasé es un extracto. La observación en Engram es la fuente completa y vigente.
Si algo en Engram contradice este mensaje, reportá la discrepancia al orquestador antes de continuar.
NO hagas mem_search genérico ni accedas a otras observaciones del plan.
NO llames mem_save — el orquestador registra el cierre de fase.
```

### 5.2 Sugerencia de Skills al Sub-agente

El orquestador debe evaluar qué skills del sistema son aplicables a la fase y sugerirlas en la delegación. No es una obligación para el sub-agente, pero sí una guía para que no ejecute sin aprovechar herramientas especializadas disponibles.

**Cómo determinar qué skills sugerir:**

| Dominio / tarea de la fase | Skills típicamente relevantes |
|---|---|
| Backend Laravel (endpoints, modelos, migraciones) | skill de Laravel, skill de API design |
| Frontend (componentes, estado, formularios) | skill de frontend-design, skill del framework en uso |
| Base de datos (migraciones, esquemas, seeds) | skill de DB / migrations |
| Documentos Word / PDF / planillas | skill docx, skill pdf, skill xlsx |
| Pruebas / QA | skill de testing del stack en uso |
| Código genérico, scripts, CLI | skill de file-reading, skill del lenguaje |

El orquestador no inventa skills que no existen. Solo sugiere las que sabe disponibles en el entorno.

**Regla adicional para fases de Frontend — DESIGN.md:**

Antes de delegar cualquier fase de dominio Frontend, el orquestador debe verificar si existe un archivo `DESIGN.md` en el repositorio. Si existe, debe incluirlo como referencia obligatoria en la delegación para que el sub-agente respete el sistema visual de la aplicación.

```
Si el repositorio tiene DESIGN.md:

  Referencia de diseño obligatoria: DESIGN.md
  Antes de implementar cualquier componente, leé DESIGN.md para entender
  la paleta de colores, tipografía, espaciado, componentes base y patrones
  visuales del proyecto. No introduzcas estilos, tokens o componentes que
  contradigan lo definido allí. Si necesitás algo que DESIGN.md no cubre,
  reportalo al orquestador en lugar de improvisar.
```

Si `DESIGN.md` no existe en el repositorio, omitir esta instrucción.

**Instrucción estándar para incluir en la delegación:**

```
Skills sugeridas para esta fase:
  - [{nombre_skill}]: {razón concreta por la que aplica a esta tarea}
  - [{nombre_skill}]: {razón concreta}

No estás obligado a usarlas, pero consultarlas antes de implementar puede ahorrarte
trabajo y asegurar que el output esté alineado con las convenciones del proyecto.
```

Si no hay skills conocidas que apliquen a la fase, omitir esta sección en lugar de inventar.

### 5.3 Plantilla de delegación completa

Ver `references/protocolo_delegacion.md` → sección "Plantilla Obligatoria".

---

## 6. Actualización de Engram al Cierre de Fase

**Obligatorio después de cada validación de entregable.**

### 6.1 Qué guardar

```
mem_save  tipo:"bugfix"
título: "[PLAN:{nombre}] Estado Fase {N} - {COMPLETADA|BLOQUEADA|RECHAZADA}"
contenido:
  estado: completada | parcial | bloqueada | rechazada
  fecha_cierre: {timestamp}
  archivos_tocados: [lista real de archivos modificados]
  supuestos_usados: [supuestos documentados por el sub-agente]
  desvíos_detectados: [si los hubo]
  deuda_técnica: [observaciones para fases futuras]
  contexto_fue_suficiente: sí | no | parcial
  notas_para_siguiente_agente: [qué debe saber quien continúe]

mem_save  tipo:"plan"
título: "[PLAN:{nombre}] Estado global"
contenido: mapa actualizado con nuevo estado de fase N
```

### 6.2 Cuándo actualizar

- Inmediatamente al aprobar o rechazar un entregable.
- Antes de invocar el siguiente sub-agente.
- Si la sesión puede compactarse, actualizar Engram antes de perder contexto — y llamar `mem_session_summary`.

### 6.3 Registro de bloqueos

```
mem_save  tipo:"bugfix"
título: "[PLAN:{nombre}] Estado Fase {N} - BLOQUEADA"
contenido:
  estado: bloqueada
  motivo_bloqueo: {descripción exacta}
  requiere: {qué se necesita para desbloquear}
  escalado_a: orquestador | redefinición de plan
```

---

## 7. Recuperación de Estado en Nueva Sesión o Compactación

Cuando el orquestador inicia en un contexto nuevo (compactación, nueva sesión, agente diferente):

### 7.1 Secuencia de recuperación

```
1. mem_context
   → inyecta contexto de sesión anterior automáticamente

2. mem_search "[PLAN:{nombre}] Estado global"
   → encontrar la observación de estado más reciente → obtener su ID

3. mem_get_observation id={ID_estado_global}
   → leer qué fases están pendientes, en curso o bloqueadas

4. Si hay fases "en_curso":
   mem_search "[PLAN:{nombre}] Fase {N}"
   mem_get_observation id={ID_fase_N}
   → determinar si se completó o quedó a medias; decidir si continuar o re-delegar

5. mem_search "[PLAN:{nombre}] Restricciones"
   mem_get_observation id={ID_restricciones}
   → cargar convenciones globales
```

### 7.2 Regla: no asumir, verificar

El orquestador nunca asume que una fase está completa porque recuerda haberla delegado. Debe leer el estado real desde Engram antes de continuar.

---

## 8. Principios de Orquestación

Ver detalle completo en `references/protocolo_delegacion.md`. Resumen operativo:

- **Separación estricta de dominios**: backend, frontend, datos, infra, QA nunca se mezclan en una misma delegación sin justificación explícita.
- **Contexto suficiente, no mínimo**: el sub-agente no debe reconstruir contexto; debe poder ejecutar con lo que recibe.
- **Contexto vigente**: siempre del plan activo, nunca de versiones previas o inferencias.
- **Prohibición de búsqueda autónoma**: el sub-agente no busca contexto externo sin autorización.
- **Fidelidad al plan**: ningún sub-agente rediseña; si detecta inconsistencia, la reporta.
- **Validación incremental**: ninguna fase avanza sin aprobación explícita del orquestador.

---

## 9. Supervisión y Validación

Después de cada entrega, el orquestador ejecuta:

1. **Verificación contra Engram** — `mem_get_observation id={ID_fase_N}` → ¿se implementó lo que decía la observación?
2. **Verificación técnica local** — coherencia lógica, consistencia con código existente.
3. **Verificación inter-dominios** — contratos compartidos entre capas.
4. **Verificación de suficiencia de contexto** — ¿el sub-agente tuvo que inferir algo?

**Decisión obligatoria:**

| Estado | Acción |
|---|---|
| ✅ Aprobada | Actualizar Engram → avanzar |
| ⚠️ Aprobada con observaciones | Actualizar Engram con deuda → avanzar |
| ❌ Rechazada | Corregir antes de actualizar Engram |
| 🔒 Bloqueada | Registrar bloqueo en Engram → escalar |

---

## 10. Reglas de Control Obligatorias

- Indexar el plan completo en Engram con `mem_save` **antes** de delegar la primera fase.
- Llamar `mem_session_start` al inicio y `mem_session_summary` + `mem_session_end` al cerrar.
- Guardar los IDs devueltos por `mem_save` durante la sesión activa.
- Actualizar el estado global en Engram **inmediatamente** tras aprobar o rechazar cada entregable.
- No delegar más de una fase activa simultáneamente al mismo sub-agente.
- No delegar una fase solo por identificador: transferir su contenido operativo.
- No avanzar si existe divergencia contractual no resuelta.
- No permitir que sub-agentes usen `mem_search` libre ni `mem_save` sin autorización explícita.
- No mezclar diagnóstico, implementación y validación sin declarar qué rol se ejecuta.
- Ante compactación inminente: llamar `mem_session_summary` antes de perder contexto.

---

## 11. Cierre de Implementación

Al terminar todas las fases:

```
mem_save  tipo:"plan"
título: "[PLAN:{nombre}] Cierre final"
contenido:
  fases_ejecutadas, sub_agentes, archivos_afectados,
  desvíos_corregidos, deuda_técnica, riesgos, fallas_de_contexto

mem_session_summary
  Goal / Discoveries / Accomplished / Files

mem_session_end
```

---

## 12. Regla Central

**Cada sub-agente debe recibir: el contenido operativo de la fase, su objetivo dentro del plan, el objetivo local, las precondiciones resueltas, los archivos afectados, las restricciones, los IDs Engram para consulta proactiva al inicio, y las skills disponibles que aplican a su tarea. Engram es la fuente de verdad del plan. Las skills son las herramientas especializadas que el sub-agente debe aprovechar. El orquestador no obliga, pero sí informa.**

---

## 13. Anti-Patrones a Evitar

- No indexar el plan en Engram antes de comenzar.
- No llamar `mem_session_summary` al cerrar la sesión.
- No guardar los IDs devueltos por `mem_save` durante la sesión.
- Actualizar Engram solo al final (demasiado tarde si hay compactación).
- Delegar fases solo por nombre/número sin contenido operativo.
- Incluir IDs Engram pero no indicar que el sub-agente los consulte proactivamente al inicio.
- No sugerir skills disponibles al sub-agente cuando hay skills aplicables a la fase.
- Inventar o sugerir skills que no existen en el entorno.
- Permitir que sub-agentes usen `mem_search` libre sin query específica autorizada.
- Permitir que sub-agentes llamen `mem_save` sin autorización explícita.
- Asumir que el estado en memoria activa es igual al estado real en Engram.
- No registrar bloqueos, desvíos ni deuda técnica en Engram.
- Encadenar fases sin actualizar Engram entre una y otra.

---

## 14. Versión Breve para Activación Rápida

1. **Indexar** el plan completo en Engram (`mem_save` por fase + estado + restricciones).
2. **Recuperar** la fase con `mem_get_observation id={ID_fase_N}` antes de cada delegación.
3. **Evaluar** qué skills disponibles aplican a la fase → incluirlas como sugerencia.
4. **Delegar** con: contexto curado + IDs Engram para consulta proactiva al inicio + skills sugeridas.
5. **Validar** el entregable contra la observación de la fase.
6. **Actualizar** Engram con el estado real (`mem_save` + estado global) antes de continuar.
7. **Recuperar** estado con `mem_context` + `mem_search "[PLAN:{nombre}]"` ante compactación o nueva sesión.

El orquestador no delega responsabilidad: delega ejecución. Engram no reemplaza el contexto: lo persiste. Las skills no son obligatorias para el sub-agente: son orientación del orquestador.
