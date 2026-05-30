# Anexo A — Arquitectura técnica detallada

> Complemento del documento `plan-cms-bloques.md`. Aquí se define, al máximo detalle, **cuántos proyectos hay**, **cómo se guardan**, **la base de datos**, **el stack de cada proyecto** y **la arquitectura interna de cada uno**. Pensado para que un junior pueda crear las carpetas y empezar sin dudas.

---

## A.0 La decisión que lo ordena todo: producto vs. consumidor

🎯 El CMS es **un producto reutilizable**. Cada web que construyas es **un consumidor** de ese producto. Por eso hay **dos clases de repositorio** que nunca se mezclan:

- **El core del CMS** (un monorepo): la API, el editor, el SDK y los tipos. Es lo que se construye **una vez** y evoluciona con versionado.
- **Cada web cliente** (un repo Astro por web): consume el SDK del core y **define sus propios bloques**. Es lo que se construye **una vez por web**.

¿Por qué separarlos? Porque si todo viviera junto, cambiar el core rompería todas las webs a la vez, y cada web arrastraría el peso del CMS. Separados, el core publica versiones (`@cms/sdk@1.4.0`) y cada web fija la versión que usa: el core puede avanzar sin romper nada en producción.

---

## A.1 Cuántos proyectos hay (el censo completo)

### Repositorio 1 — `cms-core` (monorepo, se construye 1 vez)

| # | Paquete | Tipo | Responsabilidad |
|---|---|---|---|
| 1 | `@cms/shared` | Librería TS pura | Tipos y contratos compartidos: `Block`, `BlockSchema`, `Field`, `Document`, mensajes del protocolo `postMessage`, validadores Zod. |
| 2 | `@cms/api` | Servicio backend | La API REST + acceso a base de datos + auth + servicio de imágenes. |
| 3 | `@cms/sdk` | Librería TS publicable | Cliente para hablar con la API. Lo usan el editor y **cada web**. |
| 4 | `@cms/editor` | App web (SPA) | El editor visual (el CMS que ve el usuario). |
| 5 | `@cms/blocks-base` | Librería de bloques | Bloques genéricos reutilizables (hero, texto, imagen, grid, botón…) que cada web importa y puede sobrescribir (ver A.6.5). |
| 6 | `@cms/create-web` | Plantilla / CLI | Andamiaje para arrancar una web cliente nueva ya cableada al SDK y a `blocks-base`. (Opcional pero muy recomendable.) |

> La base de datos **no es un proyecto aparte**: su esquema y migraciones viven dentro de `@cms/api` (carpeta `db/`). Se puede extraer a un paquete `@cms/db` si más adelante otro servicio necesita el esquema; no hace falta de entrada.

### Repositorio 2..N — una web cliente por cada web (se construye 1 vez por web)

| Proyecto | Tipo | Responsabilidad |
|---|---|---|
| `web-clienteA`, `web-clienteB`, … | App Astro independiente | La web pública. Define sus bloques (`.astro` + esquema), sus tokens de diseño, sus páginas. Consume `@cms/sdk`. |

**Recuento para planificar:** construyes **6 proyectos una sola vez** (el core, incluida la librería de bloques base) y **1 proyecto por web** (reutilizando la plantilla). La primera web es la que más cuesta; las siguientes son mayormente "definir los bloques propios".

---

## A.2 Cómo se guardan (repos, workspaces, versionado)

### A.2.1 El monorepo del core
- Gestor de paquetes: **pnpm** con `workspaces`.
- Orquestador de tareas: **Turborepo** (cachea builds/tests entre paquetes).
- Estructura raíz:
```
cms-core/
  package.json            # raíz: scripts globales + workspaces
  pnpm-workspace.yaml      # packages/*
  turbo.json               # pipeline de build/test/lint
  tsconfig.base.json       # config TS compartida
  .changeset/              # versionado (ver abajo)
  packages/
    shared/
    api/
    sdk/
    editor/
    create-web/
```
- `pnpm-workspace.yaml`:
```yaml
packages:
  - "packages/*"
```

