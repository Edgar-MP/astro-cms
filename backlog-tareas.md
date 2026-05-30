# Backlog de tareas — CMS de bloques SaaS

> Lista de trabajo **ordenada** para construir el proyecto. Pensada para que alguien sin contexto pueda coger una tarea de arriba, entenderla y completarla. Cada tarea tiene: **descripción**, **criterios de aceptación** (cómo sabes que está hecha), **depende de** y **estimación** en jornadas (j/d).
>
> Reglas de uso: haz las tareas **en orden**; no empieces una sin tener hechas sus dependencias; una tarea está cerrada solo cuando cumple **todos** sus criterios de aceptación y la "Definición de Hecho" (ver plan, sec. 16).
>
> Notación de IDs: `E#` = épica, `T###` = tarea.

---

## E0 — Cimientos del proyecto

### T001 · Crear el monorepo ✅
Inicializa `cms-core` con pnpm workspaces y Turborepo.
- **Criterios:** `pnpm install` funciona; existen `packages/{shared,api,sdk,editor}`; `pnpm turbo run build` no falla (aunque no haga nada aún); `tsconfig.base.json` compartido.
- **Depende de:** —
- **Estimación:** 1 j/d

### T002 · Configurar TypeScript, lint y formato ✅
ESLint + Prettier + tsconfig estricto en todos los paquetes.
- **Criterios:** `pnpm lint` y `pnpm typecheck` pasan en verde; `strict: true` activado; un commit con error de tipo falla en CI.
- **Depende de:** T001
- **Estimación:** 1 j/d

### T003 · CI básica (GitHub Actions) ✅
Pipeline que instala, lintea, typechecks y testea en cada PR.
- **Criterios:** un PR muestra los checks; Turbo cachea entre jobs; un fallo de lint bloquea el merge.
- **Depende de:** T002
- **Estimación:** 1 j/d

### T004 · Levantar PostgreSQL local con Docker
`docker-compose.yml` con Postgres 16 para desarrollo.
- **Criterios:** `docker compose up -d` levanta Postgres; se conecta con las credenciales del `.env.example`; datos persisten entre reinicios.
- **Depende de:** T001
- **Estimación:** 0.5 j/d

### T005 · Paquete `@cms/shared` con los tipos base
Define `Block`, `Field`, `FieldType`, `BlockSchema`, `Document`, `Content`, `Locale`, `Status`.
- **Criterios:** los tipos compilan y se importan desde otro paquete; documentados con comentarios; sin dependencias de runtime salvo Zod.
- **Depende de:** T002
- **Estimación:** 2 j/d

---

## E1 — El contrato de bloques y el render (sin API todavía)

### T010 · Definir el formato de bloque y los tipos de campo
Cierra el contrato: `_uid`, `component`, y los tipos de campo de la v1 (sec. 6.2 del plan).
- **Criterios:** documento de referencia de cada tipo de campo; validador Zod por tipo en `@cms/shared`; ejemplos en JSON válidos e inválidos cubiertos por tests.
- **Depende de:** T005
- **Estimación:** 2 j/d

### T011 · Crear la web cliente de ejemplo (Astro)
Proyecto Astro con output híbrido + i18n configurado.
- **Criterios:** `pnpm dev` arranca; rutas `/` (es) y `/en/...` funcionan; adaptador SSR configurado.
- **Depende de:** T010
- **Estimación:** 1.5 j/d

### T012 · Componente `<Blocks>` de render recursivo
Mapa `component → .astro` y render del árbol, incluido el anidamiento.
- **Criterios:** dado un JSON de bloques escrito a mano, renderiza HTML correcto; los `bloks` anidados se renderizan recursivamente; cada bloque lleva `data-blok-uid`; un `component` desconocido no rompe la página.
- **Depende de:** T011
- **Estimación:** 3 j/d

