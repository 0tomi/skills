# Recetas de librerías del entorno

Patrones y gotchas para las 7 librerías importables. Regla previa: antes de sumar una librería, evaluá si SVG/Tailwind/canvas a mano lo resuelve igual de bien — menos dependencias, menos fallas.

## katex (fórmulas)

El CSS ya está cargado por la plataforma: **solo importar la librería**, nunca intentar inyectar el `<link>`.

```tsx
import katex from 'katex';

function F({ tex, block = false }: { tex: string; block?: boolean }) {
  const html = useMemo(
    () => katex.renderToString(tex, { displayMode: block, throwOnError: false }),
    [tex, block]
  );
  return <span dangerouslySetInnerHTML={{ __html: html }} />;
}
```

Gotchas: barras duplicadas en strings TSX (`"\\frac{a}{b}"`); `throwOnError: false` siempre; `useMemo` para no re-renderizar fórmulas estáticas; para fórmulas que cambian con sliders, interpolar el valor en el string tex (`\`f(${x.toFixed(1)})\``).

## recharts (gráficos de datos)

```tsx
import { LineChart, Line, XAxis, YAxis, Tooltip, CartesianGrid, ResponsiveContainer } from 'recharts';

<div className="h-64 w-full">
  <ResponsiveContainer width="100%" height="100%">
    <LineChart data={datos}>...</LineChart>
  </ResponsiveContainer>
</div>
```

Gotchas críticos:
- `ResponsiveContainer` necesita un padre con **altura explícita** (`h-64`, `h-[40vh]`); con padre de altura auto el gráfico mide 0px y no se ve nada. Es el bug #1.
- `data` es array de objetos planos `{ x: 1, y: 2.5 }`; recalcular con `useMemo` dependiente de los sliders — la interacción "slider → curva en vivo" sale gratis.
- `isAnimationActive={false}` en las series cuando el slider actualiza rápido, si no el gráfico "persigue" los datos con lag.
- En 9:16 achicar fuentes de ejes (`tick={{ fontSize: 10 }}`) y márgenes.

## mathjs (motor de cálculo para interactividad)

```tsx
import { evaluate, derivative, compile } from 'mathjs';

const expr = useMemo(() => compile('a*x^2 + b*x + c'), []);
const datos = useMemo(
  () => xs.map(x => ({ x, y: expr.evaluate({ a, b, c, x }) })),
  [a, b, c]
);
```

Gotchas: `compile()` una vez + `evaluate(scope)` en el loop (evaluar el string en cada punto es lento); `derivative('x^2','x').toTex()` se conecta directo con katex para mostrar la derivada simbólica; envolver `evaluate` de input del usuario en try/catch.

## framer-motion (revelado progresivo y transiciones)

La herramienta para el "formato presentación": ideas que aparecen paso a paso.

```tsx
import { motion, AnimatePresence } from 'framer-motion';

// revelar al scrollear
<motion.div initial={{ opacity: 0, y: 24 }} whileInView={{ opacity: 1, y: 0 }}
  viewport={{ once: true, amount: 0.3 }} transition={{ duration: 0.5 }}>

// pasos de un algoritmo (montar/desmontar con animación)
<AnimatePresence mode="wait">
  <motion.div key={paso} initial={{ opacity: 0, x: 20 }}
    animate={{ opacity: 1, x: 0 }} exit={{ opacity: 0, x: -20 }}>
    {contenido[paso]}
  </motion.div>
</AnimatePresence>
```

Gotchas: `AnimatePresence` exige `key` distinta por elemento; `layout` en motion.div anima reordenamientos gratis (útil para visualizar sorting); para resaltar "qué cambió en esta iteración", animar `backgroundColor` del elemento activo. Reemplaza todo lo que antes hacías con GSAP/anime.js.

## SVG animado (diagramas que se arman ante el ojo)

SVG en JSX + framer-motion. Para rehacer diagramas del original (flujos, arquitecturas, ciclos, máquinas de estado, circuitos) y llevarlos un paso más allá.

