---
name: orquestacion_especializada_planes
description: Skill para orquestar, supervisar y validar la ejecución de un plan técnico por fases con sub-agentes especializados. El orquestador no solo distribuye tareas: vigila activamente que cada fase se implemente correctamente, valida fases críticas inmediatamente y fases estándar en lotes, y al final del plan audita el conjunto contra el objetivo original para detectar desvíos. Usa Engram como memoria persistente del plan (indexado por fases, actualizado en cada cierre, recuperable ante compactación). Activar cuando el usuario tiene un plan técnico con múltiples fases o dominios, menciona "orquestar agentes", "plan de fases", "delegación técnica", "sub-agentes", "memoria de plan", o cuando hay riesgo de pérdida de contexto entre agentes o sesiones.
---

# SKILL: Orquestación Especializada de Planes con Memoria Persistente

> Referencias modulares (cargar según necesidad):
> - `references/protocolo_delegacion.md` — plantilla de delegación, dominios, formato de respuesta
> - `references/engram_ops.md` — API completa de Engram, recuperación, plantillas de observaciones
> - `references/supervision_y_validacion.md` — criticidad, modos de validación, re-delegación, auditoría final, dry-run
> - `references/subagentes_auxiliares.md` — sdd-explore y sdd-archive: cuándo y cómo sugerirlos

---

## 1. Rol del Orquestador

El orquestador tiene **dos responsabilidades igualmente importantes**:

1. **Distribuir y delegar** — convertir las fases del plan en tareas ejecutables con contexto autosuficiente para cada sub-agente especializado.

2. **Vigilar y validar** — verificar que cada fase se implementó correctamente y que el conjunto sigue siendo coherente con el plan original. La vigilancia es activa, no pasiva.

La validación se ejecuta en dos momentos:

- **Tras cada entregable**, con frecuencia ajustada según criticidad de la fase: las fases críticas se validan inmediatamente; las estándar pueden agruparse de a 2 antes de validar (ver `supervision_y_validacion.md` §3).
- **Al cierre del plan**, mediante una auditoría que verifica cobertura del objetivo, consistencia inter-fases, desvíos del alcance original y deuda técnica acumulada (ver `supervision_y_validacion.md` §6).

Engram es la columna vertebral del estado: el plan se persiste indexado por fases antes de delegar, se actualiza en cada cierre, y se recupera ante compactación o nueva sesión.

---

## 2. Cuándo se Activa

Condiciones de activación (todas deben cumplirse):

1. existe un plan de implementación explícito con fases definidas;
2. el trabajo requiere ejecución técnica en una o más capas del sistema;
3. hay riesgo de desalineación entre diseño, código e integración.

No activar para:

- brainstorming o diseño inicial sin fases definidas;
- tareas triviales de una sola capa sin delegación;
- cambios menores que no requieran sub-agentes ni supervisión multi-dominio.

---

## 3. Flujo Maestro

