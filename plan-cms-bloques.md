# Plan técnico — CMS de bloques para Astro (multi-idioma)

> Documento de planificación pensado para un equipo junior. Cada sección explica **qué** se construye, **por qué**, **cómo** y **qué buscar** si algo no se entiende. Las estimaciones son orientativas (jornadas de desarrollo, "j/d") y asumen **2 personas de perfil junior-mid** trabajando en paralelo. Multiplica o divide según tu equipo real.

---

## 0. Cómo leer este documento

- Las palabras marcadas en `código` son nombres técnicos: búscalas tal cual si no las conoces.
- Al final de cada bloque grande hay una caja **"Para aprender"** con los conceptos exactos que deberías investigar.
- El símbolo 🎯 marca decisiones ya tomadas (no las re-discutas sin motivo).
- El símbolo ⚠️ marca trampas conocidas donde la gente pierde días.

---

## 1. Glosario (léelo primero)

| Término | Qué significa en este proyecto |
|---|---|
| **Bloque** (block / blok) | Unidad de contenido reutilizable. Tiene un `_uid` único, un `component` (su tipo) y campos. Ej: un "hero", un "botón". |
| **Esquema de bloque** (block schema) | Definición de qué campos tiene un tipo de bloque y de qué tipo es cada campo. El editor lo usa para pintar el formulario. |
| **Documento** (document / story) | Una página o entrada de contenido. Su cuerpo es un **árbol de bloques** en JSON. |
| **Campo** (field) | Una propiedad de un bloque: texto, número, imagen, richtext, lista de bloques anidados, etc. |
| **Draft / Publicado** | El draft es el contenido en edición (lo ve el editor). Publicado es lo que ve el público. |
| **Locale** | Un idioma-región concreto, ej. `es`, `en`, `ca`, `es-MX`. |
| **Site / Space** | Un sitio web concreto. El CMS puede gestionar varios. |
| **Bridge** | Script JS que se carga en el frontend **solo durante la edición** para comunicarse con el editor. |
| **Editor visual** | La aplicación (SPA) donde se edita el contenido viendo el resultado en vivo. |
| **SSG / SSR** | Static Site Generation (HTML generado en build) / Server-Side Rendering (HTML generado en cada petición). |
| **i18n** | "internationalization" — el soporte de varios idiomas. |
| **Mutación** | Un cambio discreto y con nombre sobre el contenido (`moveBlock`, `updateField`…). El editor trabaja con mutaciones, no guardando el documento entero. |
| **Morph** | Técnica para actualizar el DOM cambiando solo lo que difiere (con `idiomorph`/`morphdom`), sin recargar ni perder el scroll. |
| **Token (design token)** | Valor con nombre del sistema de diseño (ej. espaciado `md`, color `primary`). Los campos de estilo eligen tokens, no valores libres. |
| **Theme settings** | Subconjunto curado de tokens globales editable desde el CMS (estilo Shopify): color de marca, fuente, etc. |
| **postMessage** | API del navegador para que dos documentos (editor e iframe) se manden mensajes de forma segura. |

---

## 2. Visión y alcance

Construimos un **CMS headless basado en bloques con editor visual**, al estilo Storyblok, para alimentar webs hechas en **Astro**. Tres piezas independientes que se comunican por HTTP/JSON:

1. **API / Backend** — almacena y sirve el contenido, los esquemas, los assets y los usuarios.
2. **Editor visual (SPA)** — interfaz de edición con vista previa en vivo.
3. **Frontend Astro** — la web pública que renderiza los bloques.

🎯 **Decisiones ya tomadas:**
- Producción **híbrida**: cada página decide si es estática (SSG) o dinámica (SSR).
- El editor se construye **desde cero** en su lógica, reutilizando librerías para drag & drop, richtext y estado.
- 🎯 **Modelo de negocio: SaaS multi-tenant gestionado** — tú hospedas una plataforma central y cada cliente accede a su espacio. Aislamiento por tenant (ver Anexo A, multi-tenancy y RLS).
- Diseño **multi-tenant / multi-sitio** desde el día 1 (columna `site_id` + Row-Level Security), que es la base del modelo SaaS.
- **Multi-idioma obligatorio** (ver sección 7).
- **Servicio propio de imágenes obligatorio** (ver sección 11): transformación on-the-fly + caché.

**Módulo opcional (activable, no bloquea la v1):** colaboración en tiempo real (ver sección 11.bis). Se diseña para que pueda añadirse después sin reescribir el editor.

**Fuera de alcance (versión 1):** flujos de aprobación complejos, comentarios, A/B testing, plugins de terceros.

---

## 3. Arquitectura general

```
                ┌─────────────────────────────┐
                │        API / Backend        │
                │  Documentos (árbol bloques) │
                │  Esquemas · Assets · i18n   │
                │  Usuarios · Versiones       │
                └──────────┬─────────┬────────┘
        esquemas + drafts  │         │  contenido publicado
                  ▲        │         │        │
                  │        ▼         ▼        ▼
        ┌─────────┴────────┐      ┌──────────────────────┐
        │  Editor (SPA)    │      │   Frontend Astro      │
        │  ┌─────┐ ┌─────┐ │◀────▶│  Bloques recursivos   │
        │  │Campos│ │Prev.│ │ post │  component → .astro   │
        │  └─────┘ └─────┘ │ Msg  │  bridge.js (solo edit)│
        └──────────────────┘      └──────────────────────┘
```

**Flujos clave:**
- El editor **lee esquemas y drafts** de la API y **guarda** cambios.
- El frontend público **lee solo contenido publicado**.
- El preview del editor es el **frontend real en modo draft**, dentro de un iframe, comunicándose por `postMessage`.

> **Para aprender:** "headless CMS", "API REST vs GraphQL", "iframe postMessage API".

---

## 4. Stack tecnológico

🎯 Elecciones por defecto. Si el equipo domina otra herramienta equivalente, puede sustituirla, pero documenta el cambio.

