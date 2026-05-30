# Design System

> El sistema de diseño del CMS y de las webs que genera. Define los **tokens** (valores con nombre), las **bases** (cómo se usan) y las **convenciones** para construir bloques coherentes. Léelo antes de crear o estilar cualquier bloque o componente.

## Archivos

- **`README.md`** (este): visión y principios.
- **`tokens.md`**: los design tokens con sus valores y nombres de variable CSS.
- **`components.md`**: convenciones para construir bloques con el sistema (variantes, estados, accesibilidad, do/don't).

## Principios

1. **Tokens, no valores libres.** Nunca escribas `padding: 23px` ni `color: #3b82f6` en un bloque. Usa `var(--space-md)`, `var(--color-primary)`. Esto es lo que permite el theming y mantiene la coherencia (plan sec. 9.bis).
2. **Coherencia sobre creatividad puntual.** Un "hero" se ve y se comporta igual en todas las webs salvo que su variante diga lo contrario.
3. **Accesible por defecto.** Contraste suficiente, foco visible, HTML semántico. Objetivo WCAG 2.1 AA.
4. **Responsive por escala.** Tamaños y espaciados salen de escalas definidas, no de números sueltos.
5. **Variantes, no duplicados.** Un bloque con varias presentaciones usa un campo `variant`, no bloques separados.

## Cómo se aplica técnicamente

Los tokens son **variables CSS** (`:root { --color-primary: …; }`). El frontend Astro las inyecta en el `<head>`; si el theme editor está activo, sus valores vienen de `theme_settings` por tenant (plan sec. 9.bis). Los componentes y bloques solo consumen variables; nunca definen valores absolutos.

Un subconjunto **curado** de tokens puede exponerse como ajustes editables desde el CMS (color de marca, fuente, densidad). El resto es fijo, para que el sistema no se pueda romper desde el editor.