```
INICIO
  │
  ▼
[FASE 0] ── Indexar plan en Engram + clasificar criticidad ──────────────────┐
  │           • mem_session_start                                             │
  │           • mem_save: meta, restricciones, una obs. por fase, estado     │
  │           • Cada fase declara CRITICIDAD: critica | estandar             │
  │           • (Opcional) Modo dry-run: presentar preview al humano         │
  │                                                                           │
  ▼                                                                           │
[FASE N] ── Preparar delegación ─────────────────────────────────────────────┤
  │           • mem_get_observation ID_fase_N → verificar precondiciones     │
  │           • Construir contexto curado (extracto de la fase)              │
  │           • Determinar skills disponibles que aplican → sugerirlas       │
  │           • Determinar si conviene ofrecer sdd-explore / sdd-archive     │
  │           • Incluir IDs Engram para consulta proactiva del sub-agente    │
  │           • Declarar criticidad y modo de validación esperado            │
  │                                                                           │
  ▼                                                                           │
[EJECUCIÓN] ── Sub-agente opera ────────────────────────────────────────────┤
  │           • Consulta Engram proactivamente al inicio (si tiene acceso)   │
  │           • Consulta skills sugeridas antes de implementar               │
  │           • Spawnea sdd-explore / sdd-archive si los necesita            │
  │           • Devuelve entregable + reporte de estado                      │
  │                                                                           │
  ▼                                                                           │
[VALIDACIÓN] ── Vigilancia activa según criticidad ─────────────────────────┤
  │           • Crítica: validar inmediatamente, no avanzar                  │
  │           • Estándar: acumular hasta 2 entregables, luego validar        │
  │           • Decisión: Aprobada / Observaciones / Rechazada / Bloqueada   │
  │           • Si rechazada: re-delegar con feedback estructurado           │
  │                                                                           │
  ▼                                                                           │
[ACTUALIZAR ENGRAM] ── Registrar resultado real (escrituras SECUENCIALES) ──┤
  │           • mem_save: estado de fase  → esperar respuesta                │
  │           • mem_save: estado global   → recién después                   │
  │           • Nunca disparar las dos en paralelo (SQLite no lo soporta)    │
  │           • Registrar archivos tocados, desvíos, deuda técnica           │
  │                                                                           │
  ▼                                                                           │
¿Más fases? ── Sí → volver a [FASE N+1]                                      │
             └─ No → [AUDITORÍA FINAL]                                       │
                      • Verificar cobertura del objetivo del plan            │
                      • Verificar consistencia inter-fases                   │
                      • Detectar desvíos del alcance original                │
                      • Consolidar deuda técnica                             │
                      • mem_save: cierre final                               │
                      • mem_session_summary + mem_session_end                │
```

---

## 4. Fase 0: Indexación del Plan en Engram

**Obligatoria antes de delegar cualquier sub-agente.**

Engram persiste **observaciones estructuradas** recuperables por ID o por búsqueda full-text. El orquestador crea una observación por cada elemento del plan que necesite persistencia.

### 4.1 Observaciones requeridas

| Título de observación | Tipo | Contenido |
|---|---|---|
| `[PLAN:{nombre}] Meta y objetivo general` | `"plan"` | Nombre, versión, objetivo, dominio, total de fases |
| `[PLAN:{nombre}] Restricciones globales` | `"architecture"` | Convenciones, contratos, separación de capas |
| `[PLAN:{nombre}] Fase {N}: {nombre}` | `"decision"` | Contenido operativo + criticidad declarada |
| `[PLAN:{nombre}] Estado global` | `"plan"` | Mapa de progreso: todas las fases en `"pendiente"` |

Convención de títulos: prefijo `[PLAN:{nombre}]` siempre, para permitir `mem_search "[PLAN:{nombre}]"` y filtrar todo el plan.

### 4.2 Clasificar criticidad de cada fase

Cada `[PLAN:{nombre}] Fase {N}` debe incluir:

```
CRITICIDAD: critica | estandar
RAZÓN DE CRITICIDAD: {por qué — solo si crítica}
```

Criterios completos en `supervision_y_validacion.md` §2. Regla rápida: ante la duda, marcar como crítica.

### 4.3 Subdivisión de fases multi-dominio

Si una fase del plan original mezcla dominios (ej: backend + frontend), el orquestador la divide en sub-observaciones referenciando el padre:

```
[PLAN:{nombre}] Fase 2.a: {parte backend}     ← referencia a Fase 2 padre
[PLAN:{nombre}] Fase 2.b: {parte frontend}    ← referencia a Fase 2 padre
```

Cada sub-fase tiene su propia criticidad, IDs y validación.

### 4.4 Versionado si el plan cambia a mitad de ejecución

Si el humano edita el plan original durante la ejecución:

1. Crear nueva observación `[PLAN:{nombre}] Meta y objetivo general (v2)`.
2. Marcar la observación anterior como obsoleta en su contenido (no eliminar — Engram conserva historial).
3. Crear las observaciones de fase modificadas con sufijo `(v2)`.
4. Actualizar `[PLAN:{nombre}] Estado global` referenciando los nuevos IDs.

### 4.5 Secuencia de inicio

> **Las escrituras de indexación van SECUENCIALES, no en paralelo.** SQLite no acepta concurrencia. Cada `mem_save` debe terminar y devolver su ID antes de iniciar el siguiente. Ver `engram_ops.md` §0.