### T013 · Primeros bloques de ejemplo (hero, richtext, grid, button)
Un `.astro` + su `*.schema.ts` por bloque. El hero incluye **variantes** y campos condicionales (`showIf`).
- **Criterios:** los 4 bloques renderizan; cada uno tiene su esquema tipado; el hero define al menos 2 variantes con sus campos condicionales; el `registry.ts` los expone.
- **Depende de:** T012
- **Estimación:** 3 j/d

### T015 · Librería de bloques base `@cms/blocks-base`
Extrae los bloques genéricos reutilizables a un paquete y haz que el `registry.ts` de la web combine base + propios con override (Anexo A.6.5).
- **Criterios:** una web importa `@cms/blocks-base`; puede añadir bloques propios; puede sobrescribir un bloque base por uno con el mismo `name`; `push-schemas` sube el conjunto efectivo.
- **Depende de:** T013
- **Estimación:** 2 j/d

### T014 · Design tokens como variables CSS
`tokens.css` con paleta, espaciado, tipografía; los bloques los usan.
- **Criterios:** cambiar un token cambia todos los bloques; los campos `token` del esquema referencian valores reales; nada de píxeles "a mano" en los bloques.
- **Depende de:** T013
- **Estimación:** 1.5 j/d

---

## E2 — API y base de datos (multi-tenant)

### T020 · Esquema de base de datos con Drizzle
Implementa las tablas del Anexo A.3 con Drizzle + migraciones.
- **Criterios:** `drizzle-kit generate` y `migrate` crean todas las tablas; FKs e índices presentes; `UNIQUE(document_id, locale)` y `UNIQUE(site_id, locale, slug)` funcionan.
- **Depende de:** T004, T005
- **Estimación:** 3 j/d

### T021 · Activar Row-Level Security (RLS)
Políticas de aislamiento por `site_id` (Anexo A.11).
- **Criterios:** con `app.current_site` fijado, una consulta no devuelve filas de otro tenant; un test demuestra que el tenant A no ve datos del tenant B; rol de lectura pública solo accede a `published_data`.
- **Depende de:** T020
- **Estimación:** 2 j/d

### T022 · Esqueleto de la API con Hono + capas
Estructura `routes → service → repo`, config con Zod, middleware de errores y CORS.
- **Criterios:** `GET /health` responde 200; un error lanzado en un service sale como JSON uniforme; las env vars se validan al arrancar.
- **Depende de:** T020
- **Estimación:** 2 j/d

### T023 · Auth: login, sesión y resolución de tenant
Login con email/contraseña, sesión (Lucia/JWT) y middleware que inyecta `{ user, site }` + fija `app.current_site`.
- **Criterios:** login devuelve sesión válida; `GET /auth/me` funciona; una petición sin sesión a rutas privadas devuelve 401; el `site_id` se resuelve y se aplica a la conexión.
- **Depende de:** T021, T022
- **Estimación:** 4 j/d

### T024 · CRUD de documentos
Crear, listar, leer y borrar documentos (metadatos).
- **Criterios:** endpoints validados con Zod; respetan el tenant; tests de cada endpoint.
- **Depende de:** T023
- **Estimación:** 3 j/d

### T025 · Contenido: leer draft/publicado, guardar, publicar
Lógica de `draft_data`/`published_data` y publicación (Anexo A.3.3).
- **Criterios:** guardar actualiza `draft_data`; publicar copia a `published_data`, fija `published_at` y crea una `version`; leer con `status=published` no devuelve borradores; fallback de i18n al `default_locale` cuando falta el locale.
- **Depende de:** T024
- **Estimación:** 4 j/d

### T026 · Esquemas de bloque: push y lectura
Endpoints para recibir esquemas desde el código (`push-schemas`) y servirlos al editor.
- **Criterios:** `POST` de esquemas hace upsert por `(site_id, name)`; `GET` los devuelve; validados con Zod.
- **Depende de:** T023
- **Estimación:** 2 j/d

### T027 · Historial de versiones
Listar y restaurar versiones de un contenido.
- **Criterios:** cada publicación/autosave crea una versión; se pueden listar y restaurar; restaurar crea una nueva versión (no borra historial).
- **Depende de:** T025
- **Estimación:** 2 j/d

