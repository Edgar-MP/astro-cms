# Capa SaaS, cumplimiento y operación

> Lo que convierte "un CMS multi-tenant" en "un negocio SaaS que se puede vender y operar con responsabilidad". Complementa al `plan-cms-bloques.md` (producto) y al `anexo-arquitectura.md` (técnica). Aquí va el negocio, el cumplimiento legal y la operación del día a día.

---

## 1. Planes y facturación

### 1.1 Planes
Define 2–3 planes con límites distintos. Ejemplo orientativo:

| Plan | Sitios | Usuarios | Almacenamiento | Idiomas | Precio |
|---|---|---|---|---|---|
| Starter | 1 | 2 | 5 GB | 2 | € |
| Pro | 5 | 10 | 50 GB | ilimitados | €€ |
| Business | ilimitados | ilimitados | 500 GB | ilimitados | €€€ |

Los límites se guardan por plan y se comprueban en la API (sec. 3).

### 1.2 Facturación
- Integra **Stripe** (o similar): productos = planes, suscripciones, webhooks de pago.
- Tablas: `plans`, `subscriptions (tenant_id, plan_id, status, current_period_end)`, y `usage` para el consumo medido.
- ⚠️ Escucha los **webhooks de Stripe** (pago fallido, cancelación) y refleja el estado en la suscripción; no confíes solo en lo que diga el frontend.
- Gestiona estados: `trialing`, `active`, `past_due`, `canceled`. Una suscripción `past_due` puede pasar a modo solo-lectura tras un periodo de gracia.

> **Para aprender:** "Stripe Billing subscriptions", "Stripe webhooks", "metered usage billing".

---

## 2. Alta de clientes (provisioning de tenants)

Decide **cómo nace un tenant**:
- **Self-service:** el cliente se registra, se crea su tenant + primer usuario admin + sitio vacío + suscripción de prueba, automáticamente.
- **Alta manual:** tu equipo crea el tenant (útil al principio, con pocos clientes y ventas a medida).

El proceso de provisioning debe crear, en una sola operación: el `site`, el `membership` del primer admin, los `block_schemas` base (push inicial) y, si aplica, la `subscription`. Hazlo **idempotente** y registrado en el `audit_log`.

> **Para aprender:** "tenant provisioning", "onboarding flow", "idempotent operations".

---

## 3. Cuotas y límites (anti "vecino ruidoso")

Cada plan impone límites que la API comprueba **antes** de permitir la acción:
- Número de sitios, usuarios, almacenamiento, peticiones por minuto.
- Al superar un límite: error claro (`402`/`429`) explicando qué límite y cómo ampliarlo.
- **Rate limiting por tenant** para que un cliente no degrade a los demás (clave en el pool compartido).
- Mide el **uso** (`usage`) para facturación medida y para avisos ("vas al 90% de tu almacenamiento").

> **Para aprender:** "rate limiting per tenant", "quota enforcement", "usage metering".

---

## 4. Cumplimiento y datos (GDPR)

Guardas datos de empresas, así que esto **no es opcional** y conviene revisarlo con asesoría legal pronto.

- **Exportación de datos de un tenant:** un export completo (contenido + assets + esquemas) en formato abierto. Sirve como derecho de portabilidad y como garantía anti-lock-in.
- **Borrado:** borrar un tenant elimina (o anonimiza) sus datos de forma verificable, incluidos los assets en storage y los backups según política.
- **Residencia de datos:** algunos clientes exigen UE. Telo facilita la capa de resolución de tenant (Anexo A.11): un tenant puede vivir en una DB/región concreta.
- **Subprocesadores y consentimiento:** documenta qué servicios de terceros tocan datos (Stripe, email, storage, Sentry).

⚠️ No soy asesoría legal: valida el detalle (RGPD/LOPD, contratos de encargo de tratamiento) con quien lleve lo legal en tu empresa.

> **Para aprender:** "GDPR data export", "right to erasure", "data residency", "data processing agreement (DPA)".

---

## 5. Trazabilidad y recuperación

- **Audit log** (tabla `audit_log`, Anexo A.3): quién hizo qué y cuándo (publicar, borrar, restaurar, invitar). Imprescindible con varios clientes y reclamaciones.
- **Papelera / soft delete:** borrar marca `deleted_at` en vez de eliminar; hay un periodo para restaurar antes del borrado definitivo.
- **Backups y recuperación:**
  - Backups automáticos de PostgreSQL (los da el servicio gestionado) + del storage.
  - ⚠️ Un backup no probado **no es un backup**: ensaya una restauración completa en staging y documenta el procedimiento (RPO/RTO objetivos).
  - Runbook de "cómo restaurar un tenant a una fecha".

> **Para aprender:** "audit logging", "soft delete", "point-in-time recovery", "RPO RTO", "backup restore drills".

---

## 6. Emails transaccionales

En un SaaS con cuentas, necesitas enviar emails: invitar a un usuario a un tenant, recuperar contraseña, verificación de email, avisos de facturación/cuota.
- Proveedor: **Resend** o **Postmark** (transaccional, buena entregabilidad).
- Plantillas versionadas; soporte multi-idioma de los emails.
- ⚠️ Nunca metas secretos del proveedor en el repo; van en variables de entorno.

> **Para aprender:** "transactional email", "Resend / Postmark", "email deliverability (SPF, DKIM)".

---

## 7. Accesibilidad (a11y)

Transversal; mejor desde el principio que parcheada al final.
- **El HTML que generan los bloques** debe ser semántico: encabezados en orden, `alt` en imágenes (campo ya traducible), foco visible, contraste suficiente (esto último apóyalo en los design tokens).
- **El editor** debe ser navegable por teclado (incluido el drag & drop del aside con dnd-kit, que ya lo soporta) y con etiquetas `aria` en los controles.
- Referencia objetivo: **WCAG 2.1 AA**. Está en la Definición de Hecho (plan sec. 16).

> **Para aprender:** "WCAG 2.1 AA", "accessible drag and drop", "axe-core testing".

---

## 8. Observabilidad y soporte

- **Logs estructurados** con contexto de tenant; **Sentry** para errores; métricas de latencia y errores por tenant.
- **Panel de estado** y alertas (caída de API, errores de pago, cola de scheduling atascada).
- **Soporte:** un canal y un proceso; el audit log ayuda a investigar incidencias por cliente.

> **Para aprender:** "structured logging", "Sentry", "SLO / SLA", "status page".

---

## 9. Qué es imprescindible antes de cobrar al primer cliente

No necesitas todo esto a la vez. El mínimo para **cobrar con responsabilidad**:
1. Provisioning de tenant (aunque sea manual al principio).
2. Facturación básica (Stripe) y al menos un plan con sus límites aplicados.
3. Backups probados + papelera.
4. Audit log.
5. Emails de invitación y reset de contraseña.
6. Export de datos del tenant (portabilidad/GDPR).

El resto (residencia de datos, usage metering fino, métricas avanzadas) puede llegar después, cuando lo pida un cliente concreto.
