---
name: block-schema-migration
description: Use when changing the schema of an existing block that ALREADY has published content — renaming a field, changing its type, or removing it. Prevents breaking live content. Use whenever a create-block change is not purely additive. Do NOT use for database table migrations (use db-migration).
---

# Migración de esquema de bloque (evolución de contenido)

Cambiar el esquema de un bloque en uso puede dejar el contenido viejo inconsistente. Detalle en `anexo-arquitectura.md` (A.13).

## Primero: clasifica el cambio

- **Aditivo (seguro):** añadir un campo opcional nuevo. No requiere migración; el `.astro` usa un valor por defecto. Prefiere siempre esta vía.
- **Destructivo (peligroso):** renombrar, cambiar de tipo o eliminar un campo con datos. Requiere migración de contenido.

## Pasos para un cambio destructivo

1. **Sube la versión** del esquema del bloque.

2. **Escribe un script de migración de contenido** que recorra los `content` afectados (de todos los tenants) y transforme su `data`:
   - Renombrar: copia el valor del campo viejo al nuevo y elimina el viejo.
   - Cambio de tipo: convierte el valor al nuevo formato.
   - Eliminar: borra el campo huérfano del JSON.

3. **Mantén el render tolerante**: el `.astro` no debe romper si encuentra contenido a medio migrar (usa defaults).

4. **Prueba en staging** con datos reales (o una copia) antes de producción. Verifica una muestra de páginas en el preview.

5. **Aplica en producción** dentro de una ventana controlada y deja registro en el `audit_log`.

## Criterios de aceptación
- Cambios aditivos no requieren script.
- Los destructivos tienen script probado en staging y aplicado a todos los tenants.
- Ninguna página publicada se rompe durante ni después de la migración.

## Nunca
- No hagas un cambio destructivo "en caliente" sin migración: romperás contenido de clientes.
- No asumas un solo tenant: la migración recorre todos.
