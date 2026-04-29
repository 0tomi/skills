---
name: orquestacion_especializada_planes
description: Skill para orquestar, vigilar y validar la ejecución de un plan técnico con sub-agentes especializados, usando Engram como memoria persistente del plan. El orquestador no solo delega: supervisa activamente cada entregable, verifica que la implementación coincida con el plan, ajusta el rigor de validación según la criticidad de cada fase, y audita la consistencia global al cierre. Activar cuando haya un plan con múltiples fases, riesgo de pérdida de contexto entre agentes, necesidad de delegación técnica multi-dominio, o cuando el trabajo pueda interrumpirse y reanudarse. Palabras clave: "orquestar agentes", "plan de fases", "delegación técnica", "sub-agentes", "memoria de plan", "ejecución por etapas".
---

# SKILL: Orquestación Especializada de Planes con Memoria Persistente

> **Referencias modulares:**
> - `references/protocolo_delegacion.md` — plantilla de delegación, dominios, manejo de bloqueos, formato de respuesta
> - `references/engram_ops.md` — operaciones Engram completas: API, plantillas, recuperación
> - `references/supervision_y_validacion.md` — vigilancia activa, criticidad, modos de validación, re-delegación, auditoría final
> - `references/subagentes_auxiliares.md` — sdd-explore y sdd-archive: cuándo sugerirlos, tips de eficiencia

---

## 1. Propósito y Doble Responsabilidad

Esta skill define cómo un agente orquestador coordina sub-agentes especializados durante la ejecución de un **Plan de Implementación**, usando **memoria persistente con Engram** como columna vertebral del estado compartido.

El orquestador tiene **dos responsabilidades**, no una:

1. **Delegar bien** — convertir fases del plan en tareas ejecutables con contexto autosuficiente, sugerir herramientas adecuadas, proteger el alcance.
2. **Vigilar la implementación** — verificar que cada fase se haya ejecutado correctamente. Para fases críticas validación inmediata; para fases no críticas se puede acumular hasta 2 antes de revisar. Al cierre del plan, auditar consistencia global.

Confiar en el reporte del sub-agente sin verificar es delegación, no orquestación. Ver detalles completos de vigilancia en `references/supervision_y_validacion.md`.

Esta skill **no reemplaza** al plan: lo persiste, lo distribuye, lo vigila y lo audita.

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

## 3. Flujo Maestro

```
INICIO
  │
  ▼
[FASE 0] ── Indexar plan en Engram ──────────────────────────────────────────┐
  │           • mem_session_start                                             │
  │           • Clasificar criticidad de cada fase (critica | no_critica)     │
  │           • mem_save: meta, restricciones, una obs. por fase, estado     │
  │           • Guardar IDs en contexto activo                                │
  │           • [Opcional] Generar dry-run para planes complejos              │
  │                                                                           │
  ▼                                                                           │
[FASE N] ── Preparar delegación ─────────────────────────────────────────────┤
  │           • mem_get_observation ID_fase_N → verificar precondiciones      │
  │           • Construir contexto curado (extracto de la fase)              │
  │           • Determinar skills disponibles que aplican → sugerirlas       │
  │           • Determinar si aplican sub-agentes auxiliares → sugerirlos     │
  │           • Incluir IDs Engram para consulta proactiva del sub-agente    │
  │                                                                           │
  ▼                                                                           │
[EJECUCIÓN] ── Sub-agente opera ────────────────────────────────────────────┤
  │           • Consulta Engram proactivamente al inicio (si tiene acceso)   │
  │           • Consulta skills sugeridas antes de implementar               │
  │           • Spawnea sdd-explore/sdd-archive si los necesita              │
  │           • Devuelve entregable + reporte de estado                      │
  │                                                                           │
  ▼                                                                           │
[VALIDACIÓN] ── Vigilancia activa ─────────────────────────────────────────┤
  │           • Si fase CRÍTICA → validación inmediata                       │
  │           • Si fase NO CRÍTICA → acumular hasta 2, luego revisar         │
  │           • Leer código real, no confiar en el reporte                   │
  │           • Decisión: Aprobada / Observaciones / Rechazada / Bloqueada   │
  │                                                                           │
  ▼                                                                           │
[ACTUALIZAR ENGRAM] ── Registrar resultado real ─────────────────────────────┤
  │           • mem_save: estado de fase                                     │
  │           • mem_save: estado global actualizado                          │
  │           • Registrar archivos tocados, desvíos, deuda técnica           │
  │                                                                           │
  ▼                                                                           │
¿Más fases? ── Sí → volver a [FASE N+1]                                      │
             └─ No → [AUDITORÍA FINAL]                                       │
                      • Validar fases no críticas pendientes del último batch │
                      • Auditar cobertura, consistencia, desvíos acumulados  │
                      • mem_save: cierre + auditoría                         │
                      • mem_session_summary + mem_session_end                │
```

