# Design Tokens

> Valores con nombre del sistema. Cada token tiene un **nombre de escala** (lo que ve el editor en los campos `token`) y una **variable CSS** (lo que usan los componentes). Los valores de abajo son un punto de partida razonable; ajústalos a la marca, pero mantén los nombres.

## Cómo se usan

- En un esquema de bloque: `"padding": { "type": "token", "scale": "spacing" }` → el editor muestra `none · xs · sm · md · lg · xl · 2xl`.
- En un componente: `style={`padding: var(--space-md)`}` o una clase utilitaria que use la variable.

## Color

| Escala (editor) | Variable CSS | Valor ejemplo | Uso |
|---|---|---|---|
| primary | `--color-primary` | `#2563eb` | Acción principal, enlaces, marca. **Editable** (theme). |
| secondary | `--color-secondary` | `#7c3aed` | Acentos secundarios. **Editable**. |
| fg | `--color-fg` | `#0f172a` | Texto principal. |
| fg-muted | `--color-fg-muted` | `#475569` | Texto secundario. |
| bg | `--color-bg` | `#ffffff` | Fondo de página. |
| bg-subtle | `--color-bg-subtle` | `#f1f5f9` | Fondos de sección. |
| border | `--color-border` | `#e2e8f0` | Bordes y separadores. |
| success / warning / danger | `--color-success` … | `#16a34a` / `#d97706` / `#dc2626` | Estados. |

> Define cada color de texto/fondo de modo que cumpla **contraste AA** (4.5:1 texto normal).

## Espaciado (escala 4px)

| Escala | Variable | Valor |
|---|---|---|
| none | `--space-none` | `0` |
| xs | `--space-xs` | `4px` |
| sm | `--space-sm` | `8px` |
| md | `--space-md` | `16px` |
| lg | `--space-lg` | `24px` |
| xl | `--space-xl` | `40px` |
| 2xl | `--space-2xl` | `64px` |

## Tipografía

**Familias** (editables): `--font-body` (texto), `--font-heading` (titulares).

**Escala de tamaño:**
| Escala | Variable | Valor | Uso |
|---|---|---|---|
| xs | `--text-xs` | `0.75rem` | Pies, etiquetas. |
| sm | `--text-sm` | `0.875rem` | Texto secundario. |
| base | `--text-base` | `1rem` | Cuerpo. |
| lg | `--text-lg` | `1.25rem` | Destacados. |
| xl | `--text-xl` | `1.75rem` | H3. |
| 2xl | `--text-2xl` | `2.25rem` | H2. |
| 3xl | `--text-3xl` | `3rem` | H1 / hero. |

**Pesos:** `--weight-regular: 400`, `--weight-medium: 500`, `--weight-bold: 700`.
**Interlineado:** `--leading-tight: 1.2` (titulares), `--leading-normal: 1.6` (cuerpo).

## Radios

| Escala | Variable | Valor |
|---|---|---|
| none | `--radius-none` | `0` |
| sm | `--radius-sm` | `6px` |
| md | `--radius-md` | `12px` |
| lg | `--radius-lg` | `20px` |
| full | `--radius-full` | `9999px` |

## Sombras

| Escala | Variable | Valor |
|---|---|---|
| sm | `--shadow-sm` | `0 1px 2px rgba(0,0,0,.06)` |
| md | `--shadow-md` | `0 4px 12px rgba(0,0,0,.08)` |
| lg | `--shadow-lg` | `0 12px 32px rgba(0,0,0,.12)` |

## Contenedores (ancho máximo de contenido)

| Escala | Variable | Valor |
|---|---|---|
| narrow | `--container-narrow` | `640px` |
| default | `--container-default` | `1024px` |
| wide | `--container-wide` | `1280px` |

## Breakpoints (responsive)

| Nombre | Valor |
|---|---|
| sm | `640px` |
| md | `768px` |
| lg | `1024px` |
| xl | `1280px` |

## Z-index (capas)

| Escala | Variable | Valor |
|---|---|---|
| base | `--z-base` | `0` |
| dropdown | `--z-dropdown` | `1000` |
| overlay | `--z-overlay` | `1100` |
| modal | `--z-modal` | `1200` |
| toast | `--z-toast` | `1300` |

> ⚠️ Si necesitas un valor que no está en ninguna escala, **no lo metas a mano**: añade un token nuevo aquí primero, con nombre y justificación. Esa es la única forma de crecer el sistema sin romperlo.
