---
name: create-api-endpoint
description: Use when adding a new HTTP endpoint to the @cms/api backend (Hono). Ensures the layered architecture (routes → service → repo), Zod validation, tenant isolation and tests are followed. Use whenever you would otherwise write a route handler that touches the database. Do NOT use for editor UI or Astro frontend work.
---

# Crear un endpoint de API

Respeta la arquitectura en capas y el aislamiento por tenant. Detalle en `anexo-arquitectura.md` (A.5).

## Pasos

1. **Ubícalo en su módulo de dominio** `packages/api/src/modules/<dominio>/`. Si el dominio no existe, créalo con sus 4 archivos:
   - `<dominio>.routes.ts` — define la ruta y **valida** la entrada con Zod.
   - `<dominio>.service.ts` — la lógica de negocio (reglas, fallbacks, permisos).
   - `<dominio>.repo.ts` — el **único** sitio con consultas Drizzle.
   - `<dominio>.schema.ts` — los esquemas Zod de request/response.

2. **Valida la entrada** con Zod en la ruta antes de tocar el service. Nada de datos sin validar hacia dentro.

3. **Aplica el tenant**: el handler obtiene `{ user, site }` del middleware de auth. La consulta SIEMPRE filtra por `site_id` y pasa por la capa de resolución de tenant. Confía en RLS, pero filtra igualmente.

4. **Reglas de lectura/escritura:**
   - Lectura pública → solo `published_data` y `deleted_at IS NULL`.
   - Escritura → requiere sesión y permiso (admin/editor) del tenant.

5. **Errores**: lanza errores de dominio; deja que el middleware de errores los convierta en JSON uniforme. No devuelvas formatos ad hoc.

6. **Tests**: añade tests del endpoint, incluido un caso que verifique que el tenant A no accede a datos del tenant B.

## Criterios de aceptación
- Entrada validada con Zod; respuesta tipada con `@cms/shared`.
- SQL solo en `*.repo.ts`.
- Filtrado por tenant + test de aislamiento.
- Permisos comprobados en la API, no solo en la UI.
- Errores en formato uniforme.

## Nunca
- No escribas SQL fuera del repo.
- No saltes la validación ni el filtro de tenant "para ir rápido".
- No confíes en que el frontend ya filtró: la API es la frontera de seguridad.
