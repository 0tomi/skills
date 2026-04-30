# Sub-agentes Auxiliares: sdd-explore y sdd-archive

> Esta referencia describe cómo el orquestador sugiere al sub-agente implementador
> que **él mismo** spawnee sub-sub-agentes auxiliares cuando los necesite.
>
> El orquestador no obliga: ofrece la herramienta y da tips de eficiencia.
> El implementador decide si los usa según la situación concreta.

## Tabla de Contenidos

1. [Modelo conceptual](#modelo)
2. [sdd-explore — exploración acotada del repositorio](#explore)
3. [sdd-archive — documentación persistente de decisiones](#archive)
4. [Cómo sugerir su uso en una delegación](#sugerencia)
5. [Anti-patrones al usar sub-sub-agentes](#anti-patrones)

---

## 1. Modelo Conceptual {#modelo}

```
[Orquestador]
     │ delega fase →
     ▼
[Implementador (sdd-apply / etc.)]
     │ puede spawnear ↓ si lo necesita
     ├─ [sdd-explore]   → exploración rápida y acotada
     └─ [sdd-archive]   → documentación persistente
```

**Diferencia clave con las skills:**
- Skills = archivos de instrucciones que el agente lee para informar su ejecución.
- Sub-sub-agentes = procesos vivos que ejecutan una tarea acotada y devuelven un resultado.

**Diferencia clave con Engram:**
- Engram persiste el plan y su estado (lo controla el orquestador).
- sdd-archive persiste decisiones de código / arquitectura / patrones (lo dispara el implementador).

El orquestador menciona estas herramientas como **opciones disponibles**. Si la fase es chica y el implementador no las necesita, no debe usarlas solo porque están listadas.

---

## 2. sdd-explore — Exploración Acotada del Repositorio {#explore}

### Cuándo el implementador debería spawnearlo

- Necesita entender un módulo del repo que el orquestador no le pasó.
- Quiere mapear qué archivos consumen un símbolo, una función o un endpoint.
- Necesita encontrar el patrón existente para no inventar uno nuevo.
- Detecta arqueología necesaria: por ejemplo, "¿dónde se valida X actualmente?".

### Cuándo NO conviene

- La fase es chica y el contexto recibido alcanza.
- Lo que necesita explorar ya está en los archivos permitidos de la fase.
- La exploración tomaría más tiempo que leer directamente los archivos.
- Está buscando algo que el orquestador puede responder con `mem_get_observation`.

### Tips de eficiencia para la invocación

El implementador, al spawnear sdd-explore, debe darle un objetivo concreto y verificable, no una tarea abierta.

| Mal | Bien |
|---|---|
| "Explorá el repositorio" | "Encontrá dónde se valida el formato del CUIT y devolveme las rutas exactas con líneas" |
| "Mirá cómo funciona el módulo de auth" | "Listá los archivos en `src/auth/` con un resumen de 1-2 líneas por archivo" |
| "Buscá ejemplos de formularios" | "Devolveme los 3 formularios más representativos del repo y el patrón común de validación" |

### Plantilla recomendada para invocar sdd-explore

```
Objetivo: {qué tengo que descubrir, en una frase verificable}
Alcance: {directorios/módulos donde buscar — limitar siempre}
Entregable esperado:
  - {ítem 1: ej. lista de rutas y números de línea}
  - {ítem 2: ej. resumen del patrón encontrado}
Formato: {markdown / lista plana / tabla}
Profundidad: {superficial / mediana / profunda} y por qué
NO:
  - no abras más de N archivos sin foco;
  - no propongas refactors;
  - no resumas archivos no relacionados con el objetivo.
```

### Después de recibir el resultado

El implementador valida que el output responda exactamente a lo pedido. Si vuelve con más cosas de las solicitadas, las descarta. Lo que use lo cita en `Supuestos usados` al cerrar la fase, para que el orquestador sepa qué información externa influyó en la implementación.

---

## 3. sdd-archive — Documentación Persistente de Decisiones {#archive}

### Cuándo el implementador debería spawnearlo

- Tomó una decisión arquitectónica no trivial durante la implementación.
- Introdujo un patrón nuevo que otros sub-agentes deberían seguir.
- Aplicó un workaround importante que merece quedar documentado.
- Detectó deuda técnica concreta y delimitada que conviene registrar formalmente.

### Cuándo NO conviene

- Cambios obvios cubiertos por convenciones existentes.
- Decisiones que ya están en el plan o en restricciones globales (van a Engram, no a archivo).
- Comentarios menores o internos a una función — eso va en el código directamente.

### Diferencia con Engram

| Engram | sdd-archive |
|---|---|
| Estado del plan | Decisiones del código / arquitectura |
| Vive mientras dura la sesión / proyecto | Vive en el repositorio para siempre |
| Lo escribe el orquestador | Lo dispara el implementador |
| Estructurado por fases | Estructurado por temas / ADRs |

Ambos pueden coexistir: el orquestador registra en Engram que la fase tomó una decisión X; el implementador usa sdd-archive para que esa decisión quede documentada en el repo.

### Tips de eficiencia para la invocación

El implementador, al spawnear sdd-archive, debe llegar con el contenido ya sintetizado. sdd-archive no es para pensar la decisión, es para persistirla.

### Plantilla recomendada para invocar sdd-archive

```
Tipo de registro: {ADR / nota técnica / actualización de README / patrón}
Tema: {título corto y específico}
Ubicación sugerida: {ruta exacta del archivo a crear o actualizar}
Contenido a persistir:
  Contexto: {qué situación motivó la decisión}
  Decisión: {qué se decidió}
  Razones: {por qué se decidió así}
  Alternativas descartadas: {opcional pero recomendado}
  Consecuencias: {qué implica esta decisión hacia adelante}
Estilo: {formal / breve / conversacional según convención del repo}
NO:
  - no propongas decisiones nuevas;
  - no edites archivos fuera de la ubicación indicada;
  - no resumas mal: el contenido viene cerrado.
```

### Después de recibir el resultado

El implementador menciona en `Notas para siguiente agente` qué se documentó y dónde, para que el orquestador lo registre en el cierre de fase en Engram.

---

## 4. Cómo Sugerir su Uso en una Delegación {#sugerencia}

El orquestador incluye una sección dedicada en la delegación cuando la fase pinta candidata a necesitar exploración o documentación. No es obligatoria; si la fase es trivial, se omite.

### Plantilla para incluir en la delegación

```
## Sub-agentes auxiliares disponibles

Si durante esta fase necesitás:

  - Entender código que no te pasé en el contexto, o mapear cómo funciona
    una parte del repo: podés spawnear sdd-explore con un objetivo concreto.
    Tip: dale una pregunta verificable, no una tarea abierta.

  - Documentar una decisión arquitectónica o un patrón nuevo que introduzcas:
    podés spawnear sdd-archive con el contenido ya sintetizado.
    Tip: llegá con la decisión cerrada, sdd-archive solo persiste.

No estás obligado a usarlos. Si la fase es chica y el contexto alcanza, no los necesitás.
Si los usás, mencionalo en tu reporte (Supuestos usados / Notas para siguiente agente)
para que pueda registrarlo en Engram.
```

### Cuándo conviene incluir esta sección

- Fases que tocan código existente complejo (alta probabilidad de exploración).
- Fases que probablemente generen una decisión arquitectónica.
- Fases medianas o grandes donde el implementador puede toparse con incógnitas.

### Cuándo conviene omitirla

- Fases triviales: typos, ajustes mínimos, configuración trivial.
- Fases donde el orquestador ya pasó todo el contexto cerrado y verificable.
- Fases puramente de QA o testing (su propio dominio).

---

## 5. Anti-Patrones al Usar Sub-Sub-Agentes {#anti-patrones}

- **Spawnear sdd-explore "por las dudas"**: si el implementador no tiene una pregunta concreta, no debe invocarlo.
- **Pedirle a sdd-explore que decida**: solo recolecta información; las decisiones las toma el implementador.
- **Pedirle a sdd-archive que escriba sin contenido cerrado**: termina inventando contexto. Llegar con la decisión ya sintetizada.
- **Cadena de spawns**: el implementador no debe pedirle a sdd-explore que spawnee otra cosa; eso lo hace el implementador directamente.
- **No reportar el uso al orquestador**: rompe la trazabilidad. Todo uso se menciona en el cierre.
- **Reemplazar Engram con sdd-archive**: el estado del plan vive en Engram, no en archivos del repo.
- **Reemplazar el contexto de la delegación con sdd-explore**: si el implementador necesita mucha exploración para entender la fase, eso indica que el orquestador delegó mal — debe reportarlo como falla de contexto.