### A.2.2 Cómo se publican los paquetes reutilizables
`@cms/shared` y `@cms/sdk` se **publican a un registry** para que las webs los instalen. Opciones: **npm privado**, **GitHub Packages** o un **Verdaccio** autoalojado.
- Versionado con **Changesets**: cada cambio relevante añade un "changeset"; al hacer release, sube la versión (semver) y publica.
- ⚠️ Regla semver: cambios que rompen el contrato de la API o de los tipos = **major**. Las webs fijan `@cms/sdk@^1` y actualizan a su ritmo.

### A.2.3 Cómo consume una web cliente
```bash
pnpm add @cms/sdk @cms/shared
```
La web **no** vive en el monorepo del core: es su propio repo Git, su propio despliegue, su propio ciclo de vida.

### A.2.4 Ramas (sugerencia)
- `main` (producción), `develop` (integración), ramas `feature/*`.
- CI por paquete (Turbo solo reconstruye lo que cambió).

---

## A.3 Base de datos al detalle

### A.3.1 Motor y principios
- **PostgreSQL 16.**
- Lo **relacional** (sitios, usuarios, relaciones, slugs, estados) va en **columnas**: permite índices, claves foráneas y consultas eficientes.
- El **árbol de bloques** va en **`jsonb`**: es una estructura anidada y variable que no tiene sentido normalizar en tablas.
- IDs: `uuid` con `gen_random_uuid()`.
- Fechas: `timestamptz` (siempre con zona horaria).

### A.3.2 Esquema completo (DDL)