| Pieza | Tecnología | Por qué |
|---|---|---|
| Lenguaje común | **TypeScript** | Tipado compartido entre las 3 piezas; menos bugs. |
| Backend/API | **Hono** (o Fastify) | Ligero, rápido, corre en Node/Edge. Reutilizable por varias webs. |
| Base de datos | **PostgreSQL** | El tipo `jsonb` es ideal para árboles de bloques + consultas. |
| ORM | **Drizzle** | Tipado, migraciones claras, ligero. (Prisma es alternativa válida.) |
| Editor (SPA) | **React + Vite** | Ecosistema enorme de librerías de UI. |
| Drag & drop | **dnd-kit** | Estándar moderno, accesible. NO lo escribas a mano. |
| Richtext | **TipTap** (sobre ProseMirror) | Editor de texto rico serio. NO uses `contenteditable` crudo. |
| Estado del editor | **Zustand** | Simple, sin boilerplate. |
| Formularios | Generador propio + **react-hook-form** | Los formularios se generan desde el esquema (ver sección 8.2). |
| Frontend público | **Astro** (output híbrido) | Requisito del proyecto. |
| Assets | **S3 / Cloudflare R2** | Almacenamiento de objetos barato. (Local en desarrollo.) |
| Auth | **Lucia** o JWT propio | Sesiones para el editor. |

> **Para aprender:** "TypeScript monorepo", "PostgreSQL jsonb", "Drizzle ORM migrations", "dnd-kit sortable", "TipTap getting started", "Zustand store".

---

## 5. Estructura del repositorio

🎯 Hay **dos clases de repositorio** que no se mezclan: el **core del CMS** (un monorepo, se construye una vez) y **cada web cliente** (un repo Astro por web, consume el core). El core es el producto; las webs son sus consumidores.

```
cms-core/ (monorepo · pnpm workspaces)        web-clienteA/ (repo Astro aparte)
  packages/                                     src/blocks/  (sus bloques + esquemas)
    shared/   → tipos y contratos               src/pages/   (sus páginas)
    api/      → backend Hono + Drizzle + DB      styles/tokens.css
    sdk/      → cliente JS (publicable)          depende de @cms/sdk
    editor/   → SPA React + Vite
    create-web/ → plantilla de web cliente     web-clienteB/ …  (otra web, otro repo)
```

🎯 El paquete **`shared`** es crítico: ahí viven los tipos que comparten las piezas, así un cambio en el modelo rompe la compilación donde haga falta (eso te avisa). `sdk` y `shared` se **publican a un registry** y cada web los instala fijando versión.

> 📎 **El detalle completo de proyectos, base de datos (DDL), stack y arquitectura interna de cada proyecto está en el `Anexo A — Arquitectura técnica detallada`.**

> **Para aprender:** "pnpm workspaces", "monorepo turborepo", "changesets versioning", "private npm registry".

---

## 6. Modelo de datos (el contrato — lo más importante)

⚠️ **Si te equivocas aquí, lo pagas en todas las demás piezas.** Diséñalo con calma antes de escribir código de UI.

### 6.1 El formato de un bloque

Todo bloque es un objeto JSON con dos campos obligatorios:

```jsonc
{
  "_uid": "a1b2c3",        // id único e irrepetible (genéralo con nanoid/uuid)
  "component": "hero",     // tipo de bloque → mapea a un componente Astro
  // ...resto de campos según su esquema
  "title": "Bienvenido",
  "buttons": [             // campo "bloks": lista de bloques anidados
    { "_uid": "d4e5f6", "component": "button", "label": "Ver más", "url": "/x" }
  ]
}
```

⚠️ El `_uid` debe ser **estable**: una vez creado un bloque, su `_uid` no cambia nunca (lo necesita el editor para seleccionarlo y el sistema i18n para mapear traducciones).

### 6.2 El esquema de un bloque

Define qué campos tiene cada `component` y cuáles son traducibles:

```jsonc
{
  "name": "hero",
  "display_name": "Cabecera",
  "fields": {
    "title":    { "type": "text",      "translatable": true },
    "subtitle": { "type": "textarea",  "translatable": true },
    "image":    { "type": "asset" },
    "cta":      { "type": "richtext",  "translatable": true },
    "buttons":  { "type": "bloks",     "allow": ["button"] },
    "theme":    { "type": "option",    "options": ["light", "dark"] }
  }
}
```

**Tipos de campo mínimos para la v1:** `text`, `textarea`, `richtext`, `number`, `boolean`, `option` (select), `options` (multi), `asset` (imagen/archivo), `link` (interno/externo), `date`, `bloks` (lista de bloques anidados), `reference` (enlace a otro documento), `token` (valor del design system: padding, ancho, color… ver sec. 9.bis), `variant` (variación visual del bloque, ver sec. 6.4).

Además, cualquier campo admite `showIf` para mostrarse solo bajo cierta condición (campo condicional, ver sec. 6.4).

### 6.3 Tablas de base de datos

```
sites            (id, name, slug, default_locale, locales[])      ← multi-sitio
users            (id, email, password_hash, name)
memberships      (user_id, site_id, role)                         ← quién accede a qué sitio
block_schemas    (id, site_id, name, display_name, fields jsonb)  ← esquemas por sitio
documents        (id, site_id, type, parent_id)                   ← metadatos de la página
content          (id, document_id, locale, slug, status, data jsonb, seo jsonb,
                  publish_at, deleted_at, updated_at, updated_by)  ← contenido + SEO + scheduling + papelera
versions         (id, content_id, data jsonb, created_at, author) ← historial
assets           (id, site_id, filename, url, mime, width, height, alt jsonb)
theme_settings   (id, site_id, tokens jsonb)                       ← design tokens editables (opcional, sec. 9.bis)
redirects        (id, site_id, locale, from_slug, to_slug, code)   ← redirecciones 301 al cambiar slugs
audit_log        (id, site_id, user_id, action, target, created_at)← trazabilidad de acciones
```
> La facturación, planes y cuotas (tablas `plans`, `subscriptions`, `usage`) viven en la capa SaaS: ver `saas-operaciones.md`.

🎯 **Decisión clave de multi-idioma:** el contenido se guarda **una fila por (documento, locale)** en la tabla `content`. Es decir, la página "home" en español y en inglés son dos filas con el mismo `document_id` pero distinto `locale`. Los `_uid` de los bloques **se comparten entre idiomas**, lo que permite "copiar la estructura del idioma maestro y traducir solo los textos". Ver sección 7 para el detalle completo y por qué este enfoque.

⚠️ `status` vive a nivel de `content` (documento+locale), no del documento. Así puedes tener el español publicado y el inglés todavía en draft.