### T028 · `@cms/sdk` (cliente)
Cliente tipado: `getPublished`, `getDraft`, `save`, `publish`, `getSchemas`, `pushSchemas`.
- **Criterios:** funciones tipadas con `@cms/shared`; maneja `draft` vs `published`; tests contra la API real (o mock); publicable a un registry.
- **Depende de:** T025, T026
- **Estimación:** 2 j/d

---

## E3 — Frontend conectado, i18n e imágenes

### T030 · La web consume contenido de la API
`lib/content.ts` usa el SDK; las páginas renderizan contenido publicado real.
- **Criterios:** la home renderiza desde la API; SSG con `getStaticPaths` para rutas estáticas; SSR donde se marque; 404 si el slug no existe.
- **Depende de:** T028, T012
- **Estimación:** 3 j/d

### T031 · Routing i18n y slugs por idioma
Prefijos de idioma, slug por locale y fallback.
- **Criterios:** `/contacto` (es) y `/en/contact` (en) funcionan; si falta una traducción publicada, cae al idioma maestro; `hreflang` presente.
- **Depende de:** T030
- **Estimación:** 3 j/d

### T032 · Script `push-schemas`
Recorre `src/blocks/**/*.schema.ts` y los sube a la API.
- **Criterios:** `pnpm push-schemas` sincroniza; el editor ve los esquemas tras ejecutarlo; idempotente.
- **Depende de:** T026, T013
- **Estimación:** 1.5 j/d

### T033 · Servicio de imágenes
Transformación on-the-fly (`sharp`) + caché + `srcset` + presets de seguridad (Anexo, plan sec. 11.2).
- **Criterios:** `/img/<id>?w=800&format=webp` devuelve la variante; la 2ª petición sale de caché; solo se permiten presets en lista blanca; el componente de imagen del frontend genera `srcset`.
- **Depende de:** T023
- **Estimación:** 6 j/d

### T034 · Webhooks de revalidación/rebuild
Al publicar, invalidar caché o disparar rebuild de las páginas afectadas.
- **Criterios:** publicar dispara el webhook; las páginas estáticas afectadas se regeneran (o la caché SSR se invalida); con reintentos si falla.
- **Depende de:** T025, T030
- **Estimación:** 3 j/d

### T035 · SEO por página y redirecciones
Campos `seo` por (documento, locale) volcados al `<head>`; sitemap por idioma; tabla `redirects` y su resolución en el frontend; alta automática de redirect al cambiar un slug publicado (plan sec. 6.5).
- **Criterios:** title/description/og editables por idioma y presentes en el HTML; cambiar un slug crea un 301 y la URL vieja redirige; `sitemap.xml` y `hreflang` correctos.
- **Depende de:** T025, T031
- **Estimación:** 4 j/d

### T036 · Publicación programada (scheduling)
Campo `publish_at` + trabajo periódico (cron) que publica lo vencido (plan sec. 6.5).
- **Criterios:** programar una publicación futura funciona; el cron publica al llegar la hora y crea su `version`; cancelar/editar la programación es posible.
- **Depende de:** T025
- **Estimación:** 2.5 j/d

### T037 · Emails transaccionales
Integración con Resend/Postmark: invitación de usuario, reset de contraseña, verificación, avisos (saas-operaciones.md, sec. 6).
- **Criterios:** los emails se envían y llegan; plantillas multi-idioma; secretos en variables de entorno; reintentos ante fallo del proveedor.
- **Depende de:** T023
- **Estimación:** 3 j/d

---

## E4 — Editor: edición básica (sin preview visual)

### T040 · App del editor (React + Vite) + login
Esqueleto, routing, providers, pantalla de login contra la API.
- **Criterios:** login funciona; sesión persiste; rutas protegidas; layout base con aside + zona central + panel derecho.
- **Depende de:** T028
- **Estimación:** 3 j/d