```sql
-- ─── Sitios y usuarios ───────────────────────────────────────────────
CREATE TABLE sites (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name            text        NOT NULL,
  slug            text        NOT NULL UNIQUE,
  default_locale  text        NOT NULL,                    -- 'es'
  locales         text[]      NOT NULL DEFAULT '{}',       -- ['es','en','ca']
  created_at      timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE users (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  email          text        NOT NULL UNIQUE,
  password_hash  text        NOT NULL,
  name           text        NOT NULL,
  created_at     timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE memberships (
  user_id  uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  site_id  uuid NOT NULL REFERENCES sites(id) ON DELETE CASCADE,
  role     text NOT NULL DEFAULT 'editor',                 -- 'admin' | 'editor'
  PRIMARY KEY (user_id, site_id)
);

-- ─── Esquemas de bloque (registro sincronizado desde el código) ───────
CREATE TABLE block_schemas (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  site_id       uuid        NOT NULL REFERENCES sites(id) ON DELETE CASCADE,
  name          text        NOT NULL,                      -- 'hero'
  display_name  text        NOT NULL,                      -- 'Cabecera'
  fields        jsonb       NOT NULL,                      -- definición de campos
  updated_at    timestamptz NOT NULL DEFAULT now(),
  UNIQUE (site_id, name)
);

-- ─── Documentos (metadatos de cada página) ───────────────────────────
CREATE TABLE documents (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  site_id     uuid        NOT NULL REFERENCES sites(id) ON DELETE CASCADE,
  type        text        NOT NULL DEFAULT 'page',          -- 'page' | 'folder'
  parent_id   uuid        REFERENCES documents(id) ON DELETE CASCADE,
  created_at  timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_documents_site_parent ON documents (site_id, parent_id);

-- ─── Contenido: UNA fila por (documento, locale) ─────────────────────
CREATE TABLE content (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  site_id         uuid        NOT NULL REFERENCES sites(id) ON DELETE CASCADE, -- denormalizado para consultas
  document_id     uuid        NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
  locale          text        NOT NULL,                     -- 'es'
  slug            text        NOT NULL,                     -- slug por idioma: 'contacto'
  draft_data      jsonb       NOT NULL,                     -- árbol de bloques en edición
  published_data  jsonb,                                    -- snapshot publicado (NULL si nunca publicado)
  seo             jsonb       NOT NULL DEFAULT '{}',         -- { title, description, og_image } por página/locale
  publish_at      timestamptz,                              -- publicación programada (NULL = inmediata)
  published_at    timestamptz,
  deleted_at      timestamptz,                              -- papelera / soft delete (NULL = activo)
  updated_at      timestamptz NOT NULL DEFAULT now(),
  updated_by      uuid        REFERENCES users(id),
  UNIQUE (document_id, locale),                             -- un documento, una variante por idioma
  UNIQUE (site_id, locale, slug)                            -- slug único por sitio+idioma
);
-- Lectura pública (solo lo publicado y no borrado), muy frecuente:
CREATE INDEX idx_content_public ON content (site_id, locale, slug)
  WHERE published_data IS NOT NULL AND deleted_at IS NULL;
-- Cola de publicación programada:
CREATE INDEX idx_content_scheduled ON content (publish_at)
  WHERE publish_at IS NOT NULL;

-- ─── Redirecciones (301) al cambiar slugs ────────────────────────────
CREATE TABLE redirects (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  site_id    uuid        NOT NULL REFERENCES sites(id) ON DELETE CASCADE,
  locale     text        NOT NULL,
  from_slug  text        NOT NULL,
  to_slug    text        NOT NULL,
  code       integer     NOT NULL DEFAULT 301,
  created_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (site_id, locale, from_slug)
);

-- ─── Audit log (trazabilidad de acciones) ────────────────────────────
CREATE TABLE audit_log (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  site_id    uuid        NOT NULL REFERENCES sites(id) ON DELETE CASCADE,
  user_id    uuid        REFERENCES users(id),
  action     text        NOT NULL,                          -- 'publish' | 'delete' | 'restore' | …
  target     text        NOT NULL,                          -- 'content:<id>' | 'document:<id>' | …
  meta       jsonb       NOT NULL DEFAULT '{}',
  created_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_site_time ON audit_log (site_id, created_at DESC);

-- ─── Historial de versiones ──────────────────────────────────────────
CREATE TABLE versions (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  content_id  uuid        NOT NULL REFERENCES content(id) ON DELETE CASCADE,
  label       text        NOT NULL,                         -- 'publish' | 'autosave' | 'manual'
  data        jsonb       NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now(),
  author      uuid        REFERENCES users(id)
);
CREATE INDEX idx_versions_content ON versions (content_id, created_at DESC);

-- ─── Assets ──────────────────────────────────────────────────────────
CREATE TABLE assets (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  site_id      uuid        NOT NULL REFERENCES sites(id) ON DELETE CASCADE,
  filename     text        NOT NULL,
  storage_key  text        NOT NULL,                         -- clave en S3/R2
  url          text        NOT NULL,
  mime         text        NOT NULL,
  size         integer     NOT NULL,
  width        integer,
  height       integer,
  alt          jsonb       NOT NULL DEFAULT '{}',            -- { "es": "...", "en": "..." }
  created_at   timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_assets_site ON assets (site_id, created_at DESC);

-- ─── Design tokens editables (opcional, sec. 9.bis del plan) ─────────
CREATE TABLE theme_settings (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  site_id     uuid        NOT NULL REFERENCES sites(id) ON DELETE CASCADE UNIQUE,
  tokens      jsonb       NOT NULL DEFAULT '{}',
  updated_at  timestamptz NOT NULL DEFAULT now()
);
```

### A.3.3 La decisión clave: draft y publicado en la MISMA fila
🎯 Cada `(documento, locale)` es **una sola fila** con dos columnas de contenido:
- `draft_data`: lo que se está editando (lo lee el editor y el modo preview).
- `published_data`: lo que ve el público (lo lee el frontend en producción).

**Publicar** = copiar `draft_data` → `published_data`, fijar `published_at` y registrar una `version` con `label='publish'`.

El **estado** se deriva, no se guarda como campo:
- Sin publicar nunca → `published_data IS NULL`.
- Publicado y sin cambios → `draft_data = published_data`.
- Publicado con cambios pendientes → `draft_data <> published_data`.

