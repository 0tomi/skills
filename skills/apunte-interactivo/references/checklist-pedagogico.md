# Checklist pedagógico de calidad

Verificación final antes de entregar el TSX. Si algo falla, corregilo antes de guardar.

## Autocontención (lo más importante)

- [ ] Buscá en el texto del apunte frases que remitan a material externo: "como vimos", "según el apunte", "el texto", "el material", "recordemos que", "mencionado anteriormente" (cuando refiere al PDF y no a una sección del propio apunte). Cero ocurrencias.
- [ ] Un estudiante que nunca vio el material original puede seguir el apunte de punta a punta sin perderse.
- [ ] Cada término técnico se define la primera vez que aparece.

## Cobertura del temario

- [ ] Repasá la lista de conceptos mapeados en el paso 1 del workflow: TODOS están desarrollados (no apenas mencionados).
- [ ] Si hubo que recortar por tamaño, lo recortado es amplitud periférica, nunca un concepto central del temario.

## Estructura y aprendizaje activo

- [ ] Cada sección abre con la pregunta o problema que motiva el concepto, y cierra con síntesis + puente a la siguiente.
- [ ] Todo procedimiento tiene su ejemplo trabajado con narración Y un ejercicio paralelo (mismo método, datos distintos) con verificación interactiva.
- [ ] Las construcciones paso a paso invitan a predecir antes de revelar el paso siguiente.
- [ ] Las autoevaluaciones son de recuerdo activo (recuperar/aplicar, no reconocer una frase de dos pantallas arriba) y el feedback explica por qué cada distractor está mal.
- [ ] Las trampas típicas del tema están señaladas explícitamente con el error común y su corrección.
- [ ] Tono de adulto capaz: sin cheerleading, sin "¡es facilísimo!", honestidad sobre lo difícil.

## Ejemplos y fórmulas

- [ ] Ningún ejemplo "suelto": todos tienen qué representa, desarrollo paso a paso y qué observar.
- [ ] Toda fórmula está en LaTeX renderizado (KaTeX/MathJax), con sus variables explicadas la primera vez.
- [ ] Los ejemplos numéricos cierran: verificá las cuentas a mano antes de entregar.

## Visualizaciones

- [ ] Cada concepto abstracto, algoritmo o proceso tiene su representación visual interactiva o animada.
- [ ] Cada visualización modela el mecanismo REAL (no decora): si la sacás, se pierde comprensión.
- [ ] Una relación/paso/comparación por visual; ninguna animación entrega el mecanismo completo de una como primera exposición.
- [ ] Cada interactivo lleva una consigna de predicción o explicación, no un epígrafe pasivo.
- [ ] Las interacciones tienen estado inicial razonable: el estudiante ve algo con sentido antes de tocar nada.

## Indicaciones del usuario

- [ ] Si el usuario dio indicaciones de estilo/enfoque para esta materia, el apunte las respeta.
- [ ] Ninguna directiva, frase ni referencia del pedido del usuario quedó escrita dentro del apunte.

## Contrato técnico

- [ ] Un único archivo TSX con `export default`, sin boilerplate de ReactDOM/createRoot.
- [ ] Imports SOLO de la whitelist: react, recharts, lucide-react, framer-motion, katex, d3, mathjs. Cero `import()` dinámico, `require()` o inyección de `<script>`/`<link>` por CDN.
- [ ] No se intenta cargar el CSS de KaTeX (ya está en la plataforma).
- [ ] Estilos con Tailwind en `className`; inline solo para valores dinámicos calculados en JS (sin clases armadas por concatenación de strings).
- [ ] Todo `ResponsiveContainer` de recharts tiene padre con altura explícita.
- [ ] Cleanup correcto en `useEffect` (requestAnimationFrame cancelado, simulaciones d3 detenidas, timers limpiados).
- [ ] Probado mentalmente en 9:16: nada con ancho fijo que desborde, tablas/tableaux con `overflow-x-auto`, tipografía legible en pantalla chica.
- [ ] Tamaño sano: si el archivo creció tanto que corre riesgo de truncarse, recortar amplitud ANTES de entregar. Funciona > exhaustivo.
