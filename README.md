# CMS de bloques (SaaS) — Documentación del proyecto

> Punto de entrada a toda la documentación. Si acabas de llegar y no sabes de qué va esto, **empieza por aquí**.

---

## 1. ¿Qué es esto, en una frase?

Un **CMS headless basado en bloques con editor visual** (al estilo Storyblok/Shopify) para construir webs en **Astro**, ofrecido como **SaaS multi-tenant**: nosotros hospedamos una plataforma central y cada cliente edita el contenido de su web viéndolo en vivo.

## 2. ¿Cómo funciona, en cuatro ideas?

1. **El contenido es un árbol de bloques** guardado en JSON (un "hero", un "texto", una "rejilla"…). No es HTML ni markdown.
2. **El editor visual** carga la web real dentro de un iframe y deja editar cada bloque al hacer clic; la comunicación va por `postMessage`.
3. **La web (Astro)** mapea cada tipo de bloque a un componente y lo renderiza; lee el contenido de la API.
4. Es **multi-idioma** y **multi-tenant** desde el diseño, con un servicio propio de imágenes.

## 3. Las tres piezas + la base de datos

```
  Editor (CMS)  ⇄ postMessage ⇄  Web Astro (preview e iframe)
       │                              │
       │  lee esquemas / guarda       │  lee contenido publicado
       ▼                              ▼
            API central (multi-tenant)
                     │
        PostgreSQL (jsonb + RLS)  +  Storage (S3/R2)
```

## 4. Mapa de la documentación (qué hay en cada archivo)

| Archivo | Qué contiene | Léelo si… |
|---|---|---|
| **`README.md`** (este) | Visión, índice y por dónde empezar | …acabas de llegar. |
| **`plan-cms-bloques.md`** | El plan completo: visión, alcance, modelo de datos, i18n, editor, frontend, design system, API, imágenes, roadmap por fases, riesgos, glosario | …quieres entender el **qué** y el **porqué**, y el orden de construcción. |
| **`anexo-arquitectura.md`** | El detalle técnico: cuántos proyectos hay, base de datos (DDL completo), stack y arquitectura interna de cada proyecto, multi-tenancy (RLS), evolución de esquemas, caché y despliegue SaaS | …vas a programar y necesitas el **cómo** por dentro. |
| **`saas-operaciones.md`** | La capa de negocio y operación: planes, facturación, alta de tenants, cuotas, cumplimiento (GDPR), audit, backups, emails, accesibilidad, observabilidad | …te ocupas del producto como negocio o de operarlo. |
| **`backlog-tareas.md`** | Todas las tareas **ordenadas**, con criterios de aceptación, dependencias y estimación, agrupadas en épicas | …quieres saber **qué hacer ahora** (coge la primera tarea no hecha). |
| **`guia-setup-local.md`** | Cómo dejar el proyecto corriendo en tu máquina, paso a paso, desde cero | …vas a tocar código por primera vez. |
| **`guia-crear-bloque.md`** | Cómo crear un bloque nuevo: componente + esquema, variantes, campos condicionales y presets | …vas a añadir o modificar bloques (la tarea más repetida). |

> El **glosario** (qué es un bloque, un esquema, un draft, un tenant…) está al principio de `plan-cms-bloques.md`, sección 1.

## 5. Por dónde empezar según tu rol

- **Responsable de producto / gestión:** lee este README + `plan-cms-bloques.md` (secciones 2, 7, 14). Te da visión, alcance, multi-idioma y presupuesto por fases.
- **Persona desarrolladora (primer día):** lee este README → `guia-setup-local.md` (monta el entorno) → `anexo-arquitectura.md` (entiende la estructura) → coge la primera tarea de `backlog-tareas.md`.
- **Backend:** céntrate en `anexo-arquitectura.md` (A.3 base de datos, A.5 API, A.11 multi-tenancy) y las épicas E0–E2 del backlog.
- **Frontend / Astro:** `plan` sec. 9 + `anexo` A.6, y las épicas E1, E3.
- **Editor (React):** `plan` sec. 8 + `anexo` A.8, y las épicas E4–E6.

## 6. Decisiones clave ya tomadas (no las re-discutas sin motivo)

- **Modelo SaaS multi-tenant gestionado**, con aislamiento por `site_id` + Row-Level Security.
- **Producción híbrida** en Astro: cada página decide si es estática (SSG) o dinámica (SSR).
- **Multi-idioma obligatorio**: una fila de contenido por (documento, locale), con `_uid` compartidos.
- **Servicio de imágenes propio obligatorio**; theme editor y colaboración en tiempo real son **opcionales**.
- **Los esquemas de bloque se definen en código** (en cada web) y se sincronizan a la API; la fuente de verdad es el código.
- **El editor trabaja por "mutaciones"** sobre un estado que es la única fuente de verdad.
- **SEO y redirecciones** forman parte del modelo de datos (metadatos por página/idioma + 301 al cambiar slugs).
- **Accesibilidad (WCAG 2.1 AA)** es parte de la Definición de Hecho, no un extra al final.
- **Empieza por un MVP recortado** (ver `backlog-tareas.md`) antes de la v1 completa.

## 7. Estado y plan a alto nivel

El trabajo está dividido en 8 épicas (E0 cimientos → E7 lanzamiento, + E8 colaboración opcional). El presupuesto estimado de la v1 es de **~150 jornadas de desarrollo** (~4–6 meses con dos personas). El detalle está en `backlog-tareas.md` y en `plan-cms-bloques.md` (sec. 14).

> Regla de oro del proyecto: **congela el contrato de bloques pronto** (E1) y construye sobre él. Cambiar el modelo de datos a mitad es lo más caro que puede pasar.

---

*Mantén este README actualizado: es lo primero que lee quien llega nuevo.*