### T041 · Estado por mutaciones (documentStore)
`documentStore` + `mutations.ts` con `insertBlock`, `moveBlock`, `updateField`, `deleteBlock` (plan sec. 8.0).
- **Criterios:** toda modificación pasa por `applyMutation`; el estado es la fuente de verdad; tests de cada mutación.
- **Depende de:** T040
- **Estimación:** 3 j/d

### T042 · Árbol de bloques en el aside
Vista jerárquica: seleccionar, duplicar, borrar.
- **Criterios:** refleja el estado; seleccionar emite `select`; duplicar/borrar emiten su mutación; muestra el anidamiento.
- **Depende de:** T041
- **Estimación:** 4 j/d

### T043 · Generador de formularios desde el esquema
`FieldRenderer` + un componente por tipo de campo; `bloks` recursivo; `token` como select; `variant` como selector destacado; **campos condicionales `showIf`** (plan sec. 6.4).
- **Criterios:** cada tipo de campo pinta su input; editar un campo emite `updateField`; `bloks` anida formularios; `token` solo ofrece valores del design system; el selector de `variant` cambia qué campos se muestran según `showIf`.
- **Depende de:** T042
- **Estimación:** 9–13 j/d

### T044 · Guardado, autosave y publicación
Autosave con debounce; publicar por locale; indicador de estado (borrador/publicado/cambios sin publicar).
- **Criterios:** autosave guarda `draft_data`; publicar llama al endpoint; el indicador refleja el estado derivado correctamente.
- **Depende de:** T043
- **Estimación:** 5 j/d

### T045 · Undo / redo
Historial sobre las mutaciones.
- **Criterios:** Ctrl+Z / Ctrl+Y deshacen/rehacen; el preview y el árbol reflejan el cambio; límite de pila razonable.
- **Depende de:** T041
- **Estimación:** 4 j/d

---

## E5 — Editor: preview visual e interacciones

### T050 · Iframe de preview + protocolo postMessage
`PreviewFrame` + `bridgeClient` tipado (plan sec. 8.11). Ruta de preview en la web + `bridge.ts`.
- **Criterios:** el iframe carga la web en modo draft; `ready`/`select`/`hover` llegan al editor; `event.origin` validado en ambos lados; protocolo versionado.
- **Depende de:** T044, T030
- **Estimación:** 6 j/d

### T051 · Render en vivo con morph
Endpoint `/preview/render` (recibe árbol, devuelve HTML) + morph con idiomorph en el bridge (plan sec. 9.3).
- **Criterios:** editar un campo actualiza el preview sin recargar y sin perder el scroll; el endpoint no pasa por la BD; debounce aplicado.
- **Depende de:** T050
- **Estimación:** 5 j/d

### T052 · Selección y overlays de hover
Clic selecciona y abre el panel; hover resalta; overlays dibujados dentro del iframe.
- **Criterios:** clic en un bloque abre su config; hover lo resalta; los overlays no se filtran a producción.
- **Depende de:** T050
- **Estimación:** 4 j/d

### T053 · Edición inline de richtext con toolbar
TipTap montado dentro del iframe sobre el nodo richtext; bubble menu (plan sec. 8.4).
- **Criterios:** clic en richtext permite editar in situ; toolbar al seleccionar; el HTML de TipTap coincide con el de Astro (no "salta"); al salir emite `updateField`.
- **Depende de:** T051
- **Estimación:** 8 j/d

### T054 · Reordenar arrastrando dentro del iframe
Drag propio del bridge sobre los `data-blok-uid` + indicador de drop (plan sec. 8.2).
- **Criterios:** arrastrar reordena; al soltar emite `reorder`; el indicador muestra dónde caerá; funciona con bloques anidados.
- **Depende de:** T052
- **Estimación:** 6 j/d

---

## E6 — Editor: pulido