```
1. mem_session_start                  → esperar respuesta
2. mem_context                        → verificar si el plan ya existe de sesión anterior
3. mem_save(meta)                     → esperar respuesta, guardar ID
4. mem_save(restricciones)            → esperar respuesta, guardar ID
5. mem_save(fase_1)                   → esperar respuesta, guardar ID
6. mem_save(fase_2)                   → esperar respuesta, guardar ID
   ... (una a una, nunca en paralelo)
N. mem_save(estado_global)            → esperar respuesta, guardar ID
N+1. (Opcional) Modo dry-run          → ver supervision_y_validacion.md §7
N+2. Retener IDs en contexto activo
```

Plantillas completas y ejemplos en `engram_ops.md` §4.

---

## 5. Preparación de la Delegación

Antes de invocar al sub-agente, el orquestador prepara cinco elementos:

1. **Contexto curado** — extracto de la fase, suficiente para ejecutar sin reconstruir nada.
2. **Acceso a Engram** — IDs de fase y restricciones, con instrucción de consulta proactiva.
3. **Skills sugeridas** — qué skills disponibles aplican a la tarea.
4. **Sub-agentes auxiliares disponibles** — sdd-explore / sdd-archive cuando la fase los amerite.
5. **Criticidad y modo de validación esperado** — para que el sub-agente sepa qué nivel de rigor se va a aplicar al revisarlo.

Plantilla completa de delegación en `protocolo_delegacion.md` §1.

### 5.1 Acceso a Engram para el sub-agente

Si el sub-agente tiene acceso a Engram, instruirlo a consultar **al inicio de la tarea**, no solo si tiene dudas.

> **CRÍTICO — sustitución de IDs**: los `{ID_fase_N}` y `{ID_restricciones}` que aparecen en la plantilla son placeholders. El orquestador DEBE reemplazarlos por los IDs reales que `mem_save` devolvió en la Fase 0 antes de enviar la delegación. Si llegan sin sustituir (literalmente con las llaves), el sub-agente no puede consultar nada y falla en silencio. Antes de enviar: verificar que el bloque tenga IDs concretos (ej. `id=4827`), no placeholders.

Bloque a incluir en la delegación, con los IDs ya sustituidos por valores reales:

```
Antes de comenzar, consultá Engram para obtener el contexto completo de esta fase:
  mem_get_observation id=<ID_REAL_DE_LA_FASE>          ← contenido completo de la fase
  mem_get_observation id=<ID_REAL_DE_RESTRICCIONES>    ← convenciones y contratos del proyecto

Este mensaje es un extracto. La observación en Engram es la fuente completa y vigente.
Si algo en Engram contradice este mensaje, reportá la discrepancia antes de continuar.
NO hagas mem_search genérico ni accedas a otras observaciones del plan.
NO llames mem_save — el orquestador registra el cierre de fase.
```

Si el sub-agente no tiene acceso a Engram, el orquestador pasa todo el contexto inline. Si en ese caso el sub-agente reporta insuficiencia, el orquestador re-delega ampliando el contexto inline; nunca le exige consultar Engram.

### 5.2 Sugerencia de Skills

El orquestador evalúa qué skills del entorno aplican a la fase y las sugiere con razón concreta. No inventa skills que no existen. Si no hay skills aplicables, omite la sección.

| Dominio / tarea | Skills típicamente relevantes |
|---|---|
| Backend Laravel (endpoints, modelos) | skill de Laravel, skill de API design |
| Frontend (componentes, estado) | skill de frontend-design, skill del framework en uso |
| Base de datos (migraciones, esquemas) | skill de DB / migrations |
| Documentos Word / PDF / planillas | skill docx, skill pdf, skill xlsx |
| Pruebas / QA | skill de testing del stack en uso |
| Código genérico, scripts, CLI | skill de file-reading, skill del lenguaje |

### 5.3 Regla adicional para Frontend — DESIGN.md

Antes de delegar una fase de dominio Frontend, verificar si existe `DESIGN.md` en el repositorio. Si existe, incluirlo como referencia obligatoria:

```
Referencia de diseño obligatoria: DESIGN.md
Antes de implementar cualquier componente, leé DESIGN.md para entender la paleta
de colores, tipografía, espaciado, componentes base y patrones visuales del proyecto.
No introduzcas estilos, tokens o componentes que contradigan lo definido allí.
Si necesitás algo que DESIGN.md no cubre, reportalo al orquestador en lugar de improvisar.
```

