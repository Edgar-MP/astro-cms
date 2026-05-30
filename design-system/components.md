# Convenciones de componentes y bloques

> Cómo construir bloques coherentes con el sistema. Aplica a los `.astro` de bloques y a los componentes del editor.

## Estructura visual de un bloque

- Todo bloque se monta sobre un **contenedor** con un ancho máximo (`--container-*`) y un **padding vertical** por tokens de espaciado. No inventes anchos ni márgenes.
- Usa la **escala de espaciado** para todos los huecos (gaps, paddings, márgenes). Nada de píxeles sueltos.
- Los titulares usan `--font-heading` y la escala de texto; el cuerpo usa `--font-body`.

## Variantes

- Una variante cambia la **presentación**, no el contenido. Se modela con un campo `variant` (ver `guia-crear-bloque.md`).
- Nombra las variantes por intención, no por estilo concreto: `centered`, `split`, `compact`, `feature` — no `azul-grande`.
- Cada variante debe seguir cumpliendo accesibilidad y responsive.
- Si una variante necesita un campo extra, usa `showIf` para mostrarlo solo en esa variante.

## Estados interactivos

Para elementos interactivos (botones, enlaces, inputs) define siempre: `hover`, `focus-visible` (foco **visible** por teclado), `active` y `disabled`. El foco nunca se elimina sin una alternativa visible.

## Responsive

- Diseña *mobile-first*: estilos base para móvil, y `min-width` con los breakpoints para pantallas mayores.
- El texto y el espaciado pueden escalar entre breakpoints, pero siempre saltando entre **tokens**, no a valores arbitrarios.

## Accesibilidad (obligatorio)

- HTML **semántico**: un solo `<h1>` por página, encabezados en orden, listas reales, `<button>` para acciones y `<a>` para navegación.
- Imágenes con `alt` (campo traducible); imágenes decorativas con `alt=""`.
- Contraste mínimo AA (apóyate en los tokens de color, ya pensados para cumplirlo).
- Todo lo accionable, alcanzable y operable por **teclado**; `aria-*` solo cuando el HTML semántico no baste.
- Objetivo de referencia: **WCAG 2.1 AA**.

## Do / Don't

| ✅ Haz | ❌ No hagas |
|---|---|
| `padding: var(--space-lg)` | `padding: 22px` |
| `color: var(--color-primary)` | `color: #2563eb` |
| Variante con campo `variant` | Bloques `hero-azul`, `hero-rojo` |
| `<button>` para una acción | `<div onclick>` |
| Foco visible por teclado | `outline: none` sin alternativa |
| Añadir un token nuevo a `tokens.md` | Meter un valor mágico en el bloque |

## Checklist antes de dar por bueno un bloque

- [ ] Solo tokens; cero valores fijos.
- [ ] HTML semántico y accesible (teclado, contraste, `alt`).
- [ ] Responsive mobile-first con breakpoints del sistema.
- [ ] Variantes (si las hay) coherentes y con `showIf` donde toque.
- [ ] Estados `hover/focus/active/disabled` en lo interactivo.
