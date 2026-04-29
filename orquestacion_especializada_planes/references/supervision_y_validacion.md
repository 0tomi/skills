# Supervisión, Validación y Cierre

> El orquestador no es un repartidor de tareas. Es un supervisor activo de la
> implementación. Esta referencia define cómo vigila, cuándo revisa, qué hace
> cuando algo falla, y cómo audita el cierre del plan.

## Tabla de Contenidos

1. [Filosofía: el orquestador vigila](#filosofia)
2. [Criticidad de la fase y modos de validación](#criticidad)
3. [Modo "validación inmediata"](#inmediata)
4. [Modo "validación batch cada 2 fases"](#batch)
5. [Re-delegación tras rechazo](#redelegacion)
6. [Manejo de estado zombie "en_curso"](#zombie)
7. [Modo dry-run del plan](#dryrun)
8. [Auditoría final de cierre](#auditoria)

---

## 1. Filosofía: el Orquestador Vigila {#filosofia}

El orquestador tiene dos responsabilidades, no una:

1. **Delegar bien** — pasar contexto suficiente, sugerir herramientas, proteger el alcance.
2. **Vigilar la implementación** — verificar que cada fase se haya ejecutado correctamente, no solo que el sub-agente diga que terminó.

Confiar en el reporte del sub-agente sin verificar es delegación, no orquestación. El reporte puede ser optimista, incompleto, o describir algo distinto a lo que efectivamente se implementó.

Vigilar significa, según la criticidad de la fase, **leer el código real**, **comparar contra la observación de la fase en Engram**, **chequear archivos tocados**, **detectar cambios laterales no declarados**.

---

## 2. Criticidad de la Fase y Modos de Validación {#criticidad}

No todas las fases requieren el mismo nivel de revisión. El orquestador clasifica cada fase antes de delegarla y registra la criticidad en su observación Engram.

### Criterio de clasificación

Una fase es **crítica** si cumple al menos uno:

- toca lógica de negocio central (autenticación, permisos, cálculos monetarios, datos legales);
- afecta contratos compartidos entre capas (API, esquemas de BD, eventos públicos);
- modifica migraciones, esquemas o seeds de producción;
- introduce o modifica código de seguridad (validaciones, sanitización, control de acceso);
- toca infraestructura de despliegue;
- es una fase bisagra (su salida habilita varias fases siguientes);
- el plan la marca explícitamente como crítica.

Una fase es **no crítica** si:

- es un cambio acotado en una sola capa con bajo blast radius;
- es refactor cosmético, ajuste de UI sin lógica nueva, mensajes de log, comentarios;
- es una sub-tarea pequeña parte de una fase más grande;
- el costo de un error es bajo y reversible sin impacto en otras fases.

Cuando hay duda, **clasificar como crítica**. El costo de revisar de más es bajo; el costo de no revisar una fase crítica es alto.

### Registro en Engram

Al crear la observación de fase en la Fase 0, agregar al contenido:

```
CRITICIDAD: critica | no_critica
JUSTIFICACION CRITICIDAD: {por qué se clasificó así}
```

---

## 3. Modo "Validación Inmediata" {#inmediata}

**Aplica a fases críticas.**

Apenas el sub-agente entrega, el orquestador valida antes de avanzar a la siguiente fase. Sin excepciones.

### Procedimiento

1. **Leer la observación Engram de la fase** — `mem_get_observation id={ID_fase_N}`. Esto define qué se esperaba.
2. **Leer el reporte del sub-agente** — qué dice que hizo, archivos tocados, supuestos.
3. **Verificación cruzada** — leer al menos los archivos principales que el sub-agente declaró haber tocado. Confirmar:
   - los cambios existen y se ven como se describen;
   - no hay cambios laterales en archivos no declarados;
   - el contrato (si la fase tenía uno) se respeta tal como pedía el plan;
   - las restricciones globales del proyecto se cumplen.
4. **Verificación lógica** — el código tiene sentido para el objetivo, no solo compila.
5. **Verificación inter-dominios** — si la fase produce algo que otra capa va a consumir, ¿el formato es el correcto?
6. **Decisión** — Aprobada / Aprobada con observaciones / Rechazada / Bloqueada.
7. **Actualizar Engram** con el resultado real.

Solo después de aprobada se delega la siguiente fase.

### Costo y por qué vale la pena

Validación inmediata gasta tokens del orquestador. Pero detecta errores antes de que se propaguen a fases dependientes, donde el costo de corrección es exponencial.

---

## 4. Modo "Validación Batch cada 2 Fases" {#batch}

**Aplica a fases no críticas.**

El orquestador permite acumular hasta 2 fases no críticas seguidas antes de hacer una validación conjunta. Acelera la ejecución sin perder vigilancia.

### Cuándo dispara la validación

- Tras la segunda fase no crítica acumulada.
- Antes de la siguiente fase si esa siguiente es crítica (no se puede entrar a una fase crítica con deuda de validación atrás).
- Si una fase no crítica reporta algo raro (supuestos sospechosos, archivos no esperados), validar inmediatamente aunque no llegue al batch de 2.
- Antes del cierre final del plan (no quedan fases pendientes de validar).

### Procedimiento

1. Para cada fase del batch:
   - leer su observación Engram;
   - leer el reporte del sub-agente;
   - leer los archivos clave declarados.
2. Verificar consistencia entre las fases del batch (¿la segunda asumió correctamente lo que hizo la primera?).
3. Decisión por fase: aprobada / con observaciones / rechazada.
4. Si alguna queda rechazada, no avanzar hasta resolver.
5. Actualizar Engram con resultado de cada fase.

### Reglas duras

- **Nunca acumular más de 2 fases sin revisar.** Tres es demasiado: si la primera tenía un problema, la tercera ya está corrupta.
- **Una fase crítica rompe el batch.** Si después de 1 fase no crítica viene una crítica, validar la no crítica acumulada antes de delegar la crítica.
- **El batch nunca cruza un cierre de plan.** Antes del cierre se validan todas las pendientes.

---

## 5. Re-delegación tras Rechazo {#redelegacion}

Cuando una fase es rechazada, no se re-delega "lo mismo de nuevo". Hay un protocolo para que la segunda intentona no repita el error.

### Antes de re-delegar

1. **Diagnosticar la causa real del rechazo.**
   - ¿Fue **error de ejecución** del sub-agente? (no siguió la fase, modificó cosas fuera de alcance).
   - ¿Fue **delegación insuficiente** del orquestador? (no se le pasó contexto necesario, ambigüedad en la fase).
   - ¿Fue **inconsistencia del plan**? (la fase tal como está escrita no se puede ejecutar).
2. **Registrar el diagnóstico en Engram** como una observación adicional asociada a la fase.

### Persistir el intento previo

```
mem_save  tipo:"bugfix"
título: "[PLAN:{nombre}] Fase {N} - Intento rechazado #{K}"
contenido:
  intento_numero: K
  fecha: {timestamp}
  causa_rechazo: ejecucion | delegacion_insuficiente | inconsistencia_plan
  detalle: {qué falló concretamente}
  archivos_afectados_en_intento: {lista — pueden necesitar rollback}
  correccion_a_aplicar: {qué cambia en la próxima delegación}
```

### Construir la nueva delegación

Según el diagnóstico:

| Causa | Qué cambia en la nueva delegación |
|---|---|
| Error de ejecución | Misma fase, **agregar restricciones explícitas** sobre lo que falló, citar el intento anterior |
| Delegación insuficiente | Misma fase, **ampliar contexto** con lo que faltaba, considerar incluir más IDs Engram |
| Inconsistencia del plan | **No re-delegar.** Detener y resolver el plan primero (escalar al humano si hace falta) |

### Rollback de archivos del intento anterior

Si el intento rechazado dejó archivos modificados, decidir antes de re-delegar:

- **rollback completo** — descartar lo del intento previo, empezar limpio;
- **rollback parcial** — quedarse con lo que sí estuvo bien, descartar lo desviado;
- **continuar desde donde quedó** — solo si el sub-agente próximo recibe instrucciones precisas sobre el estado actual.

Cualquiera de las tres se documenta en la nueva delegación.

### Límite de intentos

Tras **3 intentos rechazados** de la misma fase, no re-delegar más. Marcar la fase como `bloqueada` con motivo "intentos agotados", actualizar Engram, y escalar al humano.

---

## 6. Manejo de Estado Zombie "en_curso" {#zombie}

Una fase en estado `en_curso` que no tiene una observación de cierre asociada en Engram es un **estado zombie**: el orquestador la delegó, pero nunca la cerró.

### Cuándo aparece

- El sub-agente nunca devolvió respuesta (timeout, error de runtime, sesión cerrada).
- El orquestador se compactó después de delegar pero antes de validar.
- Hubo crash o interrupción manual.

### Cómo detectarlo al recuperar sesión

```
1. mem_search "[PLAN:{nombre}] Estado global"
   → leer estado actual de cada fase

2. Para cada fase en estado "en_curso":
   mem_search "[PLAN:{nombre}] Estado Fase {N}"
   → ¿existe observación de cierre?

3. Si NO existe observación de cierre → es zombie.
```

### Qué hacer con un zombie

1. **Inspeccionar el repositorio** — ¿hay archivos modificados consistentes con la fase? ¿O el repo está limpio?
2. **Decisión según evidencia:**
   - **Repo limpio** → marcar la fase como `pendiente` (nunca llegó a ejecutarse) y re-delegar limpio.
   - **Repo con cambios coherentes con la fase** → tratar como entregable parcial: validar lo que hay, decidir si completarlo o revertir.
   - **Repo con cambios incoherentes o a mitad** → revertir cambios del zombie, marcar `pendiente`, re-delegar.
3. **Registrar el zombie en Engram:**

```
mem_save  tipo:"bugfix"
título: "[PLAN:{nombre}] Fase {N} - Zombie detectado"
contenido:
  fecha_deteccion: {timestamp}
  evidencia_en_repo: {limpio | cambios coherentes | cambios incoherentes}
  resolucion: re-delegacion_limpia | completar_desde_estado_actual | rollback_y_redelegar
```

### Regla dura

**Nunca asumir que una fase en `en_curso` está completa.** Si no hay observación de cierre, no está cerrada — punto. Verificar siempre.

---

## 7. Modo Dry-Run del Plan {#dryrun}

**Opcional, pero recomendado para planes con más de 5 fases o alta complejidad.**

Antes de ejecutar la primera delegación, el orquestador puede generar un **plan de delegaciones** completo: la lista de qué va a delegar a quién, en qué orden, con qué clasificación de criticidad. El humano lo valida antes de que se gaste un solo token de implementación.

### Cuándo activarlo

- Planes largos (>5 fases).
- Planes que cruzan dominios sensibles (datos de producción, infra).
- Cuando el humano lo pide explícitamente.
- Cuando el plan recién fue redefinido y se quiere revisar la nueva estructura.

### Qué entrega el dry-run

Un documento con, por cada fase:

- nombre y objetivo;
- sub-agente destinatario;
- criticidad clasificada y justificación;
- modo de validación esperado (inmediata / batch);
- skills que se sugerirán;
- IDs Engram que se referenciarán;
- precondiciones requeridas (qué fases deben estar completas antes);
- riesgos o ambigüedades detectadas en el plan original.

### Procedimiento

1. Indexar el plan en Engram (Fase 0 normal).
2. **Antes** de delegar la primera fase, generar el dry-run.
3. Mostrar al humano y esperar confirmación o ajustes.
4. Si hay ajustes, actualizar las observaciones Engram correspondientes.
5. Comenzar a delegar.

### Persistir el dry-run

```
mem_save  tipo:"plan"
título: "[PLAN:{nombre}] Dry-run de delegaciones"
contenido: {tabla completa de delegaciones planificadas}
```

---

## 8. Auditoría Final de Cierre {#auditoria}

**Obligatoria. No saltarla aunque todas las fases hayan sido validadas individualmente.**

Una fase puede pasar su validación local y aun así dejar el plan globalmente inconsistente. La auditoría final detecta desvíos acumulados.

### Cuándo se ejecuta

Tras la última fase validada, antes de hacer `mem_save` del cierre y antes de `mem_session_end`.

### Qué revisa

#### 8.1 Cobertura del plan

- ¿Todas las fases del plan original están en estado `completada` o `parcial`?
- ¿Hay fases pendientes que se "olvidaron"?
- ¿Hay fases en estado `en_curso` zombie sin resolver?

#### 8.2 Consistencia entre fases

- Los contratos definidos en fases tempranas, ¿siguen respetados en las últimas?
- Los archivos tocados por una fase, ¿no fueron alterados por otra de forma incompatible?
- Las restricciones globales registradas en `[PLAN:{nombre}] Restricciones globales`, ¿se cumplen al final?

#### 8.3 Desvíos acumulados

- ¿Cuántas fases reportaron desvíos pequeños "aprobados con observaciones"? Si hay más de 2, el plan puede haber drifteado lejos de la intención original.
- ¿Cuántos supuestos no documentados se acumularon a lo largo de la ejecución?

#### 8.4 Deuda técnica registrada

- ¿Cuál es la deuda técnica acumulada en los cierres de fase?
- ¿Hay deuda crítica que requiere acción inmediata vs. registrable para después?

#### 8.5 Verificación de archivos prohibidos

- Recorrer todas las observaciones de cierre de fase.
- ¿Algún sub-agente tocó archivos que estaban en la lista de prohibidos de su fase?

### Procedimiento de auditoría

```
1. mem_search "[PLAN:{nombre}]"
   → recuperar todas las observaciones del plan

2. Cargar la lista de fases originales (de [PLAN:{nombre}] Meta)
   y compararla con los estados finales en [PLAN:{nombre}] Estado global

3. Para cada fase, leer su observación de cierre y registrar:
   - estado final
   - archivos tocados
   - desvíos
   - deuda técnica

4. Verificación cruzada de contratos:
   - tomar las restricciones globales y verificar que ninguna fase las haya roto

5. Generar reporte de auditoría
```

### Persistir el reporte de auditoría

```
mem_save  tipo:"plan"
título: "[PLAN:{nombre}] Auditoría final de cierre"
contenido:
  fecha_auditoria: {timestamp}

  cobertura:
    fases_completadas: [lista]
    fases_parciales: [lista, con razón]
    fases_no_ejecutadas: [lista, con razón]
    zombies_resueltos: [lista]

  consistencia:
    contratos_respetados: sí | no — {detalle si hay roturas}
    archivos_con_modificaciones_cruzadas: [lista o ninguno]
    restricciones_globales_violadas: [lista o ninguna]

  desvios_acumulados:
    fases_con_observaciones: {cantidad y lista}
    supuestos_no_documentados: {cantidad}
    drift_estimado: bajo | medio | alto

  deuda_tecnica:
    items: [lista priorizada]
    requieren_accion_inmediata: [lista o ninguno]

  archivos_prohibidos_tocados: [lista o ninguno]

  veredicto: implementacion_consistente | implementacion_con_observaciones | implementacion_inconsistente

  recomendaciones: {acciones sugeridas tras el cierre}
```

### Veredicto final

| Veredicto | Significado | Acción |
|---|---|---|
| `implementacion_consistente` | Todo el plan se ejecutó como se esperaba, sin desvíos significativos | Cerrar el plan normalmente |
| `implementacion_con_observaciones` | Hay desvíos menores documentados, deuda técnica registrada | Cerrar el plan, dejar la deuda visible al humano |
| `implementacion_inconsistente` | Hay desvíos serios, archivos prohibidos tocados, contratos rotos, o cobertura incompleta | **No cerrar el plan limpio.** Escalar al humano con el reporte de auditoría |

### Después de la auditoría

Si el veredicto permite cerrar:

```
1. mem_save  tipo:"plan"
   título: "[PLAN:{nombre}] Cierre final"
   contenido: resumen + referencia al ID de la auditoría

2. mem_session_summary
   Goal / Discoveries / Accomplished / Files
   incluir veredicto de auditoría

3. mem_session_end
```

Si el veredicto es `implementacion_inconsistente`, no llamar `mem_session_end` hasta resolver con el humano.