Si `DESIGN.md` no existe, omitir.

### 5.4 Sub-agentes auxiliares disponibles

Cuando la fase pinta candidata a necesitar exploración o documentación, ofrecer sdd-explore y/o sdd-archive como herramientas que el implementador puede spawnear si las necesita. Plantilla, criterios y tips de eficiencia en `subagentes_auxiliares.md` §4.

---

## 6. Validación del Entregable

La frecuencia y profundidad de la validación dependen de la criticidad declarada para la fase. Detalle completo en `supervision_y_validacion.md` §3.

| Criticidad | Modo de validación |
|---|---|
| Crítica | Inmediata — validar antes de delegar la siguiente fase |
| Estándar | Batch — acumular hasta 2 entregables antes de validar, salvo bloqueo / dependencia / sospecha de regresión |

### 6.1 Verificaciones obligatorias en cada validación

1. Alineación con la observación Engram de la fase (`mem_get_observation id={ID_fase_N}`).
2. Correctitud técnica local y consistencia con código existente.
3. Compatibilidad con contratos compartidos entre capas.
4. Suficiencia del contexto que usó el sub-agente (¿tuvo que inferir algo crítico?).

### 6.2 Decisión y registro

| Estado | Acción |
|---|---|
| ✅ Aprobada | Actualizar Engram → avanzar |
| ⚠️ Aprobada con observaciones | Actualizar Engram con deuda → avanzar |
| ❌ Rechazada | Re-delegar con feedback (ver `supervision_y_validacion.md` §4) |
| 🔒 Bloqueada | Registrar bloqueo en Engram → escalar |

### 6.3 Persistencia ante compactación a mitad de delegación

Si el orquestador detecta riesgo de compactación entre delegar y validar, registra un checkpoint adicional:

```
mem_save tipo:"bugfix"
título: "[PLAN:{nombre}] Checkpoint - Fase {N} delegada, esperando entregable"
contenido: sub-agente asignado, hora de delegación, estado actual
```

Esto permite recuperar el estado intermedio si el contexto se pierde antes de validar.

---

## 7. Recuperación de Estado en Nueva Sesión o Compactación

```
1. mem_context
   → inyecta contexto de sesión anterior automáticamente

2. mem_search "[PLAN:{nombre}] Estado global"
3. mem_get_observation id={ID_estado_global}
   → leer qué fases están pendientes, en curso o bloqueadas

4. Detectar zombies:
   Para cada fase "en_curso":
     mem_search "[PLAN:{nombre}] Estado Fase {N}"
     → si NO existe cierre correspondiente, es zombie
     → resolver según supervision_y_validacion.md §5

5. mem_search "[PLAN:{nombre}] Restricciones"
6. Continuar desde la primera fase no completada
```

**Regla**: nunca asumir que una fase está completa porque el orquestador "recuerda" haberla delegado. Leer siempre el estado real desde Engram.

---

## 8. Auditoría Final del Plan

Cuando todas las fases reportan estado de cierre, el orquestador no termina ahí. Ejecuta una auditoría final que verifica:

- **Cobertura del objetivo**: ¿todo lo que el plan se proponía está cubierto?
- **Consistencia inter-fases**: ¿los contratos compartidos quedaron alineados?
- **Desvíos**: ¿alguna fase introdujo cambios fuera de su alcance?
- **Deuda acumulada**: ¿qué quedó pendiente y qué es bloqueante?

Solo cuando la auditoría confirma cobertura + consistencia, el orquestador escribe el cierre final y termina la sesión. Detalle completo en `supervision_y_validacion.md` §6.

```
# Las cuatro operaciones siguientes son SECUENCIALES, una termina antes que la siguiente.
# Disparar en paralelo rompe SQLite. Esperar respuesta de cada una antes de continuar.

1. mem_save tipo:"plan"
   título: "[PLAN:{nombre}] Auditoría final"
   contenido: cobertura, consistencia, desvíos, deuda, recomendaciones, apto_para_uso
   → esperar respuesta

2. mem_save tipo:"plan"
   título: "[PLAN:{nombre}] Cierre final"
   → esperar respuesta

3. mem_session_summary
     Goal / Discoveries / Accomplished / Files
   → esperar respuesta

4. mem_session_end
```