### T060 · Drag & drop en el aside (dnd-kit)
Reordenar y añadir bloques desde el árbol.
- **Criterios:** reordenar en el árbol emite `moveBlock`; añadir un bloque nuevo funciona; accesible por teclado.
- **Depende de:** T042
- **Estimación:** 4 j/d

### T061 · Media library
Subir, listar, buscar, borrar; integra con el servicio de imágenes; `alt` por idioma.
- **Criterios:** subir guarda en storage + `assets`; el campo `asset` selecciona de la librería; `alt` traducible.
- **Depende de:** T033, T043
- **Estimación:** 6 j/d

### T062 · Selector de idioma + estado de traducción
Cambiar locale; indicar traducido/publicado por idioma; "traducir desde el maestro".
- **Criterios:** cambiar locale carga su contenido; "traducir" clona la estructura con mismos `_uid`; indicador por idioma correcto.
- **Depende de:** T044
- **Estimación:** 5 j/d

### T063 · Roles y permisos por tenant
admin/editor por sitio; gestión de miembros.
- **Criterios:** un editor no puede gestionar usuarios; un admin sí; los permisos se aplican en la API, no solo en la UI.
- **Depende de:** T023
- **Estimación:** 4 j/d

### T064 · Theme editor (opcional)
Editar el subconjunto curado de tokens desde el CMS (plan sec. 9.bis).
- **Criterios:** cambiar un token global se refleja en preview y producción; solo opciones curadas; se guarda en `theme_settings`.
- **Depende de:** T044, T014
- **Estimación:** 8–12 j/d (opcional)

---

## E7 — Endurecido y lanzamiento

### T070 · Seguridad
Sanitización del richtext, rate limiting, CORS estricto, validación de orígenes, revisión de RLS.
- **Criterios:** el richtext servido está saneado (sin XSS); rate limiting activo; un pentest básico no encuentra fugas entre tenants.
- **Depende de:** todo lo anterior
- **Estimación:** 5 j/d

### T071 · Tests E2E del flujo crítico
Playwright: crear → editar → publicar → ver en la web.
- **Criterios:** el flujo completo pasa en CI; incluye un caso multi-idioma; corre contra staging.
- **Depende de:** E5, E6
- **Estimación:** 5 j/d

### T072 · Observabilidad
Logs estructurados, Sentry, métricas por tenant.
- **Criterios:** los errores llegan a Sentry con contexto de tenant; dashboard básico de latencia/errores.
- **Depende de:** T022
- **Estimación:** 3 j/d

### T073 · Infra de producción y despliegue
Contenerizar la API, DB gestionada, editor en CDN, entornos staging/prod, migraciones en el pipeline (Anexo A.12).
- **Criterios:** un push a `main` despliega a staging; promoción manual a prod; migraciones automáticas; rollback documentado.
- **Depende de:** T071
- **Estimación:** 6 j/d

### T074 · Documentación de usuario y de operación
Guía del editor para clientes + runbook de operación (backups, restore, alta de tenant).
- **Criterios:** un cliente nuevo puede usar el editor con la guía; el equipo puede dar de alta un tenant y restaurar un backup siguiendo el runbook.
- **Depende de:** T073
- **Estimación:** 3 j/d

---

## E8 (OPCIONAL) — Colaboración en tiempo real
Solo si se activa el módulo (plan sec. 11.bis). Requiere que T041 use mutaciones discretas.

### T080 · Estado colaborativo con Yjs + WebSockets
- **Criterios:** dos usuarios editan el mismo documento y ven los cambios del otro; fusión sin pérdida; presencia y cursores.
- **Depende de:** T041, E5
- **Estimación:** 20–30 j/d

---

## E9 — Capa SaaS y cumplimiento
Convierte el producto en un negocio operable. Ver `saas-operaciones.md`. Parte de esto (provisioning, audit, papelera) conviene tenerlo antes del primer cliente real.

### T090 · Provisioning de tenants
Alta de un tenant (manual y/o self-service): crea sitio + admin + esquemas base + suscripción de prueba, idempotente y auditado.
- **Criterios:** dar de alta un tenant deja todo listo en una operación; registrado en `audit_log`; repetible sin duplicar.
- **Depende de:** T023, T026
- **Estimación:** 4 j/d

