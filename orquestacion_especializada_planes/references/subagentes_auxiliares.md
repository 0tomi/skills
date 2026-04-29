# Sub-agentes Auxiliares para el Implementador

> Este documento explica cómo el orquestador sugiere al agente implementador
> (sdd-apply o equivalente) que **él** spawnee sub-sub-agentes auxiliares
> cuando los necesita. La sugerencia es informativa, no obligatoria —
> igual que las skills.

## Tabla de Contenidos

1. [Idea general](#idea)
2. [sdd-explore — exploración del repositorio](#explore)
3. [sdd-archive — documentación persistente](#archive)
4. [Cómo redactar la sugerencia en la delegación](#sugerencia)
5. [Anti-patrones](#antipatrones)

---

## 1. Idea General {#idea}

El agente implementador puede spawnear sus propios sub-agentes especializados durante la ejecución de una fase. Los más útiles son:

- **sdd-explore** — para entender código existente sin gastar contexto del implementador.
- **sdd-archive** — para dejar documentado un hallazgo, decisión o patrón emergente.

El orquestador no le ordena al implementador usarlos: le **avisa** que están disponibles y le da tips para que los aproveche bien si decide spawnearlos.

La razón de existir de esta sugerencia: muchos implementadores ejecutan la fase completa con su propio contexto, sin saber que pueden delegar exploración o documentación a un proceso aparte. El resultado son fases con menos profundidad de la que podrían tener, o decisiones que se pierden al cerrar.

---

## 2. sdd-explore — Exploración del Repositorio {#explore}

### Cuándo el implementador debería considerarlo

- La fase requiere entender código existente que no fue pasado en la delegación.
- Hace falta mapear dependencias, encontrar dónde se valida X, ver cómo se usa un módulo en otros lugares.
- El implementador siente que está "a ciegas" sobre una parte del repo y no quiere abrir 15 archivos uno por uno.
- Hay un patrón establecido en el proyecto que el implementador no conoce y debe replicar.

### Cuándo NO usarlo

- Cuando el orquestador ya pasó el contexto completo de los archivos relevantes.
- Para tareas pequeñas en archivos ya identificados (basta leerlos directo).
- Cuando la respuesta requiere entender lógica de negocio amplia (eso es para Engram + plan, no para sdd-explore).

### Tips de eficiencia para sacarle jugo

1. **Objetivo concreto, no genérico.**
   - ❌ "Explorá el repo y decime cómo está estructurado."
   - ✅ "Encontrá dónde se valida el formato de email y devolveme el archivo + líneas exactas."

2. **Entregable acotado y declarado.**
   - Pedir al sub-agente exactamente qué formato de salida querés: "lista de archivos con función responsable" o "diagrama textual de dependencias del módulo X".

3. **Limitar el alcance.**
   - Indicar qué directorios mirar y cuáles ignorar. "Mirá solo `src/services/` y `src/models/`. Ignorá `tests/` y `vendor/`."

4. **Una pregunta por invocación.**
   - No pedir 5 cosas distintas en un solo spawn. El sub-agente performa mejor con un objetivo único.

5. **Pedir conclusiones, no volcado.**
   - El sub-agente debe sintetizar lo que encontró, no devolver todo el contenido leído. El implementador no necesita los archivos crudos, necesita la respuesta.

### Plantilla de spawn (la usa el implementador, no el orquestador)

```
Tarea: {pregunta concreta}
Alcance: {directorios o archivos donde buscar}
Fuera de alcance: {qué ignorar}
Entregable: {formato exacto de salida}
Restricción: solo lectura. No modificar nada.
```

---

## 3. sdd-archive — Documentación Persistente {#archive}

### Cuándo el implementador debería considerarlo

- Se tomó una decisión técnica no trivial durante la fase (un trade-off, un workaround, una elección entre dos enfoques).
- Apareció un patrón nuevo que va a usarse en fases futuras.
- Se documentó un comportamiento no obvio del sistema que un próximo agente necesitará entender.
- Se encontró una limitación de una librería o framework que afecta decisiones futuras.

### Cuándo NO usarlo

- Para cambios triviales o esperados por el plan (eso ya queda en Engram al cerrar la fase).
- Para reportar bugs corregidos (eso va en el reporte de cambios del entregable).
- Cuando lo que hay que documentar es estado del plan — eso lo persiste el orquestador en Engram, no el implementador en archive.

### Diferencia con Engram

- **Engram** persiste el **estado del plan**: qué se hizo, qué fase está donde, IDs de fase. Lo escribe el orquestador.
- **sdd-archive** persiste **conocimiento del proyecto**: por qué se decidió X, cómo funciona Y, qué pasa si se modifica Z. Lo escribe el implementador (o el sub-agente que él spawnea).

### Tips de eficiencia

1. **Pasarle el contenido ya sintetizado.**
   - El sub-agente no debe inferir qué documentar; el implementador llega con la decisión ya tomada y le pide al archive que la deje persistida.

2. **Especificar el lugar destino.**
   - ❌ "Documentá esto en algún lado."
   - ✅ "Agregá esto a `docs/architecture/decisions.md` bajo la sección 'Decisiones de validación'."

3. **Formato consistente.**
   - Si el proyecto tiene un formato (ADR, decisión + contexto + consecuencia), pedirlo explícitamente.

4. **No documentar lo obvio.**
   - Si es información que está en el código de forma legible, no archivarla. Documentar solo lo que requiere contexto extra para entender.

### Plantilla de spawn

```
Tarea: documentar {decisión / patrón / hallazgo}
Contenido a documentar: {texto ya redactado por el implementador}
Destino: {archivo + sección}
Formato: {ADR | nota de decisión | sección de README | etc}
Restricción: solo escritura en el archivo destino. No modificar otros archivos.
```

---

## 4. Cómo Redactar la Sugerencia en la Delegación {#sugerencia}

El orquestador incluye este bloque en la delegación al implementador, **solo cuando aplique**:

```
## Sub-agentes auxiliares disponibles

Si durante la implementación lo necesitás, podés spawnear:

- sdd-explore: si necesitás entender código existente que no te pasé.
  Tips: tarea concreta, alcance limitado, una pregunta por invocación,
  pedile conclusiones sintetizadas (no volcado de archivos).
  Útil acá si: {razón específica conectada a esta fase, o "no aplica obvio"}

- sdd-archive: si tomás una decisión técnica no trivial o detectás un patrón
  que conviene dejar documentado para fases futuras.
  Tips: pasale el contenido ya redactado, especificale archivo + sección destino,
  no archives lo obvio.
  Útil acá si: {razón específica conectada a esta fase, o "no aplica obvio"}

No estás obligado a usarlos. Son herramientas para que no gastes tu propio
contexto en exploración o no pierdas decisiones importantes al cerrar.
```

### Reglas de cuándo incluir cada uno

| sdd-explore | sdd-archive |
|---|---|
| Incluir si la fase toca código existente del que el orquestador no pasó contexto completo | Incluir si la fase puede generar decisiones técnicas, patrones nuevos, o hallazgos sobre el sistema |
| Omitir si todos los archivos relevantes ya están en la delegación | Omitir si la fase es puramente mecánica (renombrar, mover, aplicar receta clara) |

Si **ninguno** aplica para la fase, omitir el bloque entero. No incluir un bloque vacío "por las dudas".

---

## 5. Anti-patrones {#antipatrones}

- Sugerir sdd-explore cuando el orquestador ya pasó todo el contexto necesario → ruido.
- Sugerir sdd-archive en fases triviales → el implementador termina archivando obviedades.
- Dar tips genéricos en vez de específicos a la fase → el implementador los ignora.
- Decirle al implementador "tenés que usar sdd-explore" → es sugerencia, no orden.
- No incluir tips de eficiencia → el implementador spawnea con prompts vagos y obtiene resultados pobres.
- Confundir sdd-archive con Engram → archive documenta el proyecto, Engram persiste el plan.
