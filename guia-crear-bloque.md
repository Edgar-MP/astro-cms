# Guía — Cómo crear un bloque nuevo

> La tarea más repetida del proyecto. Un bloque = un **componente Astro** + su **esquema**. Aquí tienes el paso a paso, desde lo más simple hasta variantes y campos condicionales. Pensada para alguien que ya tiene el entorno corriendo (ver `guia-setup-local.md`).

---

## 1. ¿Dónde vive un bloque?

Un bloque puede vivir en dos sitios (ver Anexo A.6.5):
- **`@cms/blocks-base`** — si es genérico y reutilizable por varias webs (hero, texto, imagen…).
- **`src/blocks/` de tu web** — si es específico de esa web, o si quieres **sobrescribir** uno base.

Cada bloque es **una carpeta** con dos archivos juntos:
```
src/blocks/testimonial/
  Testimonial.astro      # cómo se ve (la implementación)
  testimonial.schema.ts  # qué campos tiene (el contrato)
```

---

## 2. Paso a paso: un bloque simple

Vamos a crear un bloque "testimonial" (una cita con autor).

### Paso 1 — Crea la carpeta y el esquema
`src/blocks/testimonial/testimonial.schema.ts`
```ts
import type { BlockSchema } from "@cms/shared";

export const schema: BlockSchema = {
  name: "testimonial",            // identifica el bloque (único)
  display_name: "Testimonio",     // lo que ve el editor
  fields: {
    quote:  { type: "richtext", translatable: true },
    author: { type: "text",     translatable: true },
    avatar: { type: "asset" },
  },
};
```

### Paso 2 — Crea el componente
`src/blocks/testimonial/Testimonial.astro`
```astro
---
const { quote, author, avatar } = Astro.props;
---
<figure class="testimonial">
  <blockquote set:html={quote} />
  <figcaption>
    {avatar && <img src={avatar.url} alt={avatar.alt} width="48" height="48" />}
    <span>{author}</span>
  </figcaption>
</figure>
```
> ⚠️ Usa siempre los **design tokens** (variables CSS) para estilos; nada de píxeles "a mano" (ver plan sec. 9.bis).

### Paso 3 — Regístralo
En `src/blocks/registry.ts`, añade el bloque al mapa y a los esquemas:
```ts
import Testimonial from "./testimonial/Testimonial.astro";
import { schema as testimonialSchema } from "./testimonial/testimonial.schema";

export const blockMap = { ...base.blockMap, testimonial: Testimonial };
export const schemas  = { ...base.schemas,  testimonial: testimonialSchema };
```

### Paso 4 — Sincroniza el esquema con el editor
```bash
pnpm push-schemas
```
Esto sube el esquema a la API. Ahora el editor sabe que existe el bloque "Testimonio" y puede pintar su formulario.

### Paso 5 — Pruébalo
Abre el editor, añade un bloque "Testimonio" a una página, rellénalo y comprueba el preview. ✅

---

## 3. Añadir variantes (el mismo bloque, otra presentación)

Una **variante** es el mismo bloque mostrado de otra forma (plan sec. 6.4). Se añade con un campo `variant`, y el `.astro` decide el layout.

### En el esquema
```ts
fields: {
  variant: { type: "variant", options: ["card", "banner", "minimal"] },
  quote:   { type: "richtext", translatable: true },
  author:  { type: "text",     translatable: true },
  avatar:  { type: "asset" },
},
```

### En el componente
```astro
---
const { variant = "card", quote, author, avatar } = Astro.props;
---
<figure class={`testimonial testimonial--${variant}`}>
  <!-- el mismo contenido; el CSS cambia según la variante -->
</figure>
```
En el editor, `variant` aparece como un selector destacado.

---

## 4. Campos condicionales (`showIf`)

Si una variante necesita un campo que otra no, usa `showIf`: el campo solo aparece cuando se cumple la condición.

```ts
fields: {
  variant: { type: "variant", options: ["text", "video"] },
  quote:   { type: "richtext", translatable: true, showIf: { variant: ["text"] } },
  video:   { type: "asset",                          showIf: { variant: ["video"] } },
  author:  { type: "text", translatable: true },     // siempre visible
},
```
En el editor, al cambiar la variante a "video", desaparece `quote` y aparece `video`. El `.astro` renderiza lo que corresponda según `variant`.

---

## 5. Presets (plantillas de inserción, opcional)

Un **preset** ofrece un punto de partida con campos pre-rellenados al añadir el bloque (no cambia el modelo de datos):
```ts
export const schema: BlockSchema = {
  name: "testimonial",
  display_name: "Testimonio",
  fields: { /* … */ },
  presets: [
    { name: "Testimonio en tarjeta", values: { variant: "card",   author: "Nombre" } },
    { name: "Testimonio destacado",  values: { variant: "banner", author: "Nombre" } },
  ],
};
```
> Los presets son una mejora de la v1 posterior; las variantes y `showIf` sí entran en la v1.

---

## 6. Reglas y errores comunes

- ⚠️ El `name` del bloque debe ser **único** y **estable**: si lo cambias, el contenido existente deja de encontrar su componente.
- ⚠️ Marca como `translatable: true` todo campo con **texto visible** (lo demás no).
- ⚠️ Tras tocar un esquema, ejecuta `pnpm push-schemas` o el editor no verá los cambios.
- ⚠️ El HTML del bloque debe usar tokens; si metes valores fijos, rompes el design system y el theming.
- Si el editor no muestra el bloque: revisa que esté en `registry.ts` **y** que hayas hecho `push-schemas`.
- Si cambias campos de un bloque que **ya tiene contenido publicado**, ojo con la **evolución de esquemas** (qué pasa con el contenido viejo): es un tema aparte que conviene tener previsto antes de cambios destructivos.

---

## 7. Checklist para dar por hecho un bloque

- [ ] Carpeta con `Componente.astro` + `componente.schema.ts`.
- [ ] Esquema tipado con `BlockSchema`; campos de texto marcados `translatable`.
- [ ] Estilos con design tokens (sin píxeles a mano).
- [ ] Registrado en `registry.ts` (mapa + esquemas).
- [ ] `pnpm push-schemas` ejecutado.
- [ ] Probado en el editor: se añade, se edita, se ve en el preview y al publicar.
- [ ] Si tiene variantes: cada una renderiza bien y los `showIf` muestran/ocultan los campos correctos.
