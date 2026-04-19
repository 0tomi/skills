---
name: orquestacion_especializada_planes
description: Skill para gestionar, delegar, supervisar y validar la ejecución de sub-agentes especializados durante la implementación técnica de un plan previamente definido, asegurando que cada sub-agente reciba el contexto completo y vigente de su fase, opere con alcance controlado y produzca entregables alineados con el diseño, el código existente y la integración final.
---

# SKILL: Orquestación Especializada de Planes

## 1. Propósito

Esta skill define cómo un agente orquestador debe coordinar sub-agentes especializados durante la ejecución de un **Plan de Implementación** ya existente.

Su objetivo es:

- convertir fases del plan en tareas ejecutables;
- asignarlas al especialista correcto;
- transferir a cada sub-agente el **contexto vigente, suficiente y específico** de la fase que debe ejecutar;
- evitar que los sub-agentes reconstruyan por su cuenta el contexto consultando planes viejos, parciales o ambiguos;
- controlar que cada ejecución respete el diseño original;
- validar integración, consistencia y calidad antes de avanzar.

Esta skill **no reemplaza** al plan de implementación: lo usa como contrato de ejecución.

---

## 2. Cuándo se Activa

Esta skill debe activarse **únicamente** cuando se cumplan todas las siguientes condiciones:

1. existe un Plan de Implementación explícito;
2. el trabajo requiere ejecución técnica en una o más capas del sistema;
3. hay riesgo de desalineación entre diseño, código e integración si una sola unidad ejecuta todo sin control intermedio.

No debe activarse para:

- brainstorming;
- diseño inicial sin fases definidas;
- tareas triviales de una sola capa;
- cambios menores que no requieran delegación ni supervisión multi-dominio.

---

## 3. Rol del Orquestador

El orquestador es responsable de traducir el plan en ejecución controlada. No actúa como simple despachador.

Sus responsabilidades obligatorias son:

- interpretar el plan y descomponerlo en fases operativas;
- determinar qué dominio debe ejecutar cada fase;
- **extraer y transferir el contenido relevante de la fase**, no solo su número, nombre o identificador;
- entregar a cada sub-agente el contexto **suficiente para ejecutar sin tener que inferir qué quiso decir el plan**;
- asegurar que el contexto transferido pertenezca al **plan vigente**, no a versiones previas ni documentos alternativos;
- controlar que cada salida respete el plan, el código existente y las restricciones del sistema;
- detectar desviaciones, inconsistencias o efectos colaterales;
- ordenar correcciones antes de permitir que el plan continúe;
- validar integración entre dominios antes del cierre.

El orquestador conserva siempre la autoridad sobre:

- prioridades;
- orden de ejecución;
- criterios de aceptación;
- aprobación o rechazo de entregables parciales.

---

## 4. Principios de Orquestación

### 4.1 Separación estricta de dominios

Cada sub-agente debe operar dentro de un dominio técnico definido para evitar solapamientos, acoplamiento accidental y cambios contradictorios.

Ejemplos de dominios:

- **Backend:** lógica de negocio, persistencia, contratos de API, validaciones de servidor, jobs, eventos, seguridad de servidor.
- **Frontend:** componentes UI, estado cliente, navegación, formularios, consumo de APIs, renderizado y UX funcional.
- **Datos/DB:** migraciones, esquemas, índices, seeds, compatibilidad de modelos.
- **Infra/DevOps** (si aplica): despliegue, configuración, pipelines, variables de entorno, observabilidad.
- **QA/Test** (si aplica): pruebas unitarias, integración, e2e, validación de cobertura crítica.

Si una fase involucra varios dominios, el orquestador debe **dividirla** en subtareas coordinadas, no entregarla mezclada a un único sub-agente salvo que el plan lo justifique de forma explícita.

### 4.2 Contexto suficiente, no mínimo por reflejo

Cada sub-agente debe recibir el **mínimo necesario para evitar ruido**, pero el **máximo necesario para no perder comprensión**.

La regla correcta no es “dar poco contexto”, sino **dar contexto suficiente y curado**.

Esto incluye, según corresponda:

- contenido de la fase vigente del plan;
- objetivo de esa fase dentro del plan general;
- objetivo local de la tarea delegada;
- precondiciones y dependencias ya resueltas;
- archivos objetivo exactos;
- contratos, esquemas o fragmentos de referencia relevantes;
- convenciones existentes del proyecto;
- restricciones de arquitectura;
- criterio de salida esperado.