> **Para aprender:** "PostgreSQL jsonb indexing (GIN)", "database normalization vs denormalization", "uuid vs nanoid", "soft delete pattern".

### 6.4 Variantes, presets y campos condicionales ⭐

Tres conceptos distintos que la gente suele mezclar. Defínelos bien porque tocan el contrato y el editor.

**Variante** = el *mismo* bloque mostrado de otra forma (un "hero" centrado, a la izquierda, partido o con vídeo). El contenido es el mismo; cambia la presentación. Se modela con un campo de tipo `variant`, y el `.astro` decide el layout según su valor. 🎯 No crees bloques separados (`hero-left`, `hero-center`): es el mismo bloque con un campo de variante.

**Campo condicional (`showIf`)** = un campo que solo aparece bajo cierta condición, normalmente según la variante. Es lo que hace potentes a las variantes: la de vídeo necesita un campo `video` que las demás no.

```jsonc
{
  "name": "hero",
  "display_name": "Cabecera",
  "fields": {
    "variant": { "type": "variant", "options": ["centered", "left", "split", "video"] },
    "title":   { "type": "text",  "translatable": true },
    "image":   { "type": "asset", "showIf": { "variant": ["centered", "left", "split"] } },
    "video":   { "type": "asset", "showIf": { "variant": ["video"] } }
  },
  "presets": [
    { "name": "Hero con imagen", "values": { "variant": "left",  "title": "Tu título" } },
    { "name": "Hero con vídeo",  "values": { "variant": "video", "title": "Tu título" } }
  ]
}
```
```astro
---
const { variant = "centered" } = Astro.props;
---
<section class={`hero hero--${variant}`}> … </section>
```

**Preset** = una plantilla de inserción: un punto de partida con campos pre-rellenados que el editor ofrece al añadir el bloque. No cambia el modelo de datos, solo la experiencia de autoría (es lo que Shopify llama "presets").

🎯 **Alcance:** las **variantes** y los **campos condicionales** entran en la v1 (son baratos y de alto valor); los **presets** quedan como mejora posterior (son solo azúcar al insertar). En el editor, el campo `variant` se muestra como un selector destacado y los campos con `showIf` aparecen/ocultan según la variante elegida (ver sec. 8.3).

### 6.5 SEO, redirecciones y publicación programada ⭐

Tres cosas que tocan el modelo de datos y conviene cerrar **antes** de crear la base de datos (afectan al esquema).

**Metadatos SEO por página y por idioma.** Cada `content` (documento+locale) guarda un objeto `seo` con `title`, `description` y `og_image`. El frontend los vuelca en el `<head>`. Si faltan, se derivan de campos del contenido (p. ej. el primer título). Son traducibles por definición (viven por locale).
```jsonc
"seo": { "title": "Contacto | Mi web", "description": "Habla con nosotros", "og_image": "asset_id" }
```

**Redirecciones (301).** Cuando un editor cambia el `slug` de una página publicada, el slug viejo deja de existir y rompería enlaces y SEO. Al guardar un cambio de slug, se crea automáticamente una fila en `redirects` (`from_slug` → `to_slug`, por `site_id` + `locale`). El frontend resuelve los redirects antes de devolver 404. El editor también permite gestionarlos a mano.

**Publicación programada (scheduling).** El campo `publish_at` permite publicar en una fecha futura: un trabajo periódico (cron) revisa los contenidos con `publish_at` vencido y los publica. Si está vacío, la publicación es inmediata como hasta ahora.

> **Para aprender:** "Open Graph metadata", "301 vs 302 redirects", "scheduled jobs / cron", "sitemap.xml".

---

## 7. Multi-idioma (i18n) — sección transversal

Esto toca **las cuatro piezas** (modelo, API, editor y frontend). Léela entera.

### 7.1 Las dos estrategias posibles

| Estrategia | Cómo funciona | Pros | Contras |
|---|---|---|---|
| **A. Por documento+locale** (🎯 elegida) | Una fila de `content` por cada idioma. Los `_uid` se comparten. | Simple de razonar y consultar. Cada idioma se publica por separado. Estructuras pueden divergir. | Hay que mantener la sincronía de estructura entre idiomas (mitigable). |
| **B. Por campo** (field-level) | Un solo documento; cada campo traducible guarda `{es:"...", en:"..."}`. | Una sola estructura, imposible que diverja. | jsonb más complejo, fallbacks por campo, el editor se complica. |

🎯 **Elegimos la A** por simplicidad y porque encaja con el flujo "traduzco página entera". Si en el futuro necesitas que la estructura sea idéntica entre idiomas por contrato, migrarías a B.

### 7.2 Cómo se configura

- Cada `site` tiene un `default_locale` (idioma maestro, ej. `es`) y una lista `locales` (ej. `["es","en","ca"]`).
- Al crear un documento, se crea el contenido en el `default_locale`.
- Para traducir, el editor **clona la estructura** del idioma maestro a un nuevo locale (mismos `_uid`, textos vacíos o pre-rellenados) y el traductor rellena.

### 7.3 Fallbacks (qué pasa si falta una traducción)

⚠️ Define la política y respétala en API **y** frontend:
- Si se pide la home en `ca` y no existe contenido publicado en `ca` → **fallback** al `default_locale` (`es`).
- A nivel de campo (si usaras estrategia B) el fallback sería por campo. Con la estrategia A el fallback es a nivel de documento, más simple.

### 7.4 Routing i18n en el frontend (Astro)

Astro tiene i18n nativo. Configura:
```js
// astro.config.mjs
i18n: {
  defaultLocale: "es",
  locales: ["es", "en", "ca"],
  routing: { prefixDefaultLocale: false } // /sobre  y  /en/about
}
```
- URLs: `/contacto` (es, idioma por defecto sin prefijo) y `/en/contact`, `/ca/contacte`.
- El slug puede ser distinto por idioma (`contacto` vs `contact`): guárdalo por locale (en `content` o en una tabla `slugs(document_id, locale, slug)`).

### 7.5 i18n del propio editor (la interfaz)

No confundir: el **contenido** es multi-idioma (sección 7.2) y la **interfaz del editor** (botones, menús) también debería estarlo. Usa `react-i18next` con archivos de traducción. Esto es independiente y más sencillo.