---

## 4. Fase 0: Indexación del Plan en Engram

**Esta fase es obligatoria antes de delegar cualquier sub-agente.**

Engram no es un key-value store: persiste **observaciones estructuradas** recuperables por ID o por búsqueda full-text.

### 4.1 Observaciones requeridas al inicio

| Título de observación | Tipo | Contenido |
|---|---|---|
| `[PLAN:{nombre}] Meta y objetivo general` | `"plan"` | Nombre, versión, objetivo, dominio, total de fases |
| `[PLAN:{nombre}] Restricciones globales` | `"architecture"` | Convenciones, contratos, separación de capas |
| `[PLAN:{nombre}] Fase {N}: {nombre}` | `"decision"` | Contenido operativo + **criticidad** + justificación |
| `[PLAN:{nombre}] Estado global` | `"plan"` | Mapa de progreso: todas las fases en `"pendiente"` |

### 4.2 Clasificación de criticidad

Cada fase debe clasificarse al indexarse: `critica` o `no_critica`. Esta clasificación determina el modo de validación posterior. Ver criterios completos en `references/supervision_y_validacion.md` §2.

Regla rápida: si toca lógica de negocio central, contratos compartidos, datos sensibles, infra, o es bisagra para fases siguientes → **crítica**. En caso de duda → **crítica**.

### 4.3 Convención de títulos

Usar el prefijo `[PLAN:{nombre}]` en todos los títulos. Permite recuperar todas las memorias del plan con una sola búsqueda:

```
mem_search "[PLAN:suite_api_v2]"   → lista todas las observaciones del plan
```

### 4.4 Subdivisión de fases multi-dominio

Si una fase del plan original mezcla dominios (ej. backend + frontend en una misma unidad), el orquestador la subdivide al indexar:

```
[PLAN:{nombre}] Fase 2.a: {nombre} (backend)
[PLAN:{nombre}] Fase 2.b: {nombre} (frontend)
```

En el contenido de cada sub-fase, referenciar la fase padre y la otra sub-fase para contratos compartidos.

### 4.5 Versionado del plan si cambia

Si el plan cambia a mitad de ejecución (el humano lo edita), no eliminar las observaciones viejas. Crear una nueva versión:

```
[PLAN:{nombre}] Meta y objetivo general (v2)
```

Y en el contenido de la versión vieja anotar `OBSOLETA — reemplazada por v2 (ID: {nuevo_id})`. Las fases que cambien siguen el mismo patrón.

### 4.6 Secuencia de inicio

```
1. mem_session_start
2. mem_context                        ← verificar si el plan ya existe de sesión anterior
3. Clasificar criticidad de cada fase
4. mem_save para cada observación     ← guardar los IDs devueltos
5. Retener IDs en contexto activo     ← para acceso directo durante la sesión
```

Para planes complejos, considerar generar un **dry-run** antes de delegar la primera fase. Ver `references/supervision_y_validacion.md` §7.

Plantillas completas de contenido en `references/engram_ops.md` §4.

---

## 5. Preparación de la Delegación

Antes de invocar al sub-agente, el orquestador prepara cuatro elementos además del contexto de la fase: acceso a Engram, skills sugeridas, sub-agentes auxiliares disponibles, y alcance protegido.

### 5.1 Acceso a Engram (consulta proactiva)

Si el sub-agente tiene acceso a Engram, el orquestador **siempre** incluye los IDs relevantes e instruye al sub-agente a consultarlos **al inicio de su tarea**, no solo si tiene dudas. La observación en Engram es la fuente completa; el mensaje de delegación es un extracto.

**Instrucción estándar:**

```
Antes de comenzar, consultá Engram para obtener el contexto completo de esta fase:
  mem_get_observation id={ID_fase_N}          ← contenido completo de la fase
  mem_get_observation id={ID_restricciones}   ← convenciones y contratos del proyecto

El mensaje que te pasé es un extracto. La observación en Engram es la fuente completa y vigente.
Si algo en Engram contradice este mensaje, reportá la discrepancia al orquestador antes de continuar.
NO hagas mem_search genérico ni accedas a otras observaciones del plan.
NO llames mem_save — el orquestador registra el cierre de fase.
```

Si el sub-agente no tiene acceso a Engram: pasar todo el contexto inline y, si reporta insuficiencia, ampliar y re-delegar — nunca exigirle consultar Engram.

### 5.2 Skills sugeridas

El orquestador evalúa qué skills del sistema aplican a la fase y las sugiere. No es obligación, es guía.