⚠️ Ventaja sobre tener dos filas: no hay que sincronizar dos registros ni decidir "cuál es cuál". Una fila, dos snapshots.

### A.3.4 Dónde viven los esquemas de bloque (importante)
🎯 Los esquemas **se definen en el código de cada web** (junto al `.astro`, ver A.6) y se **sincronizan** a la tabla `block_schemas` con un comando (`cms push-schemas`). Es decir, la tabla es un **registro/caché** que alimenta al editor; la **fuente de verdad es el código**. Así el desarrollador crea bloques en su repo (versionados con git) y el editor los conoce sin tocar la base de datos a mano.

### A.3.5 Migraciones y seed
- Migraciones con **Drizzle Kit** (`drizzle-kit generate` + `migrate`). Cada cambio de esquema = una migración versionada en git.
- Seed inicial: crea un `site` por defecto, un `user` admin y su `membership`.

> **Para aprender:** "PostgreSQL jsonb", "GIN / partial index", "Drizzle Kit migrations", "uuid gen_random_uuid", "database seeding".

---

## A.4 Arquitectura de `@cms/shared`

Librería TS pura (sin runtime). Es el **contrato** entre las tres piezas: si cambias un tipo aquí, la compilación rompe donde haga falta (eso es bueno).

```
packages/shared/src/
  block.ts         # Block { _uid, component, [campo]: valor }
  field.ts         # FieldType union (incluye 'variant') + showIf en la interfaz base de campo
  schema.ts        # BlockSchema { name, display_name, fields, presets? }
  document.ts      # Document, Content, Locale, Status
  messages.ts      # BridgeMessage: unión de los mensajes postMessage (sec. 8.11)
  tokens.ts        # tipos de design tokens
  zod/             # validadores Zod reutilizados por api y editor
    block.zod.ts
    schema.zod.ts
  index.ts         # re-exporta todo
```
- Solo **tipos y validadores**, cero dependencias pesadas.
- Incluye el tipo de campo `variant`, la propiedad `showIf` (campos condicionales) y `presets` en el esquema (sec. 6.4 del plan).
- Stack: TypeScript + Zod.

---

## A.5 Arquitectura de `@cms/api`

### A.5.1 Estilo arquitectónico
**Monolito modular en capas**, organizado por **dominio** (vertical slices). Cada módulo tiene sus rutas, su lógica y su acceso a datos. Capas: `routes` → `service` → `repo` → `db`.

### A.5.2 Estructura de carpetas
```
packages/api/src/
  index.ts                 # arranca Hono, monta middlewares y módulos
  config.ts                # lee y valida env vars con Zod
  db/
    client.ts              # conexión Drizzle a Postgres
    schema.ts              # tablas Drizzle (espejo del DDL de A.3)
    migrations/
  middleware/
    auth.ts                # valida sesión/JWT → inyecta { user, site }
    error.ts               # captura errores → respuesta JSON uniforme
    cors.ts
  modules/
    auth/                  # login, logout, me
    schemas/               # push y lectura de block_schemas
    documents/             # CRUD de documentos
    content/               # leer draft/publicado, guardar, publicar, versiones
      content.routes.ts    # endpoints + validación Zod del input
      content.service.ts   # reglas de negocio (publicar, fallback i18n…)
      content.repo.ts      # consultas Drizzle
    assets/                # subida + metadatos
    images/                # servicio de transformación (sharp) + caché
    theme/                 # theme_settings
    preview/               # endpoint POST /preview/render (recibe árbol, devuelve HTML*)
  lib/
    sanitize.ts            # sanitiza HTML de richtext antes de servir
    slug.ts
    storage.ts             # wrapper S3/R2
```
> *El render de preview puede vivir en la API (invocando un render de bloques en servidor) o delegarse al propio frontend Astro vía un endpoint suyo; decide según dónde sea más simple renderizar `.astro`. En Astro suele ser más natural que el endpoint de render esté **en la web cliente** (sec. A.6).