> **Para aprender:** "Astro i18n routing", "content fallback strategy CMS", "react-i18next", "hreflang SEO multilingual".

---

## 8. El Editor visual (la pieza más grande)

Es el ~50% del esfuerzo total. Se descompone en módulos independientes.

### 8.0 El modelo mental (léelo ANTES que nada) ⭐⭐

🎯 **El estado del editor (Zustand) es la ÚNICA fuente de verdad.** Ni el iframe del preview ni el árbol del aside cambian el contenido por su cuenta: lo único que hacen es registrar **intenciones** (a las que llamamos *mutaciones*). El estado las aplica y, a partir de ahí, todo lo demás (preview, árbol, panel de config) se re-renderiza desde ese estado.

**El ciclo de edición, siempre el mismo:**
```
1. El usuario hace algo en el iframe (clic, arrastrar, escribir)
2. bridge.js NO toca el HTML: envía una INTENCIÓN por postMessage
3. El editor aplica la mutación al estado  ← la fuente de verdad
4. Se re-renderiza el HTML y se aplica al iframe con "morph" (sec. 9.3)
5. El preview queda listo para la siguiente edición → vuelve a 1
```

⚠️ **Modela los cambios como mutaciones discretas** (`insertBlock`, `moveBlock`, `updateField`, `deleteBlock`…), nunca como "guardo el documento entero de golpe". Esto te regala tres cosas: el **undo/redo** se vuelve trivial (sec. 8.7), la sincronía iframe↔aside es automática, y dejas la puerta abierta a la **colaboración en tiempo real** (sec. 11.bis) sin reescribir nada. Es la inversión barata más rentable del proyecto.

### 8.1 Árbol de bloques en el aside (outline)
Vista lateral con la jerarquía de bloques del documento. Permite seleccionar, duplicar y borrar bloques. El reordenar se trata aparte (8.2). Se alimenta del estado; al hacer clic en un nodo emite la mutación `select`.
- **Esfuerzo:** 4–6 j/d.

### 8.2 Reordenar y mover bloques — las dos vías ⭐
Existen dos formas de arrastrar, y son **técnicamente distintas** porque ocurren en documentos diferentes:

1. **Desde el aside** (el árbol): es `dnd-kit` normal, porque vive en tu app React. Al soltar, emites la mutación `moveBlock`.
2. **Desde dentro del iframe**: aquí `dnd-kit` **no llega**, porque los eventos de puntero ocurren en otro documento. La solución es que **el bridge implemente su propio arrastre**: escucha `pointerdown/pointermove/pointerup` sobre los elementos con `data-blok-uid`, dibuja la línea indicadora de dónde caería el bloque, y al soltar envía `{ type: "reorder", uid, toIndex }` al editor.

⚠️ Arrastrar **cruzando la frontera** (sacar un bloque del aside y soltarlo dentro del iframe) es lo más caro de todo el editor: los eventos de drag nativos no cruzan el límite del iframe y hay que coordinar coordenadas de puntero por `postMessage`. **Recomendación v1:** que cada contexto gestione su propio arrastre, y que "añadir un bloque nuevo" se haga con un clic o soltándolo sobre el árbol del aside. Deja el drag aside→iframe para una iteración posterior si de verdad se necesita.
- **Esfuerzo:** 6–9 j/d.

### 8.3 Panel de configuración del bloque (formularios desde el esquema) ⭐
El corazón del editor. Al hacer clic en un bloque (en el iframe o en el árbol), el editor recibe la mutación `select` y abre el **panel derecho** con un formulario **generado a partir del esquema** de ese bloque. Por cada campo del esquema pinta el input adecuado: `text`→input, `richtext`→editor (8.4), `asset`→selector de media, `bloks`→sub-formulario anidado **recursivo**, `token`→select de tokens del design system (sec. 9.bis), etc.

Como cada bloque tiene su esquema, **el mismo mecanismo sirve para todos**: cambiar la imagen del hero, el padding, el tamaño… todo lo que el esquema declare. La clave es que los campos de estilo (padding, ancho, etc.) son de tipo `token`, es decir **selects de valores válidos del sistema**, no inputs libres de píxeles. Así el editor no puede romper el diseño (ver sec. 9.bis).
- ⚠️ El campo `bloks` se renderiza **recursivamente** (un formulario dentro de otro). Diséñalo recursivo desde el principio.
- El campo `variant` se muestra como un **selector destacado** (idealmente con miniaturas), y los campos con `showIf` **aparecen/se ocultan** según la variante elegida (sec. 6.4). Diseña el formulario para reaccionar a `showIf` desde el principio: añadirlo después es incómodo.
- **Esfuerzo:** 8–12 j/d (cada tipo de campo es un pequeño componente reutilizable).

### 8.4 Edición inline de richtext con toolbar flotante ⭐
El texto enriquecido que ves en el iframe es HTML que renderizó Astro (estático, no editable). Para editarlo al hacer clic, el bridge **monta una instancia de TipTap sobre ese nodo**, convirtiéndolo en editable, con su *bubble menu* (la toolbar que aparece al seleccionar texto, el estilo "Word" clásico: negrita, enlace, encabezado…).

- TipTap tiene que correr **dentro del iframe**, porque ahí vive el DOM que edita; por tanto **lo carga el bridge**, no la app del editor.
- ⚠️ El HTML que produce TipTap y el HTML que produce el render de Astro **para el mismo richtext deben coincidir** (mismos tags, mismas clases). Si no, el texto "salta" visualmente al entrar/salir de edición. Se resuelve definiendo el **formato del richtext una sola vez** (qué marcas y nodos se permiten) y usándolo en ambos lados.
- Al terminar de editar, el bridge envía la mutación `updateField` con el nuevo richtext.
- **Esfuerzo:** 6–9 j/d (es la subtarea más arriesgada del editor; préstale margen).

### 8.5 Selección visual y resaltado (overlays)
Clic en un bloque del preview → lo selecciona y abre su panel. Hover → lo resalta. El bridge dibuja los bordes/overlays **dentro del iframe** inyectando unos pocos estilos mínimos (un `outline` y poco más): es el **único CSS del editor que entra al iframe**, y es deliberado y aislado (no toca el diseño de la web). Requiere que cada bloque lleve `data-blok-uid`.
- **Esfuerzo:** 3–5 j/d.

