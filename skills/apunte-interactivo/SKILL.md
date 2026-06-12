---
name: apunte-interactivo
description: Convierte material de estudio (PDFs, apuntes, presentaciones, capítulos) en un apunte interactivo autocontenido, entregado como un ÚNICO archivo React/TSX listo para renderizar en un iframe. Usala siempre que el usuario pida un "apunte interactivo", "clase interactiva", "apunte visual", "convertí este PDF/apunte", "armame el apunte de esto", o adjunte material de estudio y pida una clase, una página o un recurso para estudiar el tema, incluso si no dice explícitamente "interactivo". No aplica si pide preguntas o cuestionarios (para eso está banco-preguntas-examen).
---

# Apunte interactivo

Transforma material de estudio en una clase didáctica, visual e interactiva, exportada como un único TSX que se renderiza en un iframe con restricciones estrictas (ver Contrato técnico). El material recibido es solo el **temario**: la lista de temas que el apunte debe cubrir.

**El dato más importante:** el estudiante JAMÁS va a ver el material original. El apunte lo reemplaza por completo; si algo no está explicado adentro, para el estudiante no existe. No estás mejorando un texto: estás construyendo desde cero un apunte autocontenido y definitivo.

**Meta pedagógica:** que el estudiante termine **pudiendo explicar y aplicar cada concepto por su cuenta**. Un apunte que solo se lee produce un estudiante que reconoce el tema; uno que hace predecir, manipular y resolver produce un estudiante que lo sabe. Diseñá para lo segundo.

## Personalización por materia

Cada materia tiene sus peculiaridades, y a veces el usuario ya tiene en mente cómo quiere el apunte. Antes de diseñar, revisá el mensaje del usuario:

- **Si incluye indicaciones** de estilo, enfoque, estructura, qué profundizar o qué tipo de visualizaciones quiere → esas indicaciones **mandan** y guían el diseño.
- **Si no incluye nada**, aplicá el default: estilo fuertemente interactivo, exprimiendo lo que da el navegador — diagramas, gráficos en vivo, simulaciones manipulables, algoritmos paso a paso, experimentos.

En ambos casos las reglas pedagógicas y el contrato técnico se cumplen igual.

## Cómo estructurar el contenido

- **Secuencia guiada, no muro de texto.** El apunte avanza como una clase: cada idea se revela y se construye progresivamente, un paso por vez.
- **Cada sección abre con la pregunta que va a responder** o el problema que motiva el concepto ("¿Cómo sabemos si la red memorizó en vez de aprender?"). El estudiante sabe para qué está leyendo antes de leer.
- **Cada sección cierra con síntesis breve y puente** a la siguiente: qué quedó establecido y qué pregunta nueva abre.
- **Orden de dependencias:** ningún concepto se usa antes de definirse; todo término técnico se define la primera vez que aparece.
- **Cobertura total del temario** con profundidad sobre exhaustividad: desarrollá a fondo lo central antes que mencionar mucho superficialmente.

## Directivas pedagógicas (el corazón del apunte)

- **Explicá desde cero.** Cada concepto, ejemplo y fórmula desde sus fundamentos, como si el estudiante no supiera nada del tema. No tiene otro material al lado.
- **Ejemplo trabajado con narración.** Cada ejemplo viene completo: qué representa, por qué es así, el razonamiento narrado paso a paso, y qué debe observar el estudiante. Mostrar un ejemplo y seguir de largo no enseña.
- **Después del ejemplo trabajado, un caso para que lo resuelva él.** Donde haya un procedimiento (algoritmo, método, cálculo), cerrá con un ejercicio paralelo — mismo método, datos distintos — con verificación interactiva. Aplicar el método es donde el aprendizaje se cementa.
- **Predicción antes de revelado.** En las construcciones paso a paso, antes de mostrar el paso siguiente invitá a predecirlo ("¿qué celda conviene elegir ahora? Pensalo antes de seguir") y recién entonces revelá con la explicación de por qué.
- **Recuerdo activo en las autoevaluaciones.** Preguntas que obligan a recuperar y aplicar, no a reconocer. El feedback es inmediato y explica por qué la correcta es correcta y por qué cada distractor es tentador pero incorrecto.
- **Señalá las trampas.** Donde los estudiantes típicamente se confunden, decilo explícitamente ("este paso confunde a la mayoría: el error común es...") y mostrá el error y su corrección.

## Interactividad y mandato visual (no es decoración, es la forma de enseñar)

- Todo concepto abstracto, ecuación, algoritmo, estructura o proceso DEBE tener una representación visual interactiva o animada que lo modele.
- **Fidelidad sobre adorno:** la visualización representa el mecanismo real (cómo cambia una variable, cómo itera el algoritmo, cómo se deforma la curva, cómo fluyen los datos). Una visual que no carga el concepto es decoración, y la decoración enseña a pasar de largo.
- **Una relación por visual.** Mostrá un paso, una relación, una comparación por vez — no el mecanismo completo terminado de una. La animación de todo el mecanismo es la respuesta disfrazada: saltea el pensamiento del estudiante.
- **Cada interactivo lleva una pregunta, no un epígrafe.** El slider o la simulación van con una consigna que pide predecir o explicar ("antes de mover el slider: ¿qué le pasa a la curva si b crece? Ahora probalo"). La mano del estudiante en el parámetro vale más que la tuya.
- Interacciones típicas: sliders que recalculan fórmula y gráfico en vivo; algoritmos paso a paso resaltando qué cambió y por qué en cada iteración; diagramas que se arman ante el ojo; simulaciones manipulables; autoevaluaciones con feedback inmediato.