### A.5.3 Flujo de una petición (ejemplo: leer una página publicada)
```
GET /sites/:site/content?slug=contacto&locale=es&status=published
  → cors → auth (público: solo lectura publicada)
  → content.routes  : valida query con Zod
  → content.service : aplica fallback i18n si falta el locale
  → content.repo    : SELECT ... WHERE site_id, locale, slug, published_data NOT NULL
  → responde { component, body: [...] }  (tipado con @cms/shared)
```

### A.5.4 Reglas y patrones
- **Repository pattern**: las consultas Drizzle viven solo en `*.repo.ts`. El service no sabe SQL.
- **Validación en el borde**: todo input se valida con Zod en `*.routes.ts` antes de entrar.
- **Auth**: middleware que resuelve usuario y sitio; el público solo accede a `published_data`.
- **Errores**: un único handler convierte excepciones en `{ error, code }` con el status correcto.
- Stack: **Hono + TypeScript + Drizzle + PostgreSQL + Zod + sharp** (imágenes) + **Lucia/JWT** (auth).

> **Para aprender:** "Hono framework", "repository pattern", "service layer", "Zod request validation", "Drizzle queries".

---

## A.6 Arquitectura de una web cliente (proyecto Astro)

### A.6.1 Principio
Cada bloque es **un componente `.astro` + su esquema**, juntos en la misma carpeta. El esquema es la **fuente de verdad** (se sincroniza a la API con `push-schemas`); el `.astro` es su implementación visual.

### A.6.2 Estructura de carpetas
```
web-clienteA/                  # repo independiente
  astro.config.mjs             # output híbrido + i18n + adaptador (Node/CF/Vercel)
  package.json                 # depende de @cms/sdk, @cms/shared
  cms.config.ts                # { apiUrl, siteId, locales, previewToken }
  src/
    blocks/
      hero/
        Hero.astro             # el componente
        hero.schema.ts         # export const schema: BlockSchema
      richtext/
        Richtext.astro
        richtext.schema.ts
      grid/ …
      registry.ts              # blockMap (component→.astro) + lista de esquemas
    components/
    layouts/
    pages/
      [...slug].astro          # render público por slug (prerender o SSR)
      preview/[...slug].astro   # render DRAFT + inyecta el bridge (solo preview)
    lib/
      content.ts               # usa @cms/sdk para pedir contenido
      Blocks.astro             # render recursivo del árbol (plan, sec. 9.1)
    styles/
      tokens.css               # design tokens como variables CSS (sec. 9.bis)
    bridge/
      bridge.ts                # el bridge.js: postMessage + morph (solo preview)
  scripts/
    push-schemas.ts            # recorre src/blocks/**/*.schema.ts y los sube a la API
```

### A.6.3 Cómo se conecta con el core
- **Lectura de contenido:** `lib/content.ts` usa `@cms/sdk` → `getPublished(slug, locale)` (público) o `getDraft(...)` (preview).
- **Esquemas:** `pnpm push-schemas` sube los `*.schema.ts` a `block_schemas` para que el editor los muestre.
- **Preview:** la ruta `preview/[...slug].astro` corre en SSR, pide `draft_data` e incluye `bridge.ts`.

### A.6.4 Render híbrido por ruta
```astro
---
// pages/[...slug].astro
export const prerender = true;   // estática; pon false para SSR según la web/tipo
---
```
Stack: **Astro + TypeScript + @cms/sdk + idiomorph** (en el bridge) + el design system de esa web.

### A.6.5 Bloques base compartidos vs. bloques propios ⭐
🎯 Para no reescribir el "hero" o el "texto" en cada web, hay **dos orígenes de bloques**:
- **`@cms/blocks-base`** (paquete del core): bloques genéricos reutilizables (hero, texto, imagen, grid, botón…), cada uno con su `.astro` + `schema.ts`.
- **`src/blocks/` de cada web:** los bloques específicos de esa web.