### 8.6 Selector de idioma + estado de traducción
Desplegable de locale. Indica qué idiomas están traducidos/publicados. Botón "traducir desde idioma maestro" (clona la estructura, ver sec. 7.2).
- **Esfuerzo:** 4–6 j/d.

### 8.7 Undo / redo
Trivial **gracias al modelo de mutaciones** (8.0): mantienes una pila de mutaciones (con su inversa) o de snapshots del estado. Zustand + un middleware de historial.
- **Esfuerzo:** 3–5 j/d.

### 8.8 Gestor de assets (media library)
Subir, listar, buscar, borrar imágenes. Integra con el servicio de imágenes (sec. 11.2). Campo `alt` traducible por idioma.
- **Esfuerzo:** 5–8 j/d.

### 8.9 Guardado, autosave y publicación
Guardar draft (con autosave/debounce), publicar por locale, ver versiones y restaurar. Manejo de conflictos básicos (last-write-wins en v1).
- **Esfuerzo:** 5–7 j/d.

### 8.10 Login y gestión de roles
Pantalla de login, sesión, y permisos básicos (admin/editor) por sitio.
- **Esfuerzo:** 4–6 j/d.

### 8.11 Protocolo de comunicación editor ↔ bridge (postMessage) ⭐
El contrato concreto de mensajes que cruzan la frontera del iframe. Defínelo **tipado en `/shared`** para que editor y bridge no se desincronicen.

**Del bridge (iframe) → al editor:**

| Mensaje | Cuándo | Datos |
|---|---|---|
| `ready` | El preview terminó de cargar | `{ }` |
| `select` | Clic en un bloque | `{ uid }` |
| `hover` | Ratón sobre un bloque | `{ uid \| null }` |
| `reorder` | Soltó un arrastre dentro del iframe | `{ uid, toIndex, parentUid }` |
| `updateField` | Editó richtext inline | `{ uid, field, value }` |
| `requestAdd` | Pulsó "+" entre dos bloques | `{ parentUid, index }` |

**Del editor → al bridge (iframe):**

| Mensaje | Cuándo | Datos |
|---|---|---|
| `render` | El estado cambió | `{ html }` (el bridge hace *morph*) |
| `highlight` | Hover desde el árbol del aside | `{ uid \| null }` |
| `scrollTo` | Seleccionó un bloque en el aside | `{ uid }` |
| `setEditable` | Entrar/salir de edición de richtext | `{ uid, on }` |

**Reglas de oro:**
- ⚠️ **Valida siempre `event.origin`** en ambos lados; ignora todo lo que no venga del origen esperado.
- Versiona el protocolo (`{ v: 1, type, ... }`) para poder evolucionarlo sin romper despliegues.
- El bridge **nunca** muta el contenido: solo manda intenciones (coherente con 8.0).
- **Esfuerzo:** 2–3 j/d para formalizarlo y tiparlo (la implementación va repartida en 8.2–8.5).

> **Para aprender:** "schema-driven form rendering", "recursive React components", "iframe postMessage security origin check", "pointer events drag and drop", "TipTap bubble menu", "command pattern undo redo".

---

## 9. El Frontend Astro

### 9.1 Render recursivo de bloques
Un componente `<Blocks>` recibe un array de bloques y, por cada uno, busca su componente Astro en un mapa `component → .astro` y lo renderiza. Los bloques anidados vuelven a llamar a `<Blocks>`.
```astro
---
import { blockMap } from "./blockMap";
const { blocks = [] } = Astro.props;
---
{blocks.map((blok) => {
  const Cmp = blockMap[blok.component];
  return Cmp ? <Cmp {...blok} data-blok-uid={blok._uid} /> : null;
})}
```
- **Esfuerzo:** 4–6 j/d (el mapa crece con cada tipo de bloque del proyecto).

### 9.2 Obtención de datos (data fetching) + híbrido
Un pequeño SDK (`/packages/sdk`) pide a la API el contenido publicado de un slug+locale.
- Páginas estáticas: `export const prerender = true` + `getStaticPaths` recorre los slugs.
- Páginas dinámicas: `prerender = false`, fetch en cada request.
- 🎯 La decisión SSG/SSR es **por ruta**, según el proyecto y el tipo de contenido.
- **Esfuerzo:** 5–8 j/d.

### 9.3 Modo preview / draft + bridge.js ⭐
🎯 **Idea central: el preview NO es una recreación de la web dentro del CMS; es tu web Astro real**, servida en modo draft y cargada en un `<iframe>`. Por eso los estilos coinciden al 100% sin coordinarlos: es el mismo código que verá el usuario.

- El editor pone en el `src` del iframe una URL de preview, ej. `https://preview.tusitio.com/es/home?_preview=TOKEN`.
- Esa ruta corre en **SSR aunque en producción esa página sea estática**, pide el contenido **draft** a la API (autenticado con el token), renderiza el árbol poniendo `data-blok-uid` en cada bloque, e incluye `bridge.js` **solo** en este modo.
- ⚠️ `bridge.js` **nunca** debe acabar en producción pública (carga condicional + revisión en CI).

**Aislamiento de estilos (responde a "¿cómo coordino el CSS del CMS y el de Astro?"):** no se coordinan, **se aíslan**. El iframe es un documento independiente con su propio CSS (Tailwind, fuentes, tokens). El "chrome" del editor (panel, árbol, toolbar) vive **fuera** del iframe con sus propios estilos. El iframe garantiza que el CSS de uno nunca se filtre al otro: cero colisiones. La única excepción son los overlays de selección/hover, que el bridge inyecta dentro del iframe a propósito (sec. 8.5).

**Render en vivo (cómo se actualiza el preview al editar):** tres estrategias, de simple a fina:
1. **Recargar el iframe** (con debounce ~300 ms). Trivial, pero parpadea y pierde el scroll. Solo para arrancar.
2. **Re-render + morph (recomendado):** el editor manda el árbol actual a un endpoint que devuelve el HTML; el bridge hace *morph* del DOM (con `idiomorph`/`morphdom`) cambiando solo lo que difiere. Instantáneo, sin parpadeo, conserva el scroll.
3. **Render parcial por bloque** (optimización posterior): un endpoint renderiza un solo bloque por `_uid` y el bridge reemplaza ese nodo.

