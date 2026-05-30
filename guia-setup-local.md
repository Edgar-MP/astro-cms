# Guía de arranque local (desde cero)

> Para alguien que acaba de llegar al proyecto y no ha visto nada de esto. Sigue los pasos **en orden**. Al final tendrás la API, el editor y una web de ejemplo corriendo en tu máquina. Si un paso falla, mira la sección de problemas al final.

---

## 1. Qué vas a montar

Tres cosas corriendo a la vez en tu ordenador:
- **La API** (el backend) en `http://localhost:3000`
- **El editor** (el CMS) en `http://localhost:5173`
- **Una web de ejemplo** (Astro) en `http://localhost:4321`

Y una **base de datos PostgreSQL** dentro de Docker.

No necesitas entender todo el proyecto para arrancarlo. Solo sigue la receta.

---

## 2. Programas que necesitas instalar

| Programa | Versión | Para qué | Cómo comprobar |
|---|---|---|---|
| **Node.js** | 20 o superior | Ejecuta el código JS/TS | `node -v` |
| **pnpm** | 9 o superior | Gestor de paquetes del monorepo | `pnpm -v` |
| **Docker** | reciente | Levanta la base de datos sin instalarla | `docker -v` |
| **Git** | cualquiera | Descargar el código | `git -v` |

Si no tienes `pnpm`: instálalo con `npm install -g pnpm`.
Si no tienes Docker: instala "Docker Desktop" desde su web.

---

## 3. Descargar el código

```bash
git clone <URL-del-repo-cms-core>
cd cms-core
```

> "Clonar" = descargar una copia del proyecto a tu máquina. La `<URL-del-repo>` te la da quien lleve el proyecto.

---

## 4. Instalar las dependencias

Desde la raíz del proyecto:
```bash
pnpm install
```
Esto descarga todas las librerías de todos los paquetes a la vez (es un monorepo). Tarda un par de minutos la primera vez.

---

## 4.bis Activar los git hooks del repo

El repo incluye hooks en `.githooks/` (están versionados). Hay que decirle a git dónde están:

```bash
git config core.hooksPath .githooks
```

Esto activa el **hook post-commit** que regenera el índice de CodeGraph automáticamente tras cada commit. Sin este paso, el índice se queda desactualizado y las herramientas de análisis de código del agente no funcionan bien.

> Nota: CodeGraph debe estar instalado globalmente (`npm install -g @colbymchenry/codegraph` o con `npx @colbymchenry/codegraph` la primera vez). Si no está disponible, el hook lo ignora sin bloquear el commit.

---

## 5. Configurar las variables de entorno

Las "variables de entorno" son datos de configuración (contraseñas, URLs) que no van en el código. Hay un archivo de ejemplo: cópialo.

```bash
cp .env.example .env
```

Abre `.env` y revisa que tenga algo así (los valores de ejemplo sirven para local):
```bash
DATABASE_URL="postgresql://cms:cms@localhost:5432/cms_dev"
API_PORT=3000
JWT_SECRET="cambia-esto-en-produccion"
STORAGE_BUCKET="local"            # en local usamos disco; en prod, S3/R2
PREVIEW_TOKEN="dev-preview-token"
```
> ⚠️ Nunca subas tu `.env` al repositorio. El `.env.example` sí va al repo (sin secretos reales).

---

## 6. Levantar la base de datos

```bash
docker compose up -d
```
- `up` arranca PostgreSQL dentro de un contenedor.
- `-d` lo deja corriendo en segundo plano.

Comprueba que está vivo:
```bash
docker compose ps
```
Deberías ver un servicio `postgres` en estado "running".

---

## 7. Crear las tablas (migraciones) y datos iniciales (seed)

```bash
pnpm db:migrate     # crea todas las tablas en la base de datos
pnpm db:seed        # crea un tenant, un usuario admin y datos de ejemplo
```
- **Migrar** = aplicar la estructura de tablas a la base de datos.
- **Seed** = "sembrar" datos iniciales para poder empezar.

El seed te dirá por consola el usuario admin creado, algo como:
```
✔ Tenant 'demo' creado
✔ Usuario admin: admin@demo.local / contraseña: admin1234
```
Apunta esas credenciales: las usarás para entrar al editor.

---

## 8. Arrancar todo

Opción fácil (todo a la vez desde la raíz):
```bash
pnpm dev
```
Esto arranca la API, el editor y la web de ejemplo en paralelo.

Si prefieres tres terminales separadas:
```bash
pnpm --filter @cms/api dev        # API en :3000
pnpm --filter @cms/editor dev     # Editor en :5173
pnpm --filter web-demo dev        # Web Astro en :4321
```

---

## 9. Comprobar que funciona

1. Abre `http://localhost:3000/health` → debe responder `{ "status": "ok" }`.
2. Abre `http://localhost:5173` → pantalla de login del editor. Entra con las credenciales del paso 7.
3. Abre `http://localhost:4321` → la web de ejemplo renderizada.
4. Sincroniza los esquemas de bloques de la web con el editor (para que el editor sepa qué bloques existen):
   ```bash
   pnpm --filter web-demo push-schemas
   ```
5. En el editor, abre una página, edita un bloque y pulsa publicar. Recarga la web: deberías ver el cambio.

Si los 5 pasos funcionan, tienes el entorno completo. 🎉

---

## 10. Comandos del día a día

| Comando | Qué hace |
|---|---|
| `pnpm dev` | Arranca todo en modo desarrollo |
| `pnpm build` | Compila todos los paquetes |
| `pnpm lint` | Revisa el estilo del código |
| `pnpm typecheck` | Revisa los tipos de TypeScript |
| `pnpm test` | Ejecuta los tests |
| `pnpm db:migrate` | Aplica migraciones pendientes |
| `pnpm db:studio` | Abre una UI para ver/editar la base de datos |
| `docker compose down` | Apaga la base de datos |

---

## 11. Si algo falla (problemas comunes)

**"No me conecto a la base de datos"**
→ ¿Está Docker corriendo? `docker compose ps`. ¿La `DATABASE_URL` del `.env` coincide con el puerto del `docker-compose.yml` (5432)?

**"El editor no muestra ningún tipo de bloque"**
→ Falta sincronizar los esquemas. Ejecuta `pnpm --filter web-demo push-schemas` (paso 9.4).

**"El preview no se actualiza al editar"**
→ Comprueba que la web tiene la ruta de preview y el `bridge.ts`, y que `PREVIEW_TOKEN` del `.env` coincide. Mira la consola del navegador por errores de `postMessage`/origin.

**"Cambié el esquema de la base de datos y ahora hay errores"**
→ Genera y aplica una migración nueva: `pnpm db:generate` y luego `pnpm db:migrate`. Nunca edites tablas a mano.

**"`pnpm install` falla"**
→ ¿Versión de Node 20+? `node -v`. Borra `node_modules` y `pnpm-lock.yaml` solo si te lo indica quien lleve el proyecto, y reinstala.

**"Un puerto está ocupado"**
→ Otra cosa usa ese puerto. Ciérrala o cambia el puerto en el `.env` / config del paquete.

---

## 12. ¿Y ahora qué leo?

- Para entender **qué** es el proyecto y el plan: `plan-cms-bloques.md`.
- Para entender **cómo** está construido por dentro: `anexo-arquitectura.md`.
- Para saber **qué tarea coger**: `backlog-tareas.md` (empieza por la primera no hecha).
- Si un término no lo entiendes: el **glosario** está al principio del plan.