| Dominio / tarea | Skills típicamente relevantes |
|---|---|
| Backend Laravel | skill de Laravel, skill de API design |
| Frontend | skill de frontend-design, skill del framework en uso |
| Base de datos | skill de DB / migrations |
| Documentos Word/PDF/planillas | skill docx, skill pdf, skill xlsx |
| Pruebas / QA | skill de testing del stack |
| Código genérico | skill de file-reading, skill del lenguaje |

El orquestador no inventa skills que no existen.

**Frontend — DESIGN.md obligatorio si existe:**

Antes de delegar fases de Frontend, verificar si existe `DESIGN.md` en el repositorio. Si existe, incluirlo como referencia obligatoria:

```
Referencia de diseño obligatoria: DESIGN.md
Antes de implementar cualquier componente, leé DESIGN.md para entender la paleta,
tipografía, espaciado, componentes base y patrones visuales del proyecto. No
introduzcas estilos, tokens o componentes que contradigan lo definido allí.
Si necesitás algo que DESIGN.md no cubre, reportalo al orquestador.
```

### 5.3 Sub-agentes auxiliares disponibles

El orquestador sugiere al implementador los sub-sub-agentes que puede spawnear si los necesita: **sdd-explore** para entender código existente, **sdd-archive** para documentar decisiones. Cada sugerencia incluye tips de eficiencia específicos.

Detalles completos, criterios de cuándo aplica cada uno, y plantillas en `references/subagentes_auxiliares.md`.

Solo incluir el bloque cuando alguno aplica. No aplicarlos siempre por reflejo.

### 5.4 Plantilla completa de delegación

Ver `references/protocolo_delegacion.md` §1.

---

## 6. Vigilancia: Validación según Criticidad

El orquestador valida cada entregable. El **modo** de validación depende de la criticidad de la fase:

- **Fase crítica** → **validación inmediata**: leer código, comparar contra Engram, decidir antes de avanzar.
- **Fase no crítica** → **validación batch**: acumular hasta 2 fases no críticas seguidas y revisarlas en conjunto, salvo que la siguiente sea crítica (en ese caso, validar el batch antes de delegarla).

Reglas duras:
- nunca acumular más de 2 fases sin revisar;
- una fase crítica rompe el batch (validar lo acumulado primero);
- nunca cruzar el cierre del plan con fases sin validar.

### 6.1 Decisión obligatoria

| Estado | Acción |
|---|---|
| ✅ Aprobada | Actualizar Engram → avanzar |
| ⚠️ Aprobada con observaciones | Actualizar Engram con deuda → avanzar |
| ❌ Rechazada | Diagnosticar causa → re-delegar (ver §6.3) |
| 🔒 Bloqueada | Registrar bloqueo en Engram → escalar |

### 6.2 Actualización de Engram al cierre

```
mem_save  tipo:"bugfix"
título: "[PLAN:{nombre}] Estado Fase {N} - {COMPLETADA|BLOQUEADA|RECHAZADA}"
contenido:
  estado, fecha_cierre, archivos_tocados, supuestos_usados,
  desvíos_detectados, deuda_técnica, contexto_fue_suficiente,
  notas_para_siguiente_agente

mem_save  tipo:"plan"
título: "[PLAN:{nombre}] Estado global"
contenido: mapa actualizado con nuevo estado de fase N
```

**Compactación a mitad de delegación**: si el orquestador siente que se va a compactar entre delegar y validar, hacer un `mem_save` extra: `[PLAN:{nombre}] Fase {N} - Delegada, esperando entregable de {sub-agente}`. Permite al orquestador retomar sin perder el hilo.

### 6.3 Re-delegación tras rechazo

No re-delegar lo mismo de nuevo. Diagnosticar primero la causa: error de ejecución, delegación insuficiente, o inconsistencia del plan. Persistir el intento rechazado en Engram. Construir la nueva delegación corrigiendo lo que falló. Tras 3 intentos rechazados de la misma fase, escalar al humano.

Detalles completos en `references/supervision_y_validacion.md` §5.

### 6.4 Estado zombie

Una fase en `en_curso` sin observación de cierre asociada es un estado zombie. Al recuperar sesión: detectar zombies, inspeccionar repo, decidir entre re-delegar limpio, completar desde estado actual, o rollback. Nunca asumir que un `en_curso` está completo.

Detalles en `references/supervision_y_validacion.md` §6.

Detalles operativos completos del modo inmediato y modo batch en `references/supervision_y_validacion.md` §3 y §4.

---

## 7. Recuperación de Estado en Nueva Sesión

Cuando el orquestador inicia en contexto nuevo (compactación, nueva sesión, agente diferente):

```
1. mem_context                               ← inyecta contexto de sesión anterior
2. mem_search "[PLAN:{nombre}] Estado global" → obtener ID más reciente
3. mem_get_observation id={ID_estado_global} → leer fases pendientes/en_curso/bloqueadas
4. Detectar zombies (en_curso sin cierre)   → resolver antes de continuar
5. mem_search "[PLAN:{nombre}] Restricciones" → cargar convenciones globales
```