```js
// Editor: el contenido cambió → pide HTML (sin pasar por la BD) y lo manda al iframe
const html = await fetch("/preview/render", {
  method: "POST",
  body: JSON.stringify({ blocks, locale: "es" }),
}).then(r => r.text());
iframe.contentWindow.postMessage({ v: 1, type: "render", html }, PREVIEW_ORIGIN);

// bridge.js (dentro del iframe): recibe el HTML y hace morph
window.addEventListener("message", (e) => {
  if (e.origin !== EDITOR_ORIGIN) return;     // ⚠️ valida siempre el origin
  if (e.data.type === "render") {
    Idiomorph.morph(document.body, e.data.html, { morphStyle: "innerHTML" });
  }
});
```
⚠️ El endpoint `/preview/render` recibe el árbol **en el body**, sin guardarlo en la BD antes: eso es lo que hace el preview fluido (no hace falta guardar para ver el cambio).
- **Esfuerzo:** 8–12 j/d (incluye el endpoint de render y el morph).

### 9.4 Webhooks de revalidación/rebuild
Cuando se publica contenido, la API dispara un webhook que regenera las páginas estáticas afectadas (o invalida caché en SSR).
- **Esfuerzo:** 3–5 j/d.

### 9.5 SEO multi-idioma
`hreflang`, `canonical`, sitemap por idioma, metadatos por locale.
- **Esfuerzo:** 3–4 j/d.

> **Para aprender:** "Astro getStaticPaths", "Astro hybrid rendering adapter", "Astro middleware", "ISR incremental static regeneration", "iframe DOM morphing idiomorph", "hreflang tags".

---

## 9.bis Design system, tokens y theme editor ⭐

Esta sección es transversal: toca el **esquema** (campos de tipo `token`), el **frontend** (variables CSS) y el **editor** (selects en vez de inputs libres). Responde a "¿debería el design system ser editable desde el CMS, como Shopify?".

### 9.bis.1 Dos niveles que NO hay que confundir
1. **Tokens globales** (design tokens): la paleta, la escala de espaciado, la tipografía, los radios. Es lo que Shopify expone en su *theme editor*.
2. **Config por bloque:** qué token usa *este* bloque concreto (padding = `md`), eligiendo de entre los tokens.

### 9.bis.2 La regla de oro: tokens, no valores libres
🎯 Los campos de estilo de un bloque (padding, ancho, color…) son de tipo `token`: **selects de opciones válidas del sistema**, no inputs libres de píxeles o hex.
```jsonc
"padding":  { "type": "token", "scale": "spacing" },    // none · sm · md · lg · xl
"maxWidth": { "type": "token", "scale": "container" },   // narrow · default · wide
"accent":   { "type": "token", "scale": "color" }        // primary · secondary · muted
```
Así el editor **no puede romper el diseño**: el design system actúa de barandilla. Es exactamente lo que hace Shopify: opciones curadas, nunca CSS libre.

### 9.bis.3 ¿Editable desde el CMS? Recomendación matizada
- **Sí**, exponer un subconjunto **pequeño y curado** de tokens globales como "ajustes de tema" editables (color de marca, fuente, densidad de espaciado) tiene sentido y da flexibilidad sin perder coherencia.
- **No** a un editor de CSS libre: destruye el sistema y genera webs inconsistentes.
- ⚠️ **Orden recomendado:** empieza con los tokens **fijos en código** (tu design system, sin editar desde el CMS) y los campos de bloque eligiendo de ellos. Eso ya da el 90% del valor. Expón el theme editor tipo Shopify **solo cuando aparezca la necesidad real** (multi-marca, clientes que personalizan). Si lo dejas montado sobre variables CSS desde el principio, añadirlo luego es barato; lo caro es construirlo antes de necesitarlo.

### 9.bis.4 Cómo se implementa (técnica)
Los tokens son **variables CSS**. Si se hacen editables, guardas sus valores por sitio (tabla `theme_settings` o un "documento de tema" especial) y el frontend Astro los inyecta en el `<head>`:
```astro
---
const theme = await getThemeSettings(site, locale);
---
<style is:global>
  :root {
    --color-primary: {theme.colorPrimary};
    --space-md: {theme.spaceMd};
    --font-body: {theme.fontBody};
  }
</style>
```
Todos los bloques usan esas variables, así que cambiar un token reconfigura toda la web de forma coherente. Para el preview, idéntico: el iframe carga los tokens en versión draft.
- **Esfuerzo:** tokens fijos en código (incluido en el frontend). Theme editor editable desde el CMS (opcional): **8–12 j/d** adicionales.

> **Para aprender:** "design tokens", "CSS custom properties theming", "Shopify theme settings_schema", "Style Dictionary".

---

## 10. La API / Backend

### 10.1 Endpoints mínimos (REST)
```
# Esquemas
GET    /sites/:site/schemas
POST   /sites/:site/schemas
PUT    /sites/:site/schemas/:name

# Documentos y contenido
GET    /sites/:site/documents?locale=es&status=published
GET    /sites/:site/documents/:slug?locale=es&status=draft
POST   /sites/:site/documents
PUT    /sites/:site/documents/:id/content?locale=es     ← guardar draft
POST   /sites/:site/documents/:id/publish?locale=es     ← publicar
GET    /sites/:site/documents/:id/versions

# Assets
POST   /sites/:site/assets        (upload)
GET    /sites/:site/assets
DELETE /sites/:site/assets/:id

# Auth
POST   /auth/login
POST   /auth/logout
GET    /auth/me
```
- **Esfuerzo:** 10–15 j/d.

### 10.2 Reglas importantes
- ⚠️ El público **solo** accede a `status=published`. El editor accede a `draft`. Protégelo con auth.
- Caché agresiva en lectura publicada; sin caché en draft.
- Multi-sitio: cada query filtra por `site_id` (sácalo del token o de la ruta).

### 10.3 Seguridad
Validación de input (Zod), rate limiting, CORS configurado, validación de `origin` para el editor, sanitización del richtext antes de servir.
- **Esfuerzo:** 4–6 j/d.

> **Para aprender:** "REST API design", "Zod validation", "JWT auth", "CORS", "HTTP caching headers", "rate limiting", "HTML sanitization XSS".

---

