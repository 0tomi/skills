# Supervisión y Validación del Plan

> El orquestador no solo distribuye tareas: vigila que cada fase se haya implementado
> correctamente y que el conjunto sea consistente con el plan original.
>
> Esta es una de sus responsabilidades centrales, no un paso opcional.

> **Recordatorio sobre Engram**: todas las escrituras (`mem_save`, `mem_session_summary`)
> que aparecen en este archivo deben ejecutarse SECUENCIALMENTE, nunca en paralelo.
> SQLite no soporta concurrencia. Esperar respuesta de cada llamada antes de la siguiente.
> Detalle en `engram_ops.md` §0.

## Tabla de Contenidos

1. [Filosofía de vigilancia activa](#filosofia)
2. [Clasificación de criticidad de fases](#criticidad)
3. [Modos de validación: inmediata vs batch](#modos)
4. [Protocolo de re-delegación tras rechazo](#redelegacion)
5. [Manejo de estado zombie ("en_curso" sin cierre)](#zombie)
6. [Auditoría final de cierre del plan](#auditoria)
7. [Modo dry-run: preview de delegaciones](#dryrun)

---

## 1. Filosofía de Vigilancia Activa {#filosofia}

El orquestador es responsable de:

1. Que cada fase se implemente alineada con su definición en Engram.
2. Que el conjunto de fases ejecutadas mantenga coherencia con el plan original.
3. Que las desviaciones detectadas se registren y se resuelvan, no se acumulen como deuda silenciosa.

La vigilancia se ejecuta en dos momentos:

- **Durante la ejecución**: tras cada entregable, según criticidad de la fase (ver §3).
- **Al cierre del plan**: auditoría final que revisa el conjunto contra la meta original (ver §6).

No alcanza con que cada fase pase su propia validación: el plan completo debe seguir teniendo sentido al final.

---

## 2. Clasificación de Criticidad de Fases {#criticidad}

El orquestador clasifica cada fase como **crítica** o **estándar** al momento de indexarla en Engram. La clasificación determina cuándo y cómo se valida.

### Fase crítica

Cumple alguno de estos criterios:

- Toca contratos compartidos entre capas (ej: schema de API consumido por frontend).
- Modifica esquema de base de datos o migraciones con datos en producción.
- Implementa lógica de negocio sensible: pagos, autenticación, autorización, datos legales.
- Toca código de seguridad o validación de entrada.
- Es bloqueante de muchas fases siguientes (alta dependencia).
- Introduce un patrón nuevo que será replicado por otras fases.

### Fase estándar

- Cambios localizados sin impacto cruzado significativo.
- Componentes UI aislados que siguen patrones existentes.
- Refactors menores, ajustes de configuración, fixes de bugs delimitados.
- QA / testing rutinario sobre código ya validado.

### Cómo registrar la criticidad

En la observación `[PLAN:{nombre}] Fase {N}: {nombre}`, agregar el campo:

```
CRITICIDAD: critica | estandar
RAZÓN DE CRITICIDAD: {por qué — solo si crítica}
```

### Regla práctica

Ante la duda, marcar como crítica. El costo de validar de más es bajo; el costo de no validar una fase crítica puede propagarse a varias fases siguientes.

---

## 3. Modos de Validación: Inmediata vs Batch {#modos}

### Modo inmediato (fases críticas)

Tras recibir el entregable de una fase crítica:

1. El orquestador valida **antes** de delegar la siguiente fase.
2. No avanza hasta haber aprobado, rechazado, o registrado bloqueo.
3. La validación es individual y completa para esa fase.

```
Fase crítica → entregable → VALIDAR → registrar en Engram → delegar siguiente
```

### Modo batch (fases estándar)

Para fases estándar, el orquestador puede acumular hasta 2 entregables antes de validar, siempre que:

- las fases acumuladas no compartan archivos entre sí;
- no haya dependencia directa entre ellas (la 2da no necesita la 1ra cerrada);
- ambas hayan reportado entregable completo.

```
Fase estándar 1 → entregable
Fase estándar 2 → entregable
                → VALIDAR ambas → registrar en Engram → continuar
```

**Por qué batch en estándar:** reduce overhead de validación en fases de bajo riesgo. La validación sigue siendo obligatoria; lo que cambia es la frecuencia.

### Reglas de excepción

- Si una fase estándar reporta bloqueo o discrepancia → validar inmediatamente, no acumular.
- Si la fase siguiente es crítica → validar las pendientes antes de delegar la crítica.
- Si una fase estándar tarda mucho → validar lo que haya y no esperar al par.
- Si hay sospecha de regresión → validar inmediatamente, romper el batch.

### Validación batch paso a paso

```
1. mem_get_observation id={ID_fase_A}    ← contenido original
2. mem_get_observation id={ID_fase_B}    ← contenido original
3. Revisar entregable A contra ID_fase_A
4. Revisar entregable B contra ID_fase_B
5. Revisar interacción entre A y B (¿se pisaron archivos? ¿contratos consistentes?)
6. Decisión por fase: aprobar / rechazar / observar
7. Registrar cierres en Engram (uno por fase)
```

---

## 4. Protocolo de Re-Delegación tras Rechazo {#redelegacion}

Cuando una fase es rechazada, la nueva delegación no es igual a la primera: debe incluir el feedback del rechazo y referenciar el intento previo.

### Pasos obligatorios

1. **Registrar el rechazo en Engram** primero, antes de re-delegar:

```
mem_save  tipo:"bugfix"
título: "[PLAN:{nombre}] Estado Fase {N} - RECHAZADA (intento {K})"
contenido:
  estado: rechazada
  intento_numero: K
  motivo_rechazo: {descripción exacta}
  archivos_que_se_tocaron_mal: [lista]
  qué_debe_corregirse: {instrucción precisa}
  responsable: ejecucion | delegación_insuficiente
```

2. **Decidir el responsable**: ¿el sub-agente ejecutó mal o el orquestador delegó mal? Esto cambia el approach del nuevo prompt.

3. **Construir la re-delegación** con campos extra:

```
## Esta es una RE-DELEGACIÓN — intento {K+1}

### Contexto del intento previo
- ID en Engram: {ID_estado_rechazado}
- Motivo del rechazo: {breve}
- Lo que falló: {descripción concreta}

### Qué corregir específicamente
{instrucción puntual y verificable}

### Lo que NO debe volver a pasar
{lista corta de lo evitable}

### Lo que sí está bien del intento previo (si aplica)
{para que no rehaga lo correcto}
```

4. **Mantener el resto de la delegación igual** al primer intento (contexto de fase, archivos permitidos, restricciones, IDs Engram, skills sugeridas, sub-agentes auxiliares disponibles, etc.).

### Si el problema fue delegación insuficiente

El orquestador no acusa al sub-agente. Reformula la delegación con más contexto, declara que el rechazo previo fue por contexto faltante, y registra en Engram como `falla_de_transferencia_de_contexto`.

### Límite de reintentos

Si una fase es rechazada 3 veces, el orquestador debe:

- detener los reintentos automáticos;
- marcar la fase como `bloqueada` en Engram;
- escalar al humano con el resumen de los 3 intentos y sus motivos de rechazo.

---

## 5. Manejo de Estado Zombie ("en_curso" sin Cierre) {#zombie}

Una fase queda en estado zombie cuando fue marcada `en_curso` pero nunca se registró su cierre en Engram. Esto puede pasar por:

- timeout o crash del sub-agente;
- compactación a mitad de delegación sin recovery;
- cambio de sesión sin cerrar el ciclo.

### Cómo detectar zombies

Al recuperar sesión (§7 del SKILL.md principal), tras leer `[PLAN:{nombre}] Estado global`:

```
1. Identificar fases marcadas "en_curso"
2. Para cada una:
   mem_search "[PLAN:{nombre}] Estado Fase {N}"
   → si NO existe observación de cierre → es zombie
   → si existe pero el estado_global no se actualizó → es desync
```

### Resolución

| Caso | Acción |
|---|---|
| Zombie sin entregable disponible | Marcar como `parcial` en Engram, re-delegar limpio con nota de que el intento previo no completó |
| Zombie con entregable parcial accesible | Recuperar el entregable, validar lo que haya, decidir si completar o re-delegar |
| Desync (cierre existe, global desactualizado) | Actualizar `[PLAN:{nombre}] Estado global` con el estado real del cierre |

### Prevención

- Antes de cualquier compactación inminente: registrar checkpoint `[PLAN:{nombre}] Checkpoint pre-compactación` con la fase activa y su estado.
- Tras delegar una fase crítica: hacer un `mem_save` adicional registrando "fase N delegada, esperando entregable de {sub-agente}".
- Nunca confiar en que el estado en RAM del orquestador se mantendrá: persistir cualquier transición en Engram.

---

## 6. Auditoría Final de Cierre del Plan {#auditoria}

Cuando todas las fases reportan estado de cierre, el orquestador no termina ahí. Ejecuta una auditoría final que verifica que el plan completo siga siendo coherente.

### Pasos de la auditoría

```
1. mem_get_observation id={ID_meta}
   → recordar el objetivo general del plan

2. mem_search "[PLAN:{nombre}] Estado Fase"
   → recuperar todos los cierres de fase

3. Para cada fase, mem_get_observation del cierre
   → revisar archivos tocados, supuestos, deuda, desvíos

4. Construir tabla consolidada:
   - fase | estado | archivos | desvíos | deuda
```

### Verificaciones obligatorias

**Cobertura del objetivo**
- ¿Todo lo que el plan se proponía lograr está cubierto por alguna fase completada?
- ¿Quedaron objetivos parciales sin cubrir?

**Consistencia inter-fases**
- ¿Algún archivo fue tocado por fases que no debían interactuar?
- ¿Algún contrato compartido (ej: schema de API) quedó modificado por una fase de un dominio y consumido por otra que no fue actualizada?
- ¿Las fases siguientes a una con deuda técnica la heredaron correctamente o la ignoraron?

**Desvíos del plan**
- ¿Alguna fase introdujo cambios fuera de su alcance original?
- ¿Algún supuesto declarado por un sub-agente contradice supuestos de otras fases?
- ¿La arquitectura final coincide con la planeada en `[PLAN:{nombre}] Restricciones globales`?

**Deuda acumulada**
- Listar toda la deuda técnica registrada por las fases.
- Identificar cuál es bloqueante para uso productivo y cuál puede quedar pendiente.

### Resultado de la auditoría

```
mem_save  tipo:"plan"
título: "[PLAN:{nombre}] Auditoría final"
contenido:
  cobertura_objetivo: completa | parcial — {detalle}
  consistencia_interfases: ok | con observaciones — {detalle}
  desvíos_detectados: ninguno | lista
  deuda_consolidada: lista
  recomendaciones: {acciones sugeridas para resolver desvíos / deuda}
  apto_para_uso: sí | no — {condiciones}
```

### Si la auditoría detecta problemas

- Inconsistencias menores → registrar y dejar al humano la decisión de corregirlas ahora o en un nuevo plan.
- Inconsistencias bloqueantes → no marcar el plan como completado; abrir fase de remediación con su propia delegación.
- Desvíos del objetivo → escalar al humano antes de cerrar.

Solo cuando la auditoría confirma cobertura + consistencia, el orquestador escribe `[PLAN:{nombre}] Cierre final` y llama `mem_session_summary` + `mem_session_end`.

---

## 7. Modo Dry-Run: Preview de Delegaciones {#dryrun}

Para planes complejos, el orquestador puede generar la lista completa de delegaciones planeadas **antes** de ejecutar la primera, y presentarla al humano para validación.

### Cuándo conviene activarlo

- Planes con más de 5 fases.
- Planes que tocan múltiples dominios sensibles (legales, pagos, datos productivos).
- Cuando el humano lo pide explícitamente.
- Primera ejecución de un nuevo tipo de plan (sin precedente exitoso).

### Qué incluye el dry-run

```
Para cada fase planeada:
  - Sub-agente objetivo
  - Criticidad declarada
  - Resumen de la delegación (objetivo + archivos + criterio de cierre)
  - Skills sugeridas
  - Sub-agentes auxiliares ofrecidos (sdd-explore / sdd-archive)
  - Modo de validación que se aplicará (inmediata / batch)
  - Dependencias con fases previas
```

### Procedimiento

1. El orquestador indexa el plan en Engram normalmente (§4 del SKILL.md).
2. Antes de la primera delegación, genera el preview completo.
3. Se lo presenta al humano.
4. Espera confirmación o ajustes.
5. Si hay ajustes, actualiza las observaciones de fase en Engram antes de delegar.

### Cuándo omitirlo

- Planes simples de pocas fases.
- Cuando el humano pidió ejecución directa.
- Planes recurrentes ya validados en ejecuciones previas.

El dry-run es opcional. Si se omite, el resto del flujo se ejecuta normalmente.