No se debe saturar al sub-agente con información global irrelevante, pero **tampoco se le debe obligar a reconstruir el contexto faltante por inferencia**.

### 4.3 Contexto vigente y anti-ambigüedad

El sub-agente nunca debe recibir solo referencias como:

- “hacé la fase 3”;
- “implementá lo de la fase backend”; 
- “seguí con la siguiente parte del plan”.

Eso es contexto insuficiente y genera ambigüedad.

Cada delegación debe contener el **extracto operativo de la fase** o una reformulación fiel de su contenido, de manera que el sub-agente pueda ejecutar sin buscar por su cuenta a qué fase se refiere el orquestador.

### 4.4 Prohibición de búsqueda autónoma de contexto de fase

El sub-agente **no debe buscar en planes antiguos, versiones previas o documentos alternativos** para deducir qué significa la fase asignada, salvo que el orquestador lo autorice de forma explícita.

Si la fase delegada no trae contexto suficiente para ejecutarse con seguridad, el sub-agente debe:

1. detener la ejecución;
2. reportar insuficiencia de contexto;
3. solicitar ampliación o aclaración al orquestador.

No debe completar huecos con suposiciones.

### 4.5 Fidelidad al plan

Ningún sub-agente puede reinterpretar objetivos de negocio ni rediseñar arquitectura por su cuenta.

Si detecta que una fase del plan:

- es inconsistente,
- incompleta,
- inviable,
- o entra en conflicto con el código real,

entonces debe reportarlo al orquestador en lugar de improvisar una solución estructural no autorizada.

### 4.6 Validación incremental

Ninguna fase se considera cerrada solo porque “compila” o “parece correcta”.

Cada fase debe pasar por validación técnica y de integración antes de habilitar la siguiente.

---

## 5. Protocolo de Delegación

Cada invocación a un sub-agente debe incluir, como mínimo, los siguientes bloques.

### 5.1 Identificación de la fase

Debe indicarse:

- identificador o nombre exacto de la fase;
- posición de la fase dentro del plan;
- versión o referencia del plan vigente, si aplica.

El identificador por sí solo **no alcanza**.

### 5.2 Contenido operativo de la fase

El orquestador debe pasar el **contenido concreto de la fase**, resumido o citado de forma fiel.

Debe incluir, según aplique:

- qué se busca lograr en esa fase;
- qué cambios contempla;
- qué dependencias asume resueltas;
- qué restricciones impone;
- qué resultado habilita para la fase siguiente.

### 5.3 Objetivo de la fase y objetivo local

Cada delegación debe incluir dos niveles de objetivo:

- **Objetivo de la fase:** para qué existe esa fase dentro del plan general.
- **Objetivo local de la tarea:** qué debe implementar o modificar exactamente el sub-agente en esta delegación.

El sub-agente no debe recibir solo “qué hacer”, sino también **por qué esa tarea existe en la secuencia de implementación**.

### 5.4 Alcance exacto

Debe indicarse:

- qué archivos puede modificar;
- qué archivos puede crear;
- qué archivos no debe tocar;
- qué capas del sistema quedan fuera del alcance.

### 5.5 Contexto suficiente y curado

Debe incluir:

- extracto de la fase vigente del plan;
- contratos, esquemas o fragmentos de código relevantes;
- dependencias necesarias;
- convenciones existentes del proyecto;
- supuestos que no debe alterar;
- decisiones ya tomadas que afecten esta fase.

Regla: **no pasar contexto de menos; no pasar contexto basura; pasar contexto suficiente para ejecutar sin reconstrucción externa.**

### 5.6 Restricciones

Debe especificarse claramente:

- no desviarse del plan;
- no consultar planes viejos para completar ambigüedades;
- no refactorizar fuera del alcance;
- no cambiar contratos sin autorización;
- no introducir librerías o patrones nuevos sin justificación explícita;
- no romper compatibilidad con fases ya validadas.

### 5.7 Criterio de finalización

Cada sub-agente debe saber cuándo su tarea se considera terminada.

Ejemplos:

- endpoint implementado y alineado con esquema y validaciones;
- componente renderiza estados loading/error/success;
- tests críticos agregados y pasando;
- migración consistente con modelo y consultas existentes.

### 5.8 Formato de respuesta del sub-agente

Todo sub-agente debe devolver:

- cambios realizados;
- archivos tocados;
- supuestos usados;
- bloqueos o inconsistencias detectadas;
- confirmación explícita de si el contexto recibido fue suficiente o no.

---

## 6. Plantilla Obligatoria de Invocación

El orquestador debe estructurar cada delegación con este esquema:

### Sub-agente objetivo

`[backend | frontend | datos | infra | qa]`

### Plan vigente

`[nombre, versión o referencia del plan activo]`

### Fase

`[identificador y nombre exacto]`

### Contenido de la fase

`[extracto fiel y autosuficiente de la fase a ejecutar]`

### Objetivo de la fase

`[qué habilita o resuelve esta fase dentro del plan general]`

### Objetivo local de la tarea

`[resultado técnico puntual esperado del sub-agente]`

### Precondiciones ya resueltas

- `[dependencia 1]`
- `[dependencia 2]`

### Archivos permitidos

- `[ruta/archivo_1]`
- `[ruta/archivo_2]`

### Archivos prohibidos o fuera de alcance

- `[ruta/archivo_x]`

### Referencias relevantes

- `[fragmento, contrato, esquema o archivo base]`

### Restricciones

- seguir estrictamente la fase asignada;
- no rediseñar;
- no modificar otras capas;
- no buscar contexto en planes antiguos sin autorización;
- reportar bloqueos o insuficiencia de contexto.

### Entregable esperado

- cambios realizados;
- resumen técnico breve;
- supuestos detectados;
- riesgos o bloqueos;
- confirmación de suficiencia de contexto.

---

## 7. Reglas de Ejecución

### 7.1 Granularidad controlada

No delegar más de una fase activa simultánea al mismo sub-agente.

### 7.2 Dependencias explícitas

Una fase no debe comenzar si depende de otra que aún no fue validada, salvo que el plan contemple paralelismo seguro.

### 7.3 Paralelismo limitado

Solo se permite ejecución paralela cuando:

- las tareas no comparten archivos conflictivos;
- los contratos entre capas ya están definidos;
- la integración posterior es verificable.

### 7.4 Una salida por iteración

Cada sub-agente debe devolver un resultado acotado, enfocado y revisable.

### 7.5 Sin autonomía arquitectónica

Los sub-agentes ejecutan. El orquestador decide.

### 7.6 No ejecutar con contexto dudoso

Si la delegación no contiene el contenido suficiente de la fase, el sub-agente no debe adivinar, completar huecos ni buscar equivalentes aproximados en otros planes.

Debe declararlo como bloqueo de contexto.

---

## 8. Supervisión y Loop de Calidad

Después de cada entrega, el orquestador debe ejecutar un ciclo obligatorio de validación.

### 8.1 Verificación contra el plan

Comprobar que la implementación:

- corresponde exactamente a la fase asignada;
- usa como referencia el **plan vigente correcto**;
- no omitió requisitos de la fase;
- no agregó comportamiento fuera de alcance.

### 8.2 Verificación de suficiencia de contexto

El orquestador debe revisar también su propia delegación.

Debe comprobar:

- que no envió solo el nombre o número de la fase;
- que el extracto de fase era autosuficiente;
- que el sub-agente no tuvo que buscar contexto externo;
- que el contexto pasado fue suficiente, relevante y vigente.

Si falla esto, el problema no es del sub-agente: es de la orquestación.

### 8.3 Verificación técnica local

Evaluar:

- coherencia lógica;
- consistencia con el código existente;
- manejo de errores;
- compatibilidad de tipos, contratos o esquemas;
- impacto en dependencias y flujos relacionados.

### 8.4 Verificación inter-dominios

Si hay más de una capa involucrada, revisar que:

- frontend y backend usen el mismo contrato;
- modelos y persistencia reflejen la lógica implementada;
- nombres, estados y estructuras sean compatibles;
- no existan supuestos contradictorios entre sub-agentes.

### 8.5 Decisión obligatoria

Toda entrega debe terminar en uno de estos estados:

- **Aprobada** → puede avanzar la siguiente fase.
- **Aprobada con observaciones** → puede avanzar, dejando constancia de deuda acotada.
- **Rechazada** → debe corregirse antes de continuar.
- **Bloqueada** → requiere redefinición o ampliación de contexto.

---

## 9. Manejo de Desviaciones y Bloqueos

Si un sub-agente:

- modifica archivos fuera de alcance,
- altera contratos no autorizados,
- introduce lógica no pedida,
- rompe consistencia con otra capa,
- usa referencias de planes antiguos sin autorización,
- o evidencia no haber seguido la fase asignada,

el orquestador debe:

1. rechazar el entregable parcial;
2. identificar la desviación concreta;
3. distinguir si el error fue de ejecución o de contexto mal transferido;
4. emitir una instrucción correctiva precisa;
5. impedir el avance hasta validar la corrección.

