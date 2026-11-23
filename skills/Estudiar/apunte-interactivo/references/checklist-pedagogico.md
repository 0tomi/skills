# Checklist pedagógico

Verificación final antes de guardar. Si algo falla, corregilo.

## Autocontención

- [ ] Cero frases que remitan al material externo: buscá "como vimos", "según el apunte", "el texto", "el material", "recordemos que".
- [ ] Alguien que nunca vio el original lo sigue de punta a punta.

## Cobertura

- [ ] Cada concepto mapeado en el paso 1 está **desarrollado**, no apenas mencionado.
- [ ] Cada diagrama del original quedó rehecho como SVG animado o interactivo (o hay un motivo claro para no hacerlo).
- [ ] Lo recortado por tamaño es periférico, nunca central.

## Aprendizaje activo

- [ ] Secciones abren con pregunta y cierran con síntesis + puente.
- [ ] Todo procedimiento tiene ejemplo trabajado narrado Y ejercicio paralelo con verificación.
- [ ] Construcciones paso a paso piden predecir antes de revelar.
- [ ] Autoevaluaciones de recuerdo activo, con feedback por distractor.
- [ ] Trampas típicas señaladas con error y corrección.
- [ ] Ejemplos numéricos verificados a mano.

## Visuales e interfaz

- [ ] Cada visual modela el mecanismo real, una relación por vez, con consigna de predicción y estado inicial con sentido.
- [ ] Las animaciones SVG siguen el orden lógico del concepto y tienen controles cuando hay varios pasos.
- [ ] Sin etiquetas, badges ni epígrafes que repitan lo que el layout ya dice.

## Contrato técnico

- [ ] Un único TSX con `export default`; imports solo de la whitelist.
- [ ] Todo `ResponsiveContainer` tiene padre con altura explícita.
- [ ] Cleanup en `useEffect` (rAF, simulaciones d3, timers).
- [ ] En 9:16 nada desborda: sin anchos fijos, tablas con `overflow-x-auto`, SVG con `viewBox` y `w-full`.
- [ ] Indicaciones del usuario respetadas y ausentes del texto del apunte.
- [ ] Tamaño sano: si corre riesgo de truncarse, recortá amplitud antes de entregar.
