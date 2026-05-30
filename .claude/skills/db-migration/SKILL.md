---
name: db-migration
description: Use when changing the database schema — adding/altering tables or columns in @cms/api with Drizzle. Ensures migrations are generated (not hand-written), tenant columns and Row-Level Security are handled, and indexes/constraints are correct. Use whenever a task touches packages/api/src/db/schema.ts.
---

# Migración de base de datos

Cambios de esquema con Drizzle, sin romper el multi-tenant. Detalle en `anexo-arquitectura.md` (A.3, A.11).

## Pasos

1. **Edita el esquema Drizzle** en `packages/api/src/db/schema.ts` (nunca la base de datos a mano).

2. **Si la tabla tiene datos por tenant**, debe llevar `site_id` con FK a `sites` y `ON DELETE CASCADE`.

3. **Activa RLS** en la tabla nueva y crea su política de aislamiento por `site_id` (igual que las demás tablas con tenant). Sin esto, la tabla es una fuga entre clientes.

4. **Índices y constraints**: añade los índices de las consultas frecuentes y los `UNIQUE` necesarios (recuerda los índices parciales para lo publicado/no borrado).

5. **Genera y aplica** la migración:
   - `pnpm db:generate` (crea el archivo de migración versionado).
   - `pnpm db:migrate` (la aplica).
   - Revisa el SQL generado antes de commitearlo.

6. **Cambios destructivos** (renombrar/eliminar columnas con datos): escribe la migración de datos correspondiente y pruébala en staging antes de producción.

## Criterios de aceptación
- La migración está versionada en git (no hay cambios manuales en la DB).
- Las tablas con tenant tienen `site_id` + RLS + política.
- Índices/constraints correctos; índices parciales donde aplique.
- Un test confirma que el tenant A no ve datos del tenant B en la tabla nueva.

## Nunca
- No edites tablas directamente en la base de datos.
- No crees una tabla con `site_id` sin su política RLS.
- No hagas cambios destructivos sin migración de datos probada.
