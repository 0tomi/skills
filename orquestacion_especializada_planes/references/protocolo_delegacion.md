# Protocolo Completo de Delegación

## Tabla de Contenidos

1. [Plantilla Obligatoria de Invocación](#plantilla)
2. [Dominios y sus alcances](#dominios)
3. [Reglas por dominio](#reglas-dominio)
4. [Manejo de Desviaciones y Bloqueos](#desviaciones)
5. [Formato de respuesta del sub-agente](#respuesta)
6. [Criterios de Calidad Mínimos](#calidad)

---

## 1. Plantilla Obligatoria de Invocación {#plantilla}

El orquestador debe estructurar cada delegación con este esquema:

### Sub-agente objetivo
`[backend | frontend | datos | infra | qa]`

### Plan vigente
`[nombre del plan activo]`

### Fase
`[identificador y nombre exacto]`

### Criticidad de la fase
`critica | no_critica` — el orquestador la clasificó así porque: `[razón]`

### Modo de validación esperado
`inmediata | batch` — qué va a hacer el orquestador al recibir el entregable

### Contenido de la fase
`[extracto fiel y autosuficiente de la fase a ejecutar — no solo el nombre]`

### Objetivo de la fase
`[qué habilita o resuelve esta fase dentro del plan general]`

### Objetivo local de la tarea
`[resultado técnico puntual esperado del sub-agente en esta delegación]`

### Precondiciones ya resueltas
- `[dependencia 1 — cómo fue resuelta]`
- `[dependencia 2 — cómo fue resuelta]`

### Archivos permitidos
- `[ruta/archivo_1]`
- `[ruta/archivo_2]`

### Archivos prohibidos o fuera de alcance
- `[ruta/archivo_x — razón por la que está fuera de alcance]`

### Referencias relevantes
- `[fragmento, contrato, esquema o archivo base]`

### Acceso a Engram

> **REGLA CRÍTICA PARA EL ORQUESTADOR**: los placeholders `{ID_fase_N}` y `{ID_restricciones}` NO se pasan literalmente. El orquestador DEBE reemplazarlos por los IDs reales que devolvió `mem_save` en la Fase 0. Si los placeholders llegan al sub-agente sin sustituir, el sub-agente no puede consultar Engram y falla en silencio. Antes de enviar la delegación, verificar que los IDs del bloque sean números/strings reales, no placeholders entre llaves.

Bloque a incluir en la delegación (con IDs ya sustituidos):

```
Antes de comenzar, consultá Engram para obtener el contexto completo de esta fase:
  mem_get_observation id=<ID_REAL_DE_LA_FASE>          ← contenido completo de la fase
  mem_get_observation id=<ID_REAL_DE_RESTRICCIONES>    ← convenciones y contratos del proyecto

Este mensaje es un extracto. La observación en Engram es la fuente completa y vigente.
Si algo en Engram contradice este mensaje, reportá la discrepancia antes de continuar.
NO hagas mem_search genérico ni accedas a otras observaciones del plan.
NO llames mem_save — el orquestador registra el cierre de fase.
```

Ejemplo de bloque correctamente preparado (los IDs son reales, no placeholders):

```
Antes de comenzar, consultá Engram para obtener el contexto completo de esta fase:
  mem_get_observation id=4827          ← contenido completo de la fase
  mem_get_observation id=4823          ← convenciones y contratos del proyecto
...
```
*(Omitir esta sección si el sub-agente no tiene acceso a Engram; en ese caso pasar todo el contexto inline. Si en ese caso el sub-agente reporta insuficiencia de contexto, el orquestador re-delega ampliando el contexto inline; nunca le exige consultar Engram.)*

### Skills sugeridas
```
- [{nombre_skill}]: {razón concreta por la que aplica a esta tarea}
- [{nombre_skill}]: {razón concreta}

No estás obligado a usarlas, pero consultarlas antes de implementar puede ahorrarte
trabajo y asegurar que el output esté alineado con las convenciones del proyecto.
```
*(Omitir si no hay skills conocidas que apliquen a la fase — no inventar)*

### Sub-agentes auxiliares disponibles
```
Si durante la implementación lo necesitás, podés spawnear:

- sdd-explore: si necesitás entender código existente que no te pasé.
  Tips: tarea concreta, alcance limitado, una pregunta por invocación,
  pedile conclusiones sintetizadas (no volcado de archivos).
  Útil acá si: {razón conectada a esta fase}

- sdd-archive: si tomás una decisión técnica no trivial o detectás un patrón
  que conviene dejar documentado para fases futuras.
  Tips: pasale el contenido ya redactado, especificale archivo + sección destino,
  no archives lo obvio.
  Útil acá si: {razón conectada a esta fase}

No estás obligado a usarlos.
```
*(Omitir el bloque entero si ninguno aplica — ver `subagentes_auxiliares.md` para criterios)*

### Restricciones
- seguir estrictamente la fase asignada;
- no rediseñar arquitectura ni objetivos de negocio;
- no modificar capas o archivos fuera del alcance declarado;
- no buscar contexto en planes viejos o documentos alternativos sin autorización;
- si el contexto recibido es insuficiente: detener y reportar bloqueo de contexto, no inferir.

### Criterio de finalización
`[condición verificable que marca la fase como terminada]`

Ejemplos:
- endpoint implementado, alineado con esquema y validaciones, tests pasando;
- componente renderiza estados loading/error/success según contrato de API;
- migración consistente con modelo y consultas existentes;
- pipeline desplegado y validado en entorno objetivo.

### Entregable esperado
El sub-agente debe devolver:
- lista de cambios realizados;
- archivos tocados (rutas exactas);
- supuestos usados durante la ejecución;
- bloqueos o inconsistencias detectadas;
- confirmación explícita de si el contexto recibido fue suficiente o no;
- sugerencias para `notas_para_siguiente_agente` si las hay.

---

## 2. Dominios y sus Alcances {#dominios}

### Backend
- Lógica de negocio, contratos de API, validaciones de servidor.
- Jobs, eventos, colas, seguridad de servidor.
- No toca: migraciones de esquema (dominio Datos), componentes UI (Frontend).

### Frontend
- Componentes UI, estado cliente, navegación, formularios.
- Consumo de APIs, renderizado, UX funcional.
- No toca: lógica de negocio de servidor, esquemas de BD.
- **DESIGN.md**: si existe en el repositorio, es referencia obligatoria. El sub-agente debe leerlo antes de implementar y no puede introducir estilos, tokens ni componentes que lo contradigan. Si necesita algo que DESIGN.md no cubre, lo reporta al orquestador.

### Datos / DB
- Migraciones, esquemas, índices, seeds.
- Compatibilidad de modelos con la BD.
- No toca: lógica de negocio, componentes UI.

### Infra / DevOps
- Despliegue, configuración, pipelines, variables de entorno.
- Observabilidad, contenedores, redes.
- No toca: lógica de aplicación, esquemas de datos.

### QA / Test
- Pruebas unitarias, integración, e2e.
- Validación de cobertura de rutas críticas.
- No toca: implementación de funcionalidad nueva.

---

## 3. Reglas por Dominio {#reglas-dominio}

### Separación estricta

Cada sub-agente opera dentro de su dominio. Si una fase involucra varios dominios, el orquestador debe **dividirla** en subtareas coordinadas.

Excepción: el plan puede justificar explícitamente que un sub-agente cubra dos dominios relacionados (ej: backend + validaciones de datos en el mismo endpoint). Debe documentarse en la delegación.

### Paralelismo permitido solo cuando

- las tareas no comparten archivos conflictivos;
- los contratos entre capas ya están definidos y persistidos en Engram;
- la integración posterior es verificable.

### Granularidad

No delegar más de una fase activa simultánea al mismo sub-agente.

### Una salida por iteración

Cada sub-agente devuelve un resultado acotado, enfocado y revisable.

---

## 4. Manejo de Desviaciones y Bloqueos {#desviaciones}

### Si el sub-agente se desvía

Si un sub-agente:
- modifica archivos fuera de alcance,
- altera contratos no autorizados,
- introduce lógica no pedida,
- rompe consistencia con otra capa,
- usa referencias de planes viejos sin autorización,
- evidencia no haber seguido la fase asignada,

el orquestador debe:

1. rechazar el entregable parcial;
2. identificar la desviación concreta;
3. distinguir si el error fue de **ejecución** (sub-agente) o **delegación insuficiente** (orquestador);
4. emitir instrucción correctiva precisa;
5. impedir el avance hasta validar la corrección;
6. actualizar Engram con el estado `rechazada` y la razón.

### Si el sub-agente reporta insuficiencia de contexto

1. No penalizar al sub-agente.
2. Revisar la delegación propia: ¿se envió contenido operativo o solo identificador?
3. Ampliar el contexto y re-delegar.
4. Registrar en Engram como `falla_de_transferencia_de_contexto`.

### Si el plan tiene inconsistencias

Marcar como `inconsistencia_del_plan` en Engram. No continuar hasta redefinir.

---

## 5. Formato de Respuesta del Sub-agente {#respuesta}

Todo sub-agente debe devolver obligatoriamente:

```
## Cambios realizados
[descripción de qué se implementó]

## Archivos tocados
- ruta/archivo_1 — [qué cambió]
- ruta/archivo_2 — [qué cambió]

## Supuestos usados
[supuestos que el sub-agente tomó durante la ejecución]

## Bloqueos o inconsistencias detectadas
[ninguno | descripción del bloqueo]

## Suficiencia de contexto
[El contexto recibido fue suficiente / insuficiente en: {detalle}]

## Notas para siguiente agente
[qué debe saber quien continúe la ejecución]
```

---

## 6. Criterios de Calidad Mínimos {#calidad}

Antes de aprobar una fase, verificar:

**Obligatorio:**
- alineación con la observación Engram de la fase (`mem_get_observation id={ID_fase_N}`);
- correctitud lógica;
- consistencia con la arquitectura existente;
- ausencia de cambios laterales injustificados;
- compatibilidad entre capas involucradas;
- trazabilidad de qué se modificó y por qué;
- suficiencia del contexto usado para ejecutar.

**Cuando aplique:**
- validaciones de entrada;
- manejo de errores;
- seguridad básica;
- pruebas críticas;
- impacto en migraciones o datos existentes;
- retrocompatibilidad.