## Tono

Tratá al estudiante como un adulto capaz trabajando en algo difícil: cálido, directo, sin infantilizar. Sin cheerleading ni entusiasmo vacío — cuando algo es difícil, decilo: "esto le cuesta a la mayoría" enseña más que "¡es facilísimo!". Sencillo sin perder rigor, sin dejar detalles afuera.

## Anti-patrones (qué evitar)

- **Comentar en vez de enseñar.** Prohibidas las frases que solo tienen sentido con el original al lado: "como vimos", "según el apunte", "el texto menciona", "el ejemplo anterior" (refiriendo al material), "recordemos que". Si una frase remite a material externo, reescribila explicando la idea completa.
- **Ejemplos sueltos** sin desarrollo paso a paso.
- **Visuales que sobreentregan:** el mecanismo entero animado como primera exposición, o una visual por párrafo que no carga concepto.
- **Quices de reconocimiento:** preguntas cuya respuesta es literalmente una frase que aparece dos pantallas arriba.
- **Meta-contenido:** ninguna frase, referencia ni directiva del pedido del usuario dentro del apunte; se redacta exclusivamente para el estudiante.

## Contrato técnico (cumplir SIN EXCEPCIÓN)

El TSX se renderiza en el iframe de una plataforma externa con un entorno tipo artifact: imports directos de una whitelist cerrada. NO es exactamente el entorno de artifacts de Claude.ai (la lista de librerías difiere).

- **Un único archivo TSX** con `export default` del componente principal. Sin boilerplate de ReactDOM/createRoot: lo maneja la plataforma.
- **Imports permitidos ÚNICAMENTE** (cualquier otro import rechaza el apunte):
  - `react` — hooks (useState, useEffect, useRef, useMemo, useCallback, etc.)
  - `recharts` — gráficos de datos (LineChart, BarChart, AreaChart, PieChart, ScatterChart y sus componentes)
  - `lucide-react` — íconos SVG
  - `framer-motion` — animaciones declarativas (motion.div, AnimatePresence, variants)
  - `katex` — fórmulas LaTeX vía `katex.renderToString(...)`; **el CSS de KaTeX ya está cargado**, no intentar cargarlo
  - `d3` — visualizaciones científicas avanzadas (grafos, árboles, fuerzas, heatmaps)
  - `mathjs` — `evaluate`, `derivative`, `parse` para recalcular fórmulas en vivo
- **Prohibido:** `import()` dinámico, `require()`, y `document.createElement('script')` para cargar librerías por CDN. Si una librería no está en la lista (Chart.js, Mermaid, p5, GSAP, Plotly…), NO existe: su rol se cubre con las permitidas o con SVG/canvas a mano (ver recetas).
- **Estilos con Tailwind CSS v4** directamente en `className` (forma preferida), complementando con estilos inline solo para valores dinámicos calculados en JS.
- **Fórmulas matemáticas siempre en LaTeX renderizado** con katex, nunca texto plano tipo `x^2`.
- **Diseño responsivo:** debe verse bien en desktop y en pantalla de teléfono 9:16.
- **Nada de meta-contenido:** el apunte se redacta para el estudiante; no incluir frases, referencias ni directivas del pedido del usuario dentro del apunte generado.

Antes de escribir visualizaciones, **consultá `references/recetas-librerias.md`**: patrones y gotchas de cada librería permitida, y con qué reemplazar las que no están.

## Prioridad absoluta

**El apunte tiene que FUNCIONAR y renderizar de verdad.** Un artefacto que corre y explica bien lo central vale más que uno gigantesco que se trunca o no abre. Si hay que elegir, garantizá primero que funcione: recortá amplitud, nunca correctitud ni profundidad de lo central.

## Workflow

1. **Leer TODO el material** y mapear el temario: idea general + cada concepto, ejemplo y fórmula.
2. **Detectar indicaciones del usuario** (estilo, enfoque, visualizaciones deseadas) y dejar que guíen el diseño si existen.
3. **Diseñar la secuencia:** orden progresivo de secciones, y para cada concepto decidir su representación interactiva y qué librería la sirve mejor.
4. **Escribir el TSX** respetando el contrato técnico, usando `references/recetas-librerias.md` como guía de patrones por librería.
5. **Verificar** contra `references/checklist-pedagogico.md` antes de entregar.
6. **Guardar** como `.tsx` en `/mnt/user-data/outputs/` con nombre descriptivo (ej: `apunte_io_unidad3_modi.tsx`).

## Referencias auxiliares

No se cargan por defecto. Consultalas cuando corresponda:

- `references/recetas-librerias.md` — patrones y gotchas de las librerías permitidas (recharts, katex, d3 + React, framer-motion, mathjs) y cómo reemplazar las que no están (Mermaid, Chart.js, p5, GSAP).
- `references/checklist-pedagogico.md` — verificación final de autocontención, cobertura, visualizaciones y contrato técnico.