Si la auditoría detecta desvíos bloqueantes, **no se cierra el plan**: se abre fase de remediación o se escala al humano.

---

## 9. Reglas de Control Obligatorias

- Indexar el plan completo en Engram **antes** de delegar la primera fase.
- **Escrituras a Engram siempre secuenciales, NUNCA en paralelo.** SQLite no soporta concurrencia: cada `mem_save` debe terminar y devolver su ID antes de iniciar el siguiente. Aplica a indexación inicial, cierres de fase, auditoría final y cualquier escritura. Ver `engram_ops.md` §0.
- **Reemplazar los placeholders de IDs por valores reales antes de delegar.** Si la delegación incluye `{ID_fase_N}` literalmente, el sub-agente no puede consultar Engram. Sustituir por los IDs concretos devueltos por `mem_save`.
- Clasificar la criticidad de cada fase al indexarla.
- Llamar `mem_session_start` al inicio y `mem_session_summary` + `mem_session_end` al cerrar.
- Guardar los IDs devueltos por `mem_save` durante la sesión activa.
- Validar fases críticas inmediatamente; nunca acumular críticas.
- Validar fases estándar al menos cada 2 entregables.
- Re-delegar con feedback estructurado tras un rechazo, no repetir la delegación original.
- No avanzar si existe divergencia contractual no resuelta.
- No permitir que sub-agentes usen `mem_search` libre ni `mem_save` sin autorización.
- Ejecutar auditoría final antes de marcar el plan como completado.
- Ante compactación inminente: registrar checkpoint y llamar `mem_session_summary`.

---

## 10. Anti-Patrones a Evitar

- Disparar varios `mem_save` en paralelo (Promise.all, tool calls simultáneos): rompe SQLite.
- Pasar la delegación con placeholders `{ID_fase_N}` sin reemplazar por IDs reales.
- Delegar sin indexar el plan en Engram primero.
- Ejecutar todas las fases y validar solo al final.
- Tratar todas las fases con la misma frecuencia de validación (críticas y estándar mezcladas).
- Re-delegar tras rechazo sin pasar el feedback del intento previo.
- Cerrar el plan sin auditoría final del conjunto.
- Asumir que el estado en memoria activa coincide con el estado en Engram.
- Permitir que sub-agentes consulten Engram con queries genéricas.
- Sugerir skills o sub-agentes que no existen en el entorno.
- Incluir IDs Engram pero no instruir su consulta proactiva al inicio.
- Encadenar fases sin actualizar Engram entre una y otra.
- Ignorar fases zombie al recuperar sesión.
- Cerrar el plan con desvíos detectados sin escalar.

---

## 11. Regla Central

**El orquestador no delega responsabilidad: delega ejecución. Su trabajo no termina cuando el sub-agente entrega; termina cuando el plan completo está auditado, consistente y registrado en Engram. Las fases críticas se vigilan inmediatamente; las estándar al menos cada 2; el plan completo siempre. Engram es la fuente de verdad; las skills y sub-agentes auxiliares son herramientas que el orquestador ofrece y el sub-agente decide usar.**

---

## 12. Versión Breve para Activación Rápida

1. **Indexar** plan en Engram (`mem_save` por fase, con criticidad declarada, + estado global).
2. **Preparar delegación**: contexto curado, IDs Engram, skills sugeridas, sub-agentes auxiliares ofrecidos, criticidad declarada.
3. **Delegar** una fase a la vez al sub-agente correspondiente.
4. **Validar** según criticidad: crítica → inmediato, estándar → cada 2.
5. **Re-delegar** con feedback estructurado si hay rechazo (máximo 3 intentos antes de escalar).
6. **Actualizar** Engram con estado real tras cada validación.
7. **Auditar** el plan completo al cerrar todas las fases: cobertura, consistencia, desvíos, deuda.
8. **Recuperar** con `mem_context` + `mem_search` ante compactación o nueva sesión; detectar zombies.

El orquestador vigila tanto como delega. Cada fase debe quedar bien hecha; el plan entero debe quedar coherente.
