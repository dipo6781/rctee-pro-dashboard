---
name: multi-tenancy
description: Patrones de aislamiento de datos multi-tenant para R-C-T-E-E Pro. Úsala al crear cualquier tabla nueva, al escribir queries, o al agregar middleware. Cada "tenant" es un consultor o agencia que usa la plataforma — sus clientes (pymes), entregables y plantillas personalizadas no deben ser visibles entre tenants.
---

# Multi-Tenancy — R-C-T-E-E Pro

Cada organización en la plataforma es un tenant aislado: un consultor, agencia o empresa que tiene sus propios clientes pyme, proyectos, entregables e integraciones. **Ninguna query de negocio puede ejecutarse sin filtrar por `organization_id`.**

---

## Regla de oro

> **Si una query no filtra por `organization_id`, es un bug de seguridad.**

---

## Fuente de verdad del tenant: Clerk Organizations

```typescript
// src/middlewares/tenant.ts
import { getAuth } from "@clerk/express";

export function resolveTenant(req: Request, res: Response, next: NextFunction) {
  const { orgId, userId } = getAuth(req);

  if (!orgId) {
    return res.status(403).json({
      error: "no_organization",
      message: "Debes seleccionar una organización para continuar.",
    });
  }

  req.tenantId = orgId;
  req.userId = userId!;
  next();
}

// Extender tipos de Express
declare global {
  namespace Express {
    interface Request {
      tenantId: string;
      userId: string;
    }
  }
}
```

Aplicar **antes** de todos los handlers protegidos. Las rutas públicas (`/api/webhooks/*`, `/s/:slug`) NO llevan este middleware.

---

## Patrón de query segura (Drizzle + Supabase)

```typescript
// ✅ CORRECTO
async function getClientDeliverables(tenantId: string, clientId: string) {
  return db.query.deliverables.findMany({
    where: and(
      eq(deliverables.organizationId, tenantId),
      eq(deliverables.clientId, clientId)
    ),
  });
}

// ❌ INCORRECTO — expone entregables de todos los tenants
async function getClientDeliverables(clientId: string) {
  return db.query.deliverables.findMany({
    where: eq(deliverables.clientId, clientId),
  });
}
```

---

## Schema base para toda tabla de negocio

```typescript
// Todas las tablas de negocio DEBEN tener organization_id
const tenantFields = {
  id: uuid("id").primaryKey().defaultRandom(),
  organizationId: text("organization_id").notNull(), // Clerk org ID (texto, no uuid)
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
};

// NOTA: Clerk usa strings como "org_2abc123..." para los IDs de organización
// NO usar uuid para organization_id — usar text
```

**Índice obligatorio en organization_id en cada tabla:**
```sql
CREATE INDEX idx_{table}_org_id ON {table}(organization_id);
```

---

## Tablas y su tenancy

| Tabla | Tiene org_id | Notas |
|---|---|---|
| `clients` | ✅ | pymes del consultor |
| `client_projects` | ✅ | proyectos por cliente |
| `deliverables` | ✅ | documentos generados |
| `generation_jobs` | ✅ | jobs de generación |
| `integrations` | ✅ | WhatsApp, Drive, etc. |
| `subscriptions` | ✅ | plan del tenant |
| `analytics_events` | ✅ | eventos de uso |
| `usage_counters` | ✅ | contador de generaciones |
| `templates` (platform) | ❌ | plantillas globales de la plataforma, acceso controlado por plan |
| `templates` (custom) | ✅ | plantillas personalizadas del consultor |

---

## Aislamiento en Supabase Storage

Los archivos de cada tenant van bajo su propio prefijo:

```
Bucket: "deliverables"
Path: {organization_id}/{client_id}/{deliverable_id}.{ext}

Bucket: "template-previews"
Path: public/{template_id}_preview.pdf  ← público, no requiere org
```

Al generar URLs firmadas, **siempre validar** que el `organization_id` del archivo coincide con el `tenantId` del request antes de devolver la URL.

---

## Checklist al crear una tabla nueva

- [ ] ¿Incluye `organization_id text NOT NULL`?
- [ ] ¿Tiene índice en `organization_id`?
- [ ] ¿Todas las queries en el código filtran por `organization_id`?
- [ ] ¿Los endpoints que la exponen usan el middleware `resolveTenant`?
- [ ] ¿Los archivos relacionados en Storage usan el path `{org_id}/...`?

---

## Test de aislamiento (obligatorio para cada recurso nuevo)

```typescript
it("tenant A no puede ver datos de tenant B", async () => {
  const clientA = await createTestClient(tenantA);

  const response = await request(app)
    .get(`/api/clients/${clientA.id}`)
    .set("Authorization", `Bearer ${tokenTenantB}`);

  // 404, no 403 — no revelar si el recurso existe
  expect(response.status).toBe(404);
});
```

---

## Reglas clave

- `organization_id` en esta plataforma es el Clerk Org ID (string como `"org_2abc..."`) — **no es un UUID de Postgres**
- Los endpoints públicos (`/api/webhooks/*`) nunca acceden a datos de tenant sin validación explícita del token del webhook
- Las plantillas de la plataforma (`is_platform_template: true`) no tienen `organization_id` — el acceso se controla por plan en `freemium-gates`
- Al responder 404 (no 403) cuando un tenant intenta acceder a datos de otro, evitamos revelar la existencia del recurso

---

## Referencias

- Ver `platform-architecture` skill para la estructura de datos completa
- Ver `freemium-gates` skill para control de acceso a plantillas por plan
- Leer `.local/skills/clerk-auth/SKILL.md` para setup de Clerk y extracción del `orgId`