Regla: el orquestador nunca asume que una fase está completa porque "recuerda" haberla delegado. Lee el estado real desde Engram.

---

## 8. Cierre del Plan: Auditoría Final

**Obligatoria. No saltarla aunque todas las fases hayan sido validadas individualmente.**

Una fase puede pasar su validación local y aun así dejar el plan globalmente inconsistente. La auditoría final detecta desvíos acumulados antes del cierre.

### 8.1 Qué revisa la auditoría

- **Cobertura del plan** — ¿todas las fases están en estado terminal? ¿hay zombies sin resolver?
- **Consistencia entre fases** — ¿los contratos definidos al inicio se respetan al final? ¿hay archivos modificados de forma incompatible entre fases?
- **Desvíos acumulados** — ¿cuántas fases se aprobaron con observaciones? ¿hay drift respecto a la intención original?
- **Deuda técnica** — ¿qué quedó pendiente? ¿requiere acción inmediata?
- **Archivos prohibidos** — ¿algún sub-agente tocó archivos fuera de su alcance declarado?

### 8.2 Veredicto

| Veredicto | Acción |
|---|---|
| `implementacion_consistente` | Cerrar el plan normalmente |
| `implementacion_con_observaciones` | Cerrar dejando deuda visible al humano |
| `implementacion_inconsistente` | **No cerrar limpio.** Escalar al humano con el reporte |

### 8.3 Persistencia del cierre

Solo si el veredicto permite cerrar:

```
mem_save  tipo:"plan" — "[PLAN:{nombre}] Auditoría final de cierre"
mem_save  tipo:"plan" — "[PLAN:{nombre}] Cierre final"
mem_session_summary — Goal / Discoveries / Accomplished / Files + veredicto
mem_session_end
```

Procedimiento completo de auditoría en `references/supervision_y_validacion.md` §8.

---

## 9. Reglas de Control Obligatorias

- Indexar el plan completo en Engram **antes** de delegar la primera fase.
- Clasificar criticidad de cada fase al indexarla.
- Llamar `mem_session_start` al inicio y `mem_session_summary` + `mem_session_end` al cerrar.
- Guardar los IDs devueltos por `mem_save` durante la sesión activa.
- Validar fases críticas inmediatamente; no acumular más de 2 fases no críticas sin revisar.
- Leer código real al validar — no confiar solo en el reporte del sub-agente.
- Actualizar Engram inmediatamente tras aprobar o rechazar cada entregable.
- Re-delegar solo tras diagnosticar la causa del rechazo (máx. 3 intentos por fase).
- Detectar y resolver zombies al recuperar sesión — nunca asumir que `en_curso` está completo.
- Ejecutar la auditoría final antes del cierre — sin excepciones.
- Ante compactación inminente: hacer `mem_save` extra del estado actual + `mem_session_summary`.

---

## 10. Anti-Patrones

- No indexar el plan en Engram antes de comenzar.
- No clasificar criticidad — todas las fases tratadas igual.
- Validar fases críticas en modo batch — el costo de un error tardío es exponencial.
- Acumular 3+ fases sin revisar.
- Confiar en el reporte del sub-agente sin leer código.
- Re-delegar lo mismo tras un rechazo, sin diagnosticar causa.
- Ignorar zombies al recuperar sesión.
- Saltar la auditoría final porque "todas las fases fueron aprobadas".
- No incluir IDs Engram en delegaciones cuando el sub-agente tiene acceso.
- Sugerir sdd-explore/sdd-archive sin tips específicos para la fase.
- Inventar skills que no existen en el entorno.

---

## 11. Regla Central

**El orquestador delega ejecución, no responsabilidad. Vigila cada entregable con rigor proporcional a la criticidad de la fase. Engram es la fuente de verdad del plan; las skills y sub-agentes auxiliares son herramientas que el orquestador sugiere al implementador. La auditoría final cierra el ciclo: ninguna implementación se da por completada sin verificar consistencia global.**

---

## 12. Versión Breve para Activación Rápida

1. **Indexar** el plan en Engram + clasificar criticidad de cada fase.
2. **Preparar** delegación: contexto curado + IDs Engram + skills + sub-agentes auxiliares.
3. **Delegar** una fase a la vez.
4. **Vigilar** según criticidad: inmediata para críticas, batch (≤2) para no críticas.
5. **Validar** leyendo código real, no solo el reporte.
6. **Actualizar** Engram tras cada cierre de fase.
7. **Re-delegar** con diagnóstico si hay rechazo.
8. **Auditar** consistencia global antes del cierre del plan.

El orquestador no es un repartidor de tareas: es un supervisor activo de la implementación.
