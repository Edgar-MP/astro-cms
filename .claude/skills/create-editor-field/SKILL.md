---
name: create-editor-field
description: Use when adding a new field TYPE to the visual editor's schema-driven form (e.g. a color picker, a map field, a new kind of input) — i.e. extending what the FieldRenderer can render. This changes the field contract in @cms/shared. Do NOT use for creating content blocks (use create-block).
---

# Añadir un tipo de campo al editor

Los formularios del editor se generan desde el esquema del bloque. Añadir un tipo de campo toca el contrato y el renderizador. Detalle en `anexo-arquitectura.md` (A.8) y `plan-cms-bloques.md` sec. 8.3.

## Pasos

1. **Extiende el contrato** en `@cms/shared`:
   - Añade el nuevo tipo a la unión `FieldType` y su interfaz (opciones que admita).
   - Añade su validador en `zod/schema.zod.ts`.
   - ⚠️ Esto afecta a las tres piezas; coordina y comunica el cambio.

2. **Crea el componente del campo** en `packages/editor/src/features/field-forms/fields/<Tipo>Field.tsx`:
   - Recibe el valor y un `onChange` que emite la mutación `updateField`.
   - Respeta `showIf` (el campo puede estar oculto según otra condición).
   - Accesible y navegable por teclado.

3. **Regístralo** en `FieldRenderer` (el mapa `type → componente`).

4. **Soporta el render** en el frontend si el tipo implica algo nuevo en el `.astro` (p. ej. un nuevo formato de valor).

5. **Valídalo en la API** (el `data` que llega debe pasar la validación Zod del nuevo tipo).

## Criterios de aceptación
- El tipo se valida en `@cms/shared` y en la API.
- El campo se pinta, edita y emite `updateField`.
- Respeta `showIf` y es accesible.
- Si es traducible, se comporta bien por locale.

## Nunca
- No metas lógica de un campo concreto dentro del `FieldRenderer`; cada tipo es su propio componente.
- No añadas un tipo sin su validador Zod.