Si el problema nace en el plan o en una delegación insuficiente del orquestador, debe marcarse como:

- **inconsistencia del plan**, o
- **falla de transferencia de contexto**.

No debe disfrazarse como fallo técnico del sub-agente si en realidad recibió una orden ambigua.

---

## 10. Criterios de Calidad Mínimos

Antes de aprobar una fase, el orquestador debe verificar como mínimo:

- alineación con el plan vigente;
- correctitud lógica;
- consistencia con la arquitectura existente;
- ausencia de cambios laterales injustificados;
- compatibilidad entre capas involucradas;
- trazabilidad clara de qué se modificó y por qué;
- suficiencia del contexto que fue usado para ejecutar.

Cuando aplique, también debe revisar:

- validaciones;
- manejo de errores;
- seguridad básica;
- pruebas críticas;
- impacto en migraciones o datos existentes;
- retrocompatibilidad.

---

## 11. Reglas de Control Obligatorias

- No delegar más de una fase del plan simultáneamente a un mismo sub-agente.
- No delegar una fase solo por identificador: debe transferirse su contenido operativo.
- Validar cada tarea terminada antes de proceder a la siguiente fase dependiente.
- Mantener el contexto de cada sub-agente aislado y enfocado.
- Mantener el contexto **suficiente**, no artificialmente mínimo.
- No permitir rediseños implícitos durante la ejecución.
- No aceptar entregables sin contraste contra el plan.
- No avanzar con integración si existe una divergencia contractual no resuelta.
- No mezclar diagnóstico, implementación y validación sin dejar explícito qué rol se está ejecutando.
- No permitir que un sub-agente consulte planes antiguos para completar instrucciones ambiguas, salvo autorización explícita.

---

## 12. Cierre de Implementación

Una vez ejecutadas todas las fases, el orquestador debe realizar una validación final de conjunto.

Debe comprobar:

- que todas las fases del plan fueron ejecutadas;
- que no quedaron incompatibilidades entre dominios;
- que los contratos definidos en diseño coinciden con el código final;
- que los cambios parciales forman una solución integrada y operativa;
- que ninguna fase fue ejecutada con contexto ambiguo o heredado de un plan incorrecto.

El cierre debe incluir un resumen con:

- fases ejecutadas;
- sub-agentes involucrados;
- archivos afectados;
- desvíos corregidos;
- fallas de contexto detectadas y corregidas;
- riesgos remanentes o deuda técnica detectada.

---

## 13. Anti-Patrones a Evitar

Esta skill prohíbe expresamente los siguientes comportamientos:

- delegar tareas ambiguas o abiertas;
- pasar solo el nombre o número de una fase;
- asumir que el sub-agente ya sabe a qué versión del plan se hace referencia;
- pasar contexto masivo sin curación;
- pasar contexto tan escaso que obligue al sub-agente a inferir o buscar afuera;
- permitir que un sub-agente redefina el problema;
- validar superficialmente solo por ausencia de errores sintácticos;
- ejecutar frontend y backend en paralelo sin contrato claro;
- aceptar cambios fuera de alcance “porque ya estaban hechos”;
- encadenar varias fases sin puntos de control.

---

## 14. Resumen Operativo

La lógica de esta skill es simple:

1. tomar una fase concreta del plan vigente;
2. extraer su contenido operativo;
3. asignarla al especialista correcto;
4. pasarle contexto suficiente, curado y vigente;
5. exigir ejecución fiel al alcance;
6. impedir ejecución si la delegación quedó ambigua;
7. validar técnica y funcionalmente la salida;
8. corregir desviaciones;
9. avanzar recién cuando la fase esté aprobada.

El orquestador no delega responsabilidad: delega ejecución.

---

## 15. Regla Central

**Cada sub-agente debe recibir no solo el identificador de la fase, sino también el contenido operativo de esa fase, su objetivo dentro del plan, el objetivo local de la tarea, las precondiciones relevantes, los archivos afectados, las restricciones y un criterio de salida verificable. Si ese contexto no fue transferido de forma suficiente y vigente, la ejecución no debe comenzar.**

---

## 16. Versión Breve para Activación Rápida

Usar esta skill cuando exista un plan de implementación y haya que ejecutar fases con sub-agentes especializados sin perder coherencia arquitectónica.

Regla central resumida: **no delegar “fases” como etiquetas; delegar fases como unidades de trabajo autosuficientes, con contexto vigente, objetivo de fase, objetivo local, alcance, restricciones y criterio de validación.**