El `registry.ts` de cada web **combina** ambos conjuntos y puede **sobrescribir** un bloque base por uno propio con el mismo `name` (por ejemplo, un hero a medida). Al ejecutar `push-schemas`, se sube a `block_schemas` el **conjunto efectivo** del sitio (base + propios + overrides), con su `site_id`. Así:
- Lo común no se repite (vive en `blocks-base`).
- Cada proyecto tiene libertad total (añade y sobrescribe).
- El editor de cada tenant ve exactamente los bloques de su web.

```ts
// src/blocks/registry.ts
import * as base from "@cms/blocks-base";
import Hero from "./hero/Hero.astro";          // override propio
import { schema as heroSchema } from "./hero/hero.schema";

export const blockMap = { ...base.blockMap, hero: Hero };       // 'hero' sobrescrito
export const schemas  = { ...base.schemas,  hero: heroSchema };
```

> **Para aprender:** "Astro hybrid output adapter", "Astro getStaticPaths", "import.meta.glob" (para recorrer los schemas), "package exports / overrides".

---

## A.7 Arquitectura de `@cms/sdk`

Cliente fino y tipado. Lo usan el editor y cada web.
```
packages/sdk/src/
  client.ts        # createClient({ apiUrl, token? }) → instancia con fetch configurado
  content.ts       # getPublished, getDraft, save, publish, listVersions
  schemas.ts       # pushSchemas, getSchemas
  assets.ts        # upload, list
  theme.ts         # getTheme, saveTheme
  index.ts
```
- Devuelve tipos de `@cms/shared`.
- Maneja `draft` vs `published` con un parámetro, nunca mezcla.
- Stack: TypeScript + `fetch` (sin dependencias pesadas, para que pese poco en las webs).

---

## A.8 Arquitectura de `@cms/editor` (SPA React)

### A.8.1 Estilo
**Feature-based** (carpeta por funcionalidad) + **estado centralizado** con el modelo de mutaciones del plan (sec. 8.0).

### A.8.2 Estructura de carpetas
```
packages/editor/src/
  main.tsx
  app/
    routes.tsx               # react-router
    providers.tsx
  features/
    auth/
    document-tree/           # árbol del aside (plan 8.1–8.2)
    field-forms/             # generador de formularios (plan 8.3)
      FieldRenderer.tsx       # mapea field.type → componente (recursivo)
      fields/
        TextField.tsx
        RichtextField.tsx
        AssetField.tsx
        BloksField.tsx        # recursivo: vuelve a FieldRenderer
        TokenField.tsx        # select de design tokens
    preview/                 # iframe + cliente del protocolo (plan 8.5, 8.11, 9.3)
      PreviewFrame.tsx
      bridgeClient.ts         # postMessage tipado (usa @cms/shared messages)
    media/                   # media library (plan 8.8)
    i18n-locale/             # selector de idioma (plan 8.6)
  state/
    documentStore.ts         # estado del documento + applyMutation()
    mutations.ts             # creadores de mutaciones: insertBlock, moveBlock, updateField…
    historyStore.ts          # undo/redo a partir de las mutaciones (plan 8.7)
    uiStore.ts               # selección, hover, panel abierto, locale activo
  lib/
    api.ts                   # instancia de @cms/sdk
  components/ui/             # design system del editor (independiente del de las webs)
```

### A.8.3 El núcleo: estado por mutaciones
- `documentStore` guarda el árbol de bloques actual y expone `applyMutation(m)`.
- `mutations.ts` define cada cambio como un objeto con nombre (p. ej. `{ type: 'moveBlock', uid, toIndex }`).
- `historyStore` apila las mutaciones (con su inversa) → undo/redo gratis.
- `bridgeClient` traduce mensajes del iframe a mutaciones y manda el `render` de vuelta.
- ⚠️ Toda interacción (aside, formularios, iframe) termina en **una mutación**. No hay otra forma de cambiar el contenido.

Stack: **React + Vite + TypeScript + Zustand + react-router + dnd-kit + TipTap + react-hook-form + @cms/sdk**.