### T091 · Planes, límites y cuotas
Definición de planes; la API comprueba límites antes de cada acción; rate limiting por tenant; medición de uso.
- **Criterios:** superar un límite devuelve error claro (402/429); un tenant no degrada a otro; el uso se mide.
- **Depende de:** T090
- **Estimación:** 5 j/d

### T092 · Facturación con Stripe
Suscripciones, webhooks de pago, estados (trial/active/past_due/canceled).
- **Criterios:** alta de suscripción funciona; los webhooks actualizan el estado; `past_due` pasa a solo-lectura tras gracia.
- **Depende de:** T091
- **Estimación:** 6 j/d

### T093 · Audit log y papelera (soft delete)
Registro de acciones + borrado con `deleted_at` y restauración.
- **Criterios:** publicar/borrar/restaurar quedan en `audit_log`; borrar va a papelera; se puede restaurar dentro del plazo.
- **Depende de:** T025
- **Estimación:** 3 j/d

### T094 · Export y borrado de datos del tenant (GDPR)
Export completo (contenido + assets + esquemas) y borrado verificable.
- **Criterios:** un tenant puede exportarse en formato abierto y restaurarse en otra instancia; el borrado elimina/anonimiza datos y assets.
- **Depende de:** T093
- **Estimación:** 4 j/d

### T095 · Backups probados y runbook de recuperación
Backups de DB + storage y un ensayo de restauración documentado.
- **Criterios:** existe backup automático; una restauración completa se ha probado en staging; runbook escrito con RPO/RTO.
- **Depende de:** T073
- **Estimación:** 3 j/d

### T096 · Evolución de esquemas y migración de contenido
Versionado de esquemas + script de migración de contenido + render tolerante (Anexo A.13).
- **Criterios:** un cambio aditivo no requiere migración; un cambio destructivo tiene script probado en staging; el `.astro` no rompe ante campos ausentes.
- **Depende de:** T026, T013
- **Estimación:** 4 j/d

---

## MVP recortado (lánzalo antes de la v1 completa)

🎯 Antes de construir las ~158 j/d enteras, valida con clientes reales un **MVP**. Subconjunto recomendado:
- **Incluye:** E0, E1, E2, E3 (con SEO/redirects), E4, una E5 **reducida** (preview con recarga en vez de morph fino; sin drag dentro del iframe), y lo mínimo de E9 (T090 provisioning manual, T093 audit+papelera, T095 backups).
- **Aplaza tras validar:** richtext inline (T053), drag en iframe (T054), theme editor (T064), facturación automática (T092 — al principio cobra a mano), scheduling (T036), colaboración (E8).
- **Objetivo:** llegar antes a "un cliente edita su web y publica" para aprender de uso real, y luego invertir en lo caro.

---

## Resumen de épicas

| Épica | Contenido | Tareas | j/d aprox. |
|---|---|---|---|
| E0 | Cimientos | T001–T005 | 5.5 |
| E1 | Contrato + render + bloques base | T010–T015 | 13 |
| E2 | API + DB multi-tenant | T020–T028 | 24 |
| E3 | Frontend + i18n + imágenes + SEO/scheduling/emails | T030–T037 | 26 |
| E4 | Editor básico | T040–T045 | 29 |
| E5 | Preview visual e interacciones | T050–T054 | 29 |
| E6 | Editor pulido | T060–T063 (+T064 opc.) | 19 |
| E7 | Lanzamiento | T070–T074 | 22 |
| | **Total producto v1** | | **~168 j/d** |
| E9 | Capa SaaS y cumplimiento | T090–T096 | +29 |
| E8 | Colaboración (opcional) | T080 | +20–30 |

> El MVP recortado (ver arriba) llega a un cliente funcional con bastante menos. Ajusta los totales tras E2, con velocidad real del equipo.
