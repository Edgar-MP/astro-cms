# CLAUDE.md

Instrucciones para agentes de IA que trabajan en este repositorio. Léelas antes de tocar código. Para el porqué de cada decisión, ve a los documentos enlazados al final.

## Qué es este proyecto

CMS headless basado en bloques con editor visual (estilo Storyblok), para webs en Astro, ofrecido como **SaaS multi-tenant**. Tres piezas (API, editor, frontend Astro) + base de datos. Ver `README.md` y `plan-cms-bloques.md`.

## Estructura del repo (monorepo, pnpm workspaces)

```
packages/shared       tipos y contratos compartidos (TS + Zod)
packages/api          backend Hono + Drizzle + PostgreSQL
packages/sdk          cliente HTTP tipado
packages/editor       SPA React (Vite, Zustand)
packages/blocks-base  bloques genéricos reutilizables
packages/create-web   plantilla de web cliente
```
Las webs cliente son repos Astro aparte que consumen `@cms/sdk` y `@cms/blocks-base`.

## Comandos

- `pnpm install` — instala todo el monorepo.
- `pnpm dev` — arranca API, editor y web de ejemplo.
- `pnpm build` / `pnpm lint` / `pnpm typecheck` / `pnpm test` — por turborepo.
- `pnpm db:migrate` / `pnpm db:seed` / `pnpm db:studio` — base de datos.
- `pnpm --filter <web> push-schemas` — sincroniza esquemas de bloques con la API.

## Convenciones de código

- **TypeScript estricto.** Prohibido `any` sin un comentario que lo justifique.
- Los **tipos compartidos viven en `@cms/shared`**; no los dupliques en otros paquetes.
- Valida toda entrada externa con **Zod** en el borde (rutas de API, mensajes del bridge).
- Nombres: `camelCase` variables/funciones, `PascalCase` componentes y tipos, `kebab-case` archivos no-componente.
- Commits pequeños y descriptivos. No mezcles refactor + feature en el mismo commit.

## Reglas de arquitectura (no las rompas)

- **API en capas:** `routes` (valida) → `service` (reglas) → `repo` (Drizzle). El SQL solo vive en `*.repo.ts`.
- **Editor por mutaciones:** todo cambio de contenido pasa por `applyMutation` sobre el estado, que es la única fuente de verdad. Nunca mutes el DOM del iframe directamente. Ver `plan-cms-bloques.md` sec. 8.0.
- **Render de bloques tolerante:** un `.astro` nunca asume que un campo existe; usa valores por defecto (clave para la evolución de esquemas).
- **Estilos solo con design tokens.** Prohibido escribir píxeles/colores fijos en bloques o componentes (ver `design-system/`).

## Reglas multi-tenant (críticas) 🔒

- **Toda** consulta a datos debe ir filtrada por `site_id`. Nunca devuelvas datos sin tenant.
- Confía en **Row-Level Security**, pero no como única defensa: pasa siempre por la capa de resolución de tenant (`getTenantContext`).
- El público solo accede a `published_data` y no borrado (`deleted_at IS NULL`).
- Nunca registres datos de un tenant en logs de otro.

## Seguridad (no negociable)

- `bridge.js` **nunca** debe acabar en el bundle de producción pública (carga condicional al modo preview).
- Valida siempre `event.origin` en cualquier `postMessage`.
- Sanitiza el HTML de richtext antes de servirlo.
- Secretos solo en variables de entorno; jamás en el repo.

## Tests y Definición de Hecho

Una tarea está hecha cuando: pasa lint/typecheck/CI, tiene tests donde aplica, funciona en todos los idiomas configurados, cumple accesibilidad básica (WCAG 2.1 AA) y está documentada. Ver `plan-cms-bloques.md` sec. 16.

## Skills disponibles (cárgalas cuando apliquen)

Están en `.claude/skills/`. Úsalas en lugar de improvisar el procedimiento:
- **create-block** — crear un bloque de contenido nuevo (componente + esquema + registro).
- **create-api-endpoint** — añadir un endpoint respetando capas, Zod y tenant.
- **create-editor-field** — añadir un tipo de campo al editor.
- **db-migration** — crear/aplicar una migración de base de datos con RLS.
- **block-schema-migration** — migrar contenido al cambiar el esquema de un bloque.

## Nunca hagas esto

- No reinventes drag&drop ni el editor de texto: usa `dnd-kit` y `TipTap`.
- No introduzcas valores de estilo fijos; usa tokens.
- No saltes la validación Zod ni el filtrado por tenant "para ir rápido".
- No cambies el contrato de bloque (`@cms/shared`) sin avisar: rompe las tres piezas.
- No hagas cambios destructivos de esquema sin una migración de contenido probada en staging.

## Dónde mirar

`README.md` (índice) · `plan-cms-bloques.md` (qué y porqué) · `anexo-arquitectura.md` (cómo, DDL, multi-tenancy) · `saas-operaciones.md` (negocio/operación) · `backlog-tareas.md` (qué hacer ahora) · `guia-setup-local.md` (arranque) · `guia-crear-bloque.md` y `design-system/` (autoría de bloques).