> **Para aprender:** "feature-based React architecture", "Zustand", "command/mutation pattern", "react-router", "Vite".

---

## A.9 Tabla resumen: stack por proyecto

| Proyecto | Lenguaje | Framework/Runtime | Datos/Estado | Librerías clave | Despliegue |
|---|---|---|---|---|---|
| `@cms/shared` | TS | — | — | Zod | (paquete) |
| `@cms/api` | TS | Hono / Node | PostgreSQL + Drizzle | Zod, sharp, Lucia/JWT | Docker en PaaS/VPS |
| `@cms/sdk` | TS | — | — | fetch | registry (npm priv.) |
| `@cms/editor` | TS | React + Vite | Zustand | dnd-kit, TipTap, react-router, react-hook-form | estático (Pages) |
| web cliente | TS | Astro (híbrido) | — | @cms/sdk, idiomorph | estático o SSR |
| Base de datos | SQL | PostgreSQL 16 | — | — | gestionada |

---

## A.10 Cómo encaja todo (dependencias)

- **Build-time (qué importa a qué):** `api`, `sdk` y `editor` dependen de `shared`. `editor` y cada `web` dependen de `sdk`.
- **Run-time (quién llama a quién):** el `editor` y cada `web` llaman por HTTP a la `api` central. `api` habla con PostgreSQL y con S3/R2. El preview de la `web` y el `editor` se comunican por `postMessage`.
- **Sincronización de esquemas:** cada `web` empuja sus `*.schema.ts` a `api` (`push-schemas`), y el `editor` los lee desde `api`.

---

## A.11 Multi-tenancy (modelo SaaS) ⭐

🎯 **Modelo: SaaS multi-tenant gestionado.** Tú operas una plataforma central (api + editor + DB + storage) y cada cliente es un **tenant** que accede a su espacio. El aislamiento es por software, no por instancia.

### A.11.1 Estrategia de tenancy
- **Pool compartido con discriminador por columna:** una sola base de datos; cada tabla lleva `site_id` (el tenant). Todas las consultas filtran por él. Es lo barato y simple, y para un CMS aguanta de sobra cientos de tenants (recuerda: el tráfico público se sirve cacheado y no golpea la DB; solo llegan ediciones e invalidaciones).
- **Capa de resolución de tenant:** todas las consultas pasan por `getTenantContext(siteId)`, que hoy devuelve la conexión compartida. Mantenerla desde el día 1 permite, más adelante, **promover un cliente concreto a su propia DB** sin reescribir la lógica de negocio (modelo híbrido).

### A.11.2 Aislamiento robusto con Row-Level Security (RLS)
⚠️ No confíes solo en acordarte del `WHERE site_id`: actívalo en la base de datos con **RLS de PostgreSQL**, para que un tenant no pueda ver filas de otro aunque una consulta esté mal escrita.
```sql
-- 1. La conexión de la app fija el tenant actual en cada petición:
SET app.current_site = '<site_id>';

-- 2. Se activa RLS y se define la política en cada tabla con site_id:
ALTER TABLE content ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON content
  USING (site_id = current_setting('app.current_site')::uuid);
-- (repetir para documents, block_schemas, assets, theme_settings, versions vía content)
```
- El middleware `auth.ts` (sec. A.5.2) resuelve el `site_id` del usuario/token y ejecuta el `SET app.current_site` al abrir la transacción.
- Las lecturas **públicas** (frontend) usan un rol con permiso solo de lectura de `published_data`.

### A.11.3 Escalado, por orden de coste
1. **Escala vertical** del Postgres (más CPU/RAM): da muchísimo de sí para un CMS.
2. **Réplicas de lectura** si las lecturas crecen.
3. **API stateless → escala horizontal:** varias instancias detrás de un balanceador (el estado vive en Postgres y S3).
4. **DB dedicada por cliente** solo cuando un tenant lo justifique (volumen, cumplimiento): se hace vía la capa de resolución, sin tocar a los demás.

