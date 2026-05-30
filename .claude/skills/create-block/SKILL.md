---
name: create-block
description: Use when creating a new content block for the CMS (a "hero", "testimonial", "gallery", etc.) — anything that becomes a selectable block in the visual editor. Covers the Astro component, its schema, variants, conditional fields, registration and schema sync. Do NOT use for editor field types (use create-editor-field) or API endpoints (use create-api-endpoint).
---

# Crear un bloque de contenido

Un bloque = una carpeta con un componente Astro + su esquema. Sigue estos pasos. Detalle ampliado en `guia-crear-bloque.md`.

## Decisión previa
- ¿Es genérico y reutilizable por varias webs? → va en `packages/blocks-base`.
- ¿Es específico de una web (o sobrescribe uno base)? → va en `src/blocks/` de esa web.

## Pasos

1. **Crea la carpeta** `<destino>/blocks/<name>/` con dos archivos:
   - `<Name>.astro` (la vista)
   - `<name>.schema.ts` (el contrato)

2. **Escribe el esquema** importando `BlockSchema` de `@cms/shared`:
   - `name` único y estable (no lo cambies luego: el contenido lo referencia).
   - Marca `translatable: true` todo campo con texto visible.
   - Campos de estilo (padding, ancho, color) → tipo `token`, nunca valores libres.
   - Si tiene variaciones visuales → campo `variant` + `showIf` en los campos que dependan de la variante.

3. **Escribe el componente** `.astro`:
   - Lee props con valores por defecto (render tolerante: nunca asumas que un campo existe).
   - Aplica estilos **solo con design tokens** (variables CSS). Cero píxeles/colores fijos.
   - HTML semántico y accesible (`alt`, encabezados en orden, `aria` donde toque).
   - Si hay `variant`, decide layout/clase según su valor.

4. **Regístralo** en `registry.ts`: añade el componente a `blockMap` y el esquema a `schemas`.

5. **Sincroniza** los esquemas: `pnpm --filter <web> push-schemas`.

## Criterios de aceptación
- El bloque aparece en el editor y se puede añadir, editar y publicar.
- Cada variante renderiza bien y los `showIf` muestran/ocultan los campos correctos.
- Estilos por tokens; HTML accesible; render no rompe ante campos ausentes.
- Probado en el preview en vivo.

## Errores comunes
- Olvidar `push-schemas` → el editor no ve el bloque.
- `name` no único o cambiado → el contenido existente pierde su componente.
- Meter estilos fijos → rompe el theming y el design system.