```tsx
const [paso, setPaso] = useState(0);

<svg viewBox="0 0 400 200" className="w-full h-auto">
  {/* trazo que se dibuja */}
  <motion.path d="M40 100 C 140 20, 260 180, 360 100" fill="none"
    className="stroke-indigo-500" strokeWidth={3} strokeLinecap="round"
    initial={{ pathLength: 0 }} animate={{ pathLength: paso >= 1 ? 1 : 0 }}
    transition={{ duration: 1.2, ease: 'easeInOut' }} />
  {/* nodos que aparecen en orden */}
  {nodos.map((n, i) => (
    <motion.g key={n.id} initial={{ opacity: 0, scale: 0.6 }}
      animate={{ opacity: paso >= n.paso ? 1 : 0.15, scale: paso >= n.paso ? 1 : 0.6 }}
      style={{ transformOrigin: `${n.x}px ${n.y}px` }}>
      <circle cx={n.x} cy={n.y} r={18} className="fill-white stroke-slate-700" />
      <text x={n.x} y={n.y + 4} textAnchor="middle" className="text-xs fill-slate-800">{n.label}</text>
    </motion.g>
  ))}
</svg>
```

- **Dibujar trazos:** `pathLength` 0→1 en `motion.path`/`motion.line`.
- **Partícula que recorre un camino** (dato fluyendo, token, paquete): `motion.circle` con `style={{ offsetPath: \`path('${d}')\` }}` y `animate={{ offsetDistance: ['0%', '100%'] }}`, `repeat: Infinity`.
- **Estado activo:** animar `fill`/`stroke` del nodo o arista que cambió en este paso; el resto en baja opacidad.
- **Controles:** un `paso` en estado con botones play/pausa/anterior/siguiente (lucide) — el estudiante marca el ritmo. Autoplay solo para loops cortos e ilustrativos.
- **Responsivo:** siempre `viewBox` + `w-full h-auto`, nunca width/height fijos en px.
- **Bonito:** trazos redondeados, paleta acotada (2–3 colores con significado), transiciones `easeInOut` de 0.4–1.2 s, espacio en blanco generoso.

## d3 (lo que recharts no cubre: grafos, árboles, fuerzas)

Patrón preferido — **d3 calcula, React renderiza**: usar d3 solo para escalas, layouts y generadores de path, y dibujar el SVG con JSX. Cero peleas por el DOM.

```tsx
import * as d3 from 'd3';

const x = d3.scaleLinear().domain([0, 10]).range([40, ancho - 10]);
const linea = d3.line<Punto>().x(p => x(p.x)).y(p => y(p.y));
// en JSX: <path d={linea(datos) ?? ''} className="stroke-blue-600 fill-none" />
```

Para layouts: `d3.hierarchy` + `d3.tree()` da posiciones de nodos de un árbol (B-tree, árbol de decisión, Branch and Bound) y los nodos se dibujan como `<circle>`/`<text>` en JSX — esto reemplaza a Mermaid con más control. Excepción al patrón: `d3.forceSimulation` (grafos con física) sí muta posiciones por tick; ahí usar un ref al `<g>` o copiar posiciones a estado en cada tick, y **detener la simulación en el cleanup** del useEffect.

## lucide-react (íconos)

```tsx
import { BookOpen, ChevronRight, Play, RotateCcw } from 'lucide-react';
<Play className="w-4 h-4" />
```

Gotcha único: importar solo los íconos usados, con nombre exacto (PascalCase). Útiles para apuntes: Play/Pause/RotateCcw (controles de simulación), ChevronRight/ChevronDown (acordeones), Lightbulb/AlertTriangle (cajas de idea/ojo), Check/X (autoevaluación).

## Tailwind v4 (estilos)

Clases de utilidad en `className` como forma principal. Valores dinámicos calculados en JS van como style inline (`style={{ width: `${pct}%` }}`), no como clases armadas por concatenación. Para responsive 9:16: mobile-first (`flex-col md:flex-row`, `text-sm md:text-base`, `overflow-x-auto` en tablas y tableaux).

## Equivalencias de lo que NO está disponible

- **Mermaid (diagramas de flujo/árboles)** → SVG animado propio (ver arriba), con d3.hierarchy si hay layout de árbol.
- **Chart.js** → recharts (o d3 si es algo exótico).
- **p5.js (simulaciones/física)** → `<canvas>` + `useRef` + `requestAnimationFrame` en un useEffect (cancelar en cleanup), o SVG animado si son pocos elementos. Estado de sliders se lee desde un ref para no congelar el closure.
- **GSAP / anime.js** → framer-motion.
- **Plotly (3D/científico pesado)** → repensar la visualización en 2D con d3/recharts; el entorno no tiene 3D.