> **Para aprender:** "PostgreSQL Row-Level Security", "multi-tenant data architecture (pool vs silo)", "stateless API horizontal scaling", "read replicas".

---

## A.12 Despliegue e infraestructura (SaaS)

| Pieza | Dónde corre | Notas |
|---|---|---|
| `api` | Contenedor Docker en PaaS (Railway/Fly/Render) o VPS, ≥1 instancia tras balanceador | Stateless; escala horizontal. |
| PostgreSQL | Servicio gestionado (Supabase/Neon/RDS) | Backups automáticos + RLS activado. |
| `editor` | Estático en CDN (Cloudflare/Netlify/Vercel Pages) | SPA; apunta a la `api`. |
| Storage | S3 / Cloudflare R2 | Assets + variantes de imagen. |
| Servicio de imágenes | Dentro de `api` o como worker aparte + CDN delante | Caché de variantes. |
| webs cliente | Estático (CDN) o SSR (adaptador) según la web | Consumen la `api` central. |

- **Entornos:** `local` → `staging` → `production`, con su propia base de datos cada uno.
- **CI/CD:** GitHub Actions; Turborepo solo reconstruye lo que cambió. Migraciones de DB como paso del pipeline de release.
- **Secretos:** variables de entorno gestionadas por el proveedor; nunca en el repo.
- **Observabilidad:** logs estructurados + Sentry + métricas básicas (latencia, errores por tenant).

> **Para aprender:** "Docker compose", "GitHub Actions CI/CD", "managed PostgreSQL", "Sentry", "infrastructure as code".

---

## A.13 Evolución de esquemas y migración de contenido ⭐

⚠️ El problema más espinoso de un CMS de bloques: cuando cambias el esquema de un bloque que **ya tiene contenido publicado**, el contenido viejo puede quedar inconsistente.

### A.13.1 Tipos de cambio y su riesgo
- **Aditivo (seguro):** añadir un campo nuevo opcional. El contenido viejo simplemente no lo tiene; el `.astro` usa un valor por defecto.
- **Renombrar / cambiar de tipo (peligroso):** el contenido viejo apunta a un campo que ya no existe o tiene otro formato.
- **Eliminar (peligroso):** el dato se queda huérfano en el JSON.

### A.13.2 Estrategia
1. **Versiona el esquema:** cada `block_schema` lleva un número de versión; el contenido guarda con qué versión se creó.
2. **Migraciones de contenido:** un script que recorre el `data` de los contenidos afectados y los transforma (renombrar campo, rellenar defaults, etc.), igual que una migración de base de datos pero sobre el JSON de bloques. Storyblok llama a esto "migrations".
3. **Tolerancia en el render:** el `.astro` nunca asume que un campo existe; usa defaults. Así un desajuste temporal no rompe la web.
4. **Regla de oro:** prefiere cambios **aditivos**; los destructivos exigen migración escrita y probada en staging antes de producción.

> **Para aprender:** "schema migration", "data migration scripts", "backward compatible schema changes", "Storyblok migrations".

---

## A.14 Caché y entrega de contenido

- **Contenido publicado = cacheable.** Se sirve con cabeceras de caché y, idealmente, desde un CDN. Así el tráfico del público **no golpea la base de datos** (clave para el escalado, sec. A.11).
- **Invalidación al publicar:** publicar dispara la invalidación de la caché/CDN de las rutas afectadas y, si la web es estática, su rebuild (webhooks, plan sec. 9.4).
- **Tokens de acceso a contenido:** las webs leen contenido con un **token público de solo lectura** (publicado); el preview usa un **token de preview** (draft). Nunca se mezclan.
- **Draft = sin caché.** El preview siempre pide datos frescos.

> **Para aprender:** "HTTP cache-control", "CDN cache invalidation", "stale-while-revalidate", "API tokens scopes".

---

*Fin del anexo. Léelo junto al plan principal; este define el "cómo se construye", el plan define el "qué y en qué orden".*
