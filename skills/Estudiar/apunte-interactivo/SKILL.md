---
name: apunte-interactivo
description: Convierte material de estudio (PDFs, apuntes, presentaciones, capítulos) en un apunte interactivo autocontenido, entregado como un único archivo React/TSX. Usala cuando el usuario pida un "apunte interactivo", "clase interactiva", "convertí este PDF/apunte", o adjunte material y pida una clase o recurso para estudiar el tema. No aplica a preguntas o cuestionarios (eso es banco-preguntas-examen).
---

# Apunte interactivo

Transforma material de estudio en una **clase** interactiva: un único TSX que se renderiza en un iframe con restricciones estrictas (ver Contrato técnico).

El material recibido es solo el **temario**. El estudiante JAMÁS va a ver el original: si algo no está explicado dentro del apunte, para él no existe. No mejorás un texto, construís desde cero un apunte autocontenido.

**Meta:** que el estudiante termine pudiendo **explicar y aplicar** cada concepto por su cuenta. Leer produce reconocimiento; predecir, manipular y resolver produce saber. Diseñá para lo segundo.

**Prioridad absoluta:** el apunte tiene que renderizar. Si hay que elegir, recortá amplitud periférica, nunca correctitud ni profundidad de lo central.

## Workflow

1. **Mapear el temario.** Leé TODO el material: cada concepto, ejemplo, fórmula y **diagrama** del original. Listá los diagramas: son candidatos a SVG animado.
2. **Leer las indicaciones del usuario.** Si dio estilo, enfoque o visualizaciones deseadas para la materia, mandan. Si no, default: fuertemente interactivo, exprimiendo el navegador.
3. **Diseñar la secuencia:** orden de secciones por dependencias, y para cada concepto su representación visual y la librería que la sirve (ver `references/recetas-librerias.md`).
4. **Escribir el TSX** respetando el contrato técnico.
5. **Verificar** con `references/checklist-pedagogico.md`. Completo cuando cada ítem pasa.
6. **Guardar** en `/mnt/user-data/outputs/` con nombre descriptivo (ej: `apunte_io_unidad3_modi.tsx`).

## Estructura

- **Secuencia guiada:** cada idea se construye un paso por vez, como en clase.
- **Cada sección abre con la pregunta que responde** ("¿Cómo sabemos si la red memorizó en vez de aprender?") y **cierra con síntesis y puente** a la siguiente.
- **Dependencias primero:** ningún concepto se usa antes de definirse; todo término se define al aparecer.
- **Profundidad sobre exhaustividad:** desarrollá a fondo lo central antes que mencionar mucho.

## Pedagogía

- **Desde cero.** Cada concepto, ejemplo y fórmula desde sus fundamentos. Cada frase se sostiene sola: lo que el original da por visto, el apunte lo explica completo.
- **Ejemplo trabajado narrado:** qué representa, el razonamiento paso a paso y qué observar.
- **Ejercicio paralelo:** después de cada procedimiento, mismo método con datos distintos y verificación interactiva.
- **Predicción antes de revelado:** antes del paso siguiente, pedí predecirlo; después revelá con el porqué.
- **Recuerdo activo:** las autoevaluaciones piden recuperar y aplicar. El feedback explica por qué la correcta lo es y por qué cada distractor tienta.
- **Trampas señaladas:** donde la mayoría se confunde, decilo y mostrá el error común con su corrección.
- **Tono de adulto capaz:** cálido y directo, honesto con lo difícil ("esto le cuesta a la mayoría"), riguroso sin adornos.

## Visuales

La visual es la forma de enseñar, no decoración. Todo concepto abstracto, algoritmo, estructura o proceso lleva una representación interactiva o animada.

- **Fidelidad:** modela el mecanismo real (cómo cambia la variable, cómo itera el algoritmo, cómo fluyen los datos). Si la sacás y no se pierde comprensión, sobra.
- **Una relación por visual:** un paso o una comparación por vez. El mecanismo entero animado de entrada es la respuesta disfrazada.
- **Una consigna por interactivo:** el slider viene con una pregunta que pide predecir ("¿qué le pasa a la curva si b crece? Ahora probalo").
- **Estado inicial con sentido:** antes de tocar nada, ya se ve algo que enseña.

### Diagramas SVG animados

Buscá activamente lo que se explica mejor dibujado y hacelo en **SVG animado, bonito y cuidado**: trazos que se dibujan, flujos que recorren un grafo, nodos que aparecen en orden, estados que transicionan.

- **Rescatá los diagramas del original.** Cada diagrama del material es una oportunidad: rehacelo en SVG y llevalo más lejos — animado paso a paso, con controles para avanzar, o manipulable. Nunca lo describas en prosa.
- **La animación narra el mecanismo:** el orden y el ritmo siguen la lógica del concepto, con controles de play/pausa/paso cuando hay más de un paso.
- Recetas en `references/recetas-librerias.md` → *SVG animado*.

### Interfaz sobria

El layout explica. Cada etiqueta, badge, subtítulo o epígrafe tiene que aportar algo que la disposición no dice; si repite el título o rotula lo obvio ("Interactivo", "Concepto clave", "Sección 3"), sacalo. El texto del apunte es para enseñar, no para rotular la interfaz.

## Contrato técnico (sin excepción)

El TSX corre en el iframe de una plataforma externa, tipo artifact pero con su propia whitelist.

- **Un único TSX** con `export default` del componente principal, sin boilerplate de ReactDOM/createRoot.
- **Imports SOLO de:** `react`, `recharts`, `lucide-react`, `framer-motion`, `katex` (su CSS ya está cargado), `d3`, `mathjs`. Cualquier otro import, `import()` dinámico, `require()` o `<script>` por CDN rechaza el apunte. Lo que falta se cubre con las permitidas o SVG/canvas a mano.
- **Tailwind v4** en `className`; inline solo para valores calculados en JS.
- **Fórmulas siempre en LaTeX** con katex.
- **Responsivo:** desktop y teléfono 9:16.
- **Solo para el estudiante:** ninguna frase, referencia ni directiva del pedido del usuario dentro del apunte.