## 11. Assets + servicio propio de imágenes (obligatorio)

### 11.1 Gestión de assets
- Subida directa a S3/R2 (idealmente con URLs prefirmadas).
- Guarda metadatos en `assets`. `alt` es un campo traducible (`jsonb` por locale).
- **Esfuerzo:** incluido en 8.7 + 3 j/d de integración backend.

### 11.2 Servicio de imágenes 🎯 (obligatorio)
Un servicio que **transforma imágenes bajo demanda** a partir de la original y las cachea. La URL lleva los parámetros y el servicio genera la variante la primera vez que se pide, sirviéndola cacheada después.

```
/img/<asset_id>?w=800&h=600&fit=cover&format=webp&q=80
        ↓
1. ¿Existe ya esta variante en caché/almacén?  → sí: sírvela.
2. No: descarga la original, transforma con `sharp`, guarda la variante, sírvela.
3. CDN delante para que la 2ª petición no toque tu servidor.
```

**Qué debe soportar la v1:**
- Redimensionar (`w`, `h`) y modo de recorte (`fit`: cover/contain).
- Conversión de formato a **WebP/AVIF** (mucho más ligeros que JPG/PNG).
- Calidad (`q`).
- **`srcset` automático**: el componente de imagen del frontend genera varios tamaños para responsive.
- Caché de variantes + invalidación al reemplazar la original.

**Cómo construirlo sin reinventar demasiado:**
- Procesamiento con **`sharp`** (la librería estándar de Node para imágenes).
- Almacén de variantes en el mismo S3/R2, en una carpeta `derived/`.
- ⚠️ **Limita las combinaciones permitidas** (lista blanca de tamaños/formatos). Si dejas parámetros libres, un atacante puede pedir millones de variantes y reventar tu almacenamiento y CPU. Firma las URLs o valida contra presets.
- CDN: Cloudflare/CloudFront delante del endpoint.
- **Atajo válido:** si el coste de construirlo se dispara, **Cloudflare Images** o **imgix** ofrecen lo mismo gestionado; el plan asume servicio propio pero deja la puerta abierta a externalizar el motor manteniendo tu misma API de URLs.
- **Esfuerzo:** 12–18 j/d (procesado + caché + invalidación + integración con el componente del frontend + presets de seguridad).

> **Para aprender:** "sharp image processing", "S3 presigned upload URLs", "Cloudflare R2 / CDN", "responsive images srcset sizes", "image proxy signed URLs", "WebP AVIF".

---

## 11.bis Colaboración en tiempo real (módulo opcional)

No bloquea la v1, pero el editor se **diseña para poder añadirla** sin reescribirlo. Tres capacidades:

1. **Presencia:** ver quién está en el documento y qué bloque toca cada uno.
2. **Edición concurrente sin pisarse:** fusión automática de cambios simultáneos.
3. **Sincronización en vivo:** propagar cambios en milisegundos (WebSockets).

**Cómo dejar el editor preparado (hazlo aunque no actives la colaboración):**
- Modela el estado del editor como **operaciones/mutaciones discretas** sobre el árbol de bloques, no como "un blob que guardo entero". Esto es lo que luego permite enchufar un CRDT.
- Aísla el estado en su store (Zustand) detrás de una capa de "aplicar mutación", para poder sustituir el backend de estado más adelante.

**Si se activa el módulo:**
- Estado colaborativo con **Yjs** (CRDT): resuelve fusiones automáticamente.
- Transporte por WebSocket: servidor Yjs propio, o un servicio gestionado (**Liveblocks**, **PartyKit**).
- Presencia y cursores (la mayoría de estos servicios ya lo traen).
- ⚠️ Migrar el modelo de estado a CRDT **después** de tenerlo hecho de otra forma es costoso; por eso la única inversión obligatoria ahora es el punto de "mutaciones discretas" (barato) que deja la puerta abierta.
- **Esfuerzo (si se activa):** 20–30 j/d.

> **Para aprender:** "CRDT Yjs", "operational transformation", "WebSocket realtime", "Liveblocks", "PartyKit", "presence cursors".

---

## 12. Infraestructura y despliegue
- **API:** contenedor (Docker) en un PaaS (Railway/Fly/Render) o VPS. Postgres gestionado.
- **Editor:** estático en Netlify/Vercel/Cloudflare Pages.
- **Frontend:** según proyecto — estático (Pages) o con adaptador SSR (Node/Cloudflare/Vercel).
- **CI/CD:** GitHub Actions: lint, test, build, deploy por paquete.
- **Entornos:** `local` → `staging` → `production`.
- **Esfuerzo:** 6–10 j/d (incluye configurar entornos y CI).

> **Para aprender:** "Docker basics", "GitHub Actions CI/CD", "managed PostgreSQL", "environment variables secrets".

---

## 13. Calidad: testing y observabilidad
- **Unit tests** del render recursivo, el generador de formularios y los fallbacks i18n (Vitest).
- **E2E** del flujo crítico: crear → editar → publicar → ver en el frontend (Playwright).
- **Logging** estructurado y **error tracking** (Sentry).
- **Esfuerzo:** 8–12 j/d (transversal, repártelo durante todo el proyecto, no al final).

> **Para aprender:** "Vitest", "Playwright E2E", "Sentry error tracking".

---

## 14. Roadmap por fases (el "presupuesto")

Cada fase produce algo **funcional y demostrable**. No saltes el orden: las fases tempranas validan el modelo antes de invertir en lo caro.

### Fase 0 — Cimientos · ~8–12 j/d
Monorepo, paquete `/shared` con los tipos, base de datos + migraciones, CI básica.
**Entregable:** repo que compila, BD creada, tipos compartidos.

### Fase 1 — El contrato y el render · ~10–15 j/d
Definir formato de bloque y esquemas. Frontend Astro renderizando un **JSON escrito a mano** (sin API). Validar recursión y mapa de componentes.
**Entregable:** una página real renderizada desde un árbol de bloques estático.

### Fase 2 — API mínima · ~15–22 j/d
CRUD de documentos, esquemas y contenido. Draft/publicado. Multi-sitio. Auth básica. SDK cliente.
**Entregable:** se puede crear/leer/publicar contenido vía API (con Postman).

