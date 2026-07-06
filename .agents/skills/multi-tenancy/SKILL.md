---
name: multi-tenancy
description: Patrones de aislamiento de datos multi-tenant para la plataforma. Úsala al crear cualquier tabla nueva en la DB, al escribir queries, o al agregar middleware de autenticación. Previene filtraciones de datos entre organizaciones.
---

# Multi-Tenancy

Toda la plataforma usa **tenancy por columna**: cada tabla de negocio incluye `organization_id` y todas las queries lo filtran obligatoriamente. No hay schemas separados por tenant.

---

## Regla de oro

> **Ninguna query de negocio puede ejecutarse sin filtrar por `organization_id`.**

Si una query no tiene `.where(eq(table.organizationId, orgId))`, es un bug de seguridad.

---

## Middleware de tenant (Express)

```typescript
// src/middlewares/tenant.ts
import { clerkMiddleware, getAuth } from "@clerk/express";

export async function resolveTenant(req: Request, res: Response, next: NextFunction) {
  const { orgId, userId } = getAuth(req);

  if (!orgId) {
    return res.status(403).json({ error: "no_organization", message: "Select an organization first" });
  }

  // Adjuntar al request para uso en handlers
  req.tenantId = orgId;
  req.userId = userId;
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

Aplicar este middleware a **todas** las rutas protegidas, antes de los handlers.

---

## Patrón de query segura (Drizzle)

```typescript
// ✅ CORRECTO — siempre filtrar por tenantId
async function getProjects(tenantId: string) {
  return db.query.projects.findMany({
    where: eq(projects.organizationId, tenantId),
  });
}

// ❌ INCORRECTO — expone datos de todos los tenants
async function getProjects() {
  return db.query.projects.findMany(); // BUG DE SEGURIDAD
}
```

---

## Schema base para toda tabla de negocio

```typescript
// Todas las tablas de negocio DEBEN incluir estos campos
import { pgTable, uuid, timestamp } from "drizzle-orm/pg-core";

const tenantFields = {
  id: uuid("id").primaryKey().defaultRandom(),
  organizationId: uuid("organization_id").notNull(),  // FK implícita al tenant
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
};

// Índice obligatorio en organization_id para performance
// CREATE INDEX idx_<table>_org ON <table>(organization_id);
```

---

## Tablas que NO llevan organization_id

Solo las tablas de sistema/plataforma:
- `subscriptions` — lleva `organization_id` pero es una tabla de plataforma
- `plan_limits` — tabla estática de configuración
- `public_templates` — plantillas públicas del marketplace

---

## Aislamiento en uploads / archivos

- Los archivos de cada org se guardan bajo el prefijo `/<organization_id>/` en object storage
- Nunca exponer URLs directas con paths de otros tenants
- Al generar URLs firmadas, validar que `organization_id` del archivo coincide con el del request

---

## Checklist al crear una tabla nueva

- [ ] ¿Incluye `organization_id uuid NOT NULL`?
- [ ] ¿Tiene índice en `organization_id`?
- [ ] ¿Todas las queries filtran por `organization_id`?
- [ ] ¿El endpoint que la expone usa el middleware `resolveTenant`?
- [ ] ¿El test verifica que org A no puede leer datos de org B?

---

## Test de aislamiento (escribir para cada recurso nuevo)

```typescript
it("should not expose data across organizations", async () => {
  const org1Data = await createTestData(org1Id);
  const response = await request(app)
    .get(`/api/resource/${org1Data.id}`)
    .set("x-org-id", org2Id); // org diferente
  expect(response.status).toBe(404); // no 403, para no revelar existencia
});
```

---

## Referencias

- Ver `platform-architecture` skill para estructura de datos completa
- Ver `clerk-auth` skill para extracción de `orgId` desde el token