### Fase 3 — Frontend conectado + i18n + imágenes · ~22–32 j/d
Frontend que lee de la API. Routing i18n, fallbacks, slugs por idioma. **Servicio de imágenes** (transformación + caché + `srcset` + presets de seguridad). Modo draft + `bridge.js`. SEO multi-idioma. Webhooks de rebuild.
**Entregable:** web pública multilingüe con imágenes optimizadas, alimentada por la API.

### Fase 4 — Editor: edición básica · ~25–35 j/d
Login, árbol de bloques, generador de formularios desde esquema, guardado/autosave, publicación. **Sin** preview visual todavía (se edita por formularios).
**Entregable:** se puede editar y publicar contenido desde la UI.

### Fase 5 — Editor: preview visual e interacciones · ~22–30 j/d
Iframe + protocolo `postMessage` (sec. 8.11), selección por clic y overlays de hover, render en vivo con morph (sec. 9.3), **edición inline de richtext con toolbar** (sec. 8.4) y **reordenar arrastrando dentro del iframe** (sec. 8.2).
**Entregable:** el editor visual estilo Storyblok funciona: clic en un bloque abre su config, el texto se edita en sitio y se reordena arrastrando.

### Fase 6 — Editor: pulido · ~20–28 j/d
Drag & drop en el aside, undo/redo, media library, selector de idioma + estado de traducción, roles, tokens del design system en los campos.
**Entregable:** editor completo y usable por no-técnicos.

### Fase 7 — Endurecido y lanzamiento · ~12–18 j/d
Seguridad, tests E2E, observabilidad, documentación, optimización, despliegue productivo.
**Entregable:** v1 en producción.

### Fase 8 (OPCIONAL) — Colaboración en tiempo real · ~20–30 j/d
Solo si se activa el módulo. Estado colaborativo con Yjs, WebSockets, presencia y cursores. Requiere que el editor ya use "mutaciones discretas" (preparado desde la Fase 4).
**Entregable:** varias personas editan el mismo documento a la vez.

### Resumen del presupuesto

| Fase | j/d (rango) |
|---|---|
| 0. Cimientos | 8–12 |
| 1. Contrato + render | 10–15 |
| 2. API mínima | 15–22 |
| 3. Frontend + i18n + imágenes | 22–32 |
| 4. Editor básico | 25–35 |
| 5. Preview visual e interacciones | 22–30 |
| 6. Editor pulido | 20–28 |
| 7. Lanzamiento | 12–18 |
| **TOTAL v1** | **134–192 j/d** |
| 8. Colaboración tiempo real (opcional) | +20–30 |
| Theme editor editable estilo Shopify (opcional) | +8–12 |
| **TOTAL con ambos opcionales** | **162–234 j/d** |

> v1 (sin opcionales): con 2 personas en paralelo (~40% solapamiento real), aprox. **4 – 6 meses** de calendario. Cada módulo opcional añade calendario aparte. ⚠️ Estos números son una guía para planificar, no una promesa: ajústalos a la experiencia real del equipo tras la Fase 2. La Fase 5 concentra el mayor riesgo técnico (richtext inline + drag en iframe + morph); dale margen.

---

## 15. Riesgos y cómo mitigarlos

| Riesgo | Impacto | Mitigación |
|---|---|---|
| Cambiar el modelo de bloques a mitad | Alto — rompe todo | Congelar el contrato en Fase 1; versionar el esquema. |
| Subestimar el editor visual | Alto | Es el 50%; trocearlo en módulos (sección 8) y demostrar pronto. |
| i18n añadido tarde "a posteriori" | Alto | Está en el modelo desde Fase 0 y en el frontend desde Fase 3. |
| `bridge.js` filtrado a producción | Medio — seguridad | Carga condicional + revisión en CI. |
| Reinventar drag&drop / richtext | Medio — tiempo | Usar dnd-kit y TipTap, no escribir desde cero. |
| Conflictos de edición simultánea | Bajo (v1) | Last-write-wins + aviso. Colaboración real es módulo opcional (Fase 8); el editor se deja preparado con "mutaciones discretas". |
| Servicio de imágenes sin límites | Medio — coste/seguridad | Lista blanca de presets + URLs firmadas; opción de externalizar el motor. |
| Richtext salta al editar (HTML de TipTap ≠ HTML de Astro) | Medio | Definir el formato del richtext una sola vez y usarlo en ambos lados (sec. 8.4). |
| Querer drag aside→iframe en v1 | Medio — tiempo | Cada contexto gestiona su propio drag; añadir bloques con clic. Cruzar la frontera, más adelante (sec. 8.2). |
| Mutaciones no discretas desde el inicio | Alto si llega la colaboración | Modelar cambios como mutaciones desde la Fase 4 (sec. 8.0); barato ahora, carísimo después. |

---

## 16. Definición de "Hecho" (Definition of Done)

Una tarea está terminada cuando:
1. Funciona en local y en staging.
2. Tiene tests donde aplica (lógica crítica siempre).
3. Está tipada (sin `any` sin justificar).
4. Pasa lint y CI en verde.
5. Está documentada (al menos un README o comentario para el siguiente junior).
6. Funciona en **todos los idiomas configurados**, no solo en el maestro.
7. Cumple **accesibilidad** básica (a11y): el HTML de los bloques usa etiquetas semánticas y atributos `alt`/`aria` donde toca; el editor es navegable por teclado. Apunta a WCAG 2.1 AA como referencia.

---

## 17. Bibliografía rápida (qué leer, por orden)

1. **Astro docs** — Content, i18n routing, hybrid rendering, getStaticPaths.
2. **Storyblok docs** — leer su modelo de "components/bloks" y "visual editor bridge" como referencia conceptual del editor y del preview.
3. **dnd-kit** y **TipTap** (incl. *bubble menu*) — tutoriales de inicio.
4. **idiomorph / morphdom** — actualización del DOM por morph (render en vivo del preview).
5. **PostgreSQL jsonb** — guía oficial + índices GIN.
6. **MDN: Window.postMessage** — el protocolo editor↔bridge.
7. **Design tokens / CSS custom properties** y **Shopify theme settings_schema** — para el sistema de diseño y el theme editor opcional.
8. **OWASP** — XSS y sanitización (porque servimos HTML de richtext).

---

*Fin del plan. Revísalo en equipo antes de la Fase 0 y anota cualquier decisión que cambiéis.*
