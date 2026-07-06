---
name: freemium-gates
description: Implementación del modelo freemium con límites por plan en la plataforma. Úsala al agregar cualquier feature con cuotas, al construir el sistema de upgrades, o cuando necesites verificar si un usuario puede realizar una acción según su plan.
---

# Freemium Gates

El modelo freemium limita **recursos cuantificables**, no features. Cada acción que consuma un recurso limitado debe pasar por un gate antes de ejecutarse.

---

## Planes y límites

```typescript
export const PLAN_LIMITS = {
  free: {
    projects: 3,
    agents_per_project: 1,
    ai_messages_per_month: 500,
    team_members: 1,
    integrations: 0,
    crm_clients: 50,
    deployments: 1,
    templates_access: "public_only",
  },
  starter: {
    projects: 10,
    agents_per_project: 5,
    ai_messages_per_month: 5000,
    team_members: 3,
    integrations: 3,
    crm_clients: 500,
    deployments: 5,
    templates_access: "all",
  },
  pro: {
    projects: Infinity,
    agents_per_project: Infinity,
    ai_messages_per_month: 50000,
    team_members: Infinity,
    integrations: Infinity,
    crm_clients: Infinity,
    deployments: Infinity,
    templates_access: "all",
  },
} as const;

export type PlanTier = keyof typeof PLAN_LIMITS;
```

---

## Patrón del gate (middleware de Express)

```typescript
// lib: src/lib/freemium.ts
import { PLAN_LIMITS } from "./plan-limits";
import { db } from "@workspace/db";

export async function checkLimit(
  organizationId: string,
  resource: keyof typeof PLAN_LIMITS.free,
  currentCount: number
): Promise<{ allowed: boolean; limit: number; plan: PlanTier }> {
  const sub = await db.query.subscriptions.findFirst({
    where: eq(subscriptions.organizationId, organizationId),
  });
  const plan = (sub?.plan ?? "free") as PlanTier;
  const limit = PLAN_LIMITS[plan][resource] as number;

  return {
    allowed: currentCount < limit,
    limit,
    plan,
  };
}
```

---

## Respuesta estándar cuando se supera un límite

Siempre devolver HTTP **402 Payment Required** con este cuerpo:

```json
{
  "error": "limit_reached",
  "resource": "projects",
  "current": 3,
  "limit": 3,
  "plan": "free",
  "upgrade_url": "/settings/billing"
}
```

El frontend debe interceptar 402 y mostrar el modal de upgrade automáticamente.

---

## Tabla `subscriptions` en DB

```typescript
// lib/db/src/schema/subscriptions.ts
export const subscriptions = pgTable("subscriptions", {
  id: uuid("id").primaryKey().defaultRandom(),
  organizationId: uuid("organization_id").notNull().unique(),
  plan: text("plan").notNull().default("free"), // free | starter | pro
  stripeCustomerId: text("stripe_customer_id"),
  stripeSubscriptionId: text("stripe_subscription_id"),
  currentPeriodEnd: timestamp("current_period_end"),
  status: text("status").notNull().default("active"), // active | past_due | canceled
  createdAt: timestamp("created_at").defaultNow(),
  updatedAt: timestamp("updated_at").defaultNow(),
});
```

---

## Contador de mensajes IA (reset mensual)

```typescript
// Tabla: ai_usage
// Columnas: organization_id, year_month (ej: "2024-01"), message_count
// Incrementar con cada llamada a agente, verificar antes de ejecutar

async function incrementAiUsage(orgId: string): Promise<void> {
  const yearMonth = new Date().toISOString().slice(0, 7); // "YYYY-MM"
  await db
    .insert(aiUsage)
    .values({ organizationId: orgId, yearMonth, messageCount: 1 })
    .onConflictDoUpdate({
      target: [aiUsage.organizationId, aiUsage.yearMonth],
      set: { messageCount: sql`${aiUsage.messageCount} + 1` },
    });
}
```

---

## Modal de upgrade en el frontend

- Interceptar HTTP 402 globalmente en el custom-fetch del cliente API
- Mostrar modal con: recurso bloqueado, plan actual, beneficios del siguiente plan, CTA de upgrade
- El modal redirige a `/settings/billing` con `?upgrade=true&from=<resource>`

---

## Reglas clave

- Verificar el límite ANTES de crear el recurso, nunca después
- Los límites se evalúan al momento de la acción, no se cachean en el frontend
- Plan `free` nunca expira — no hay trial
- Al hacer downgrade, los recursos existentes que superen el límite quedan en modo "read-only" hasta que el usuario los archive
- Nunca bloquear el acceso a datos existentes, solo la creación de nuevos

---

## Referencias

- Ver `platform-monetization` skill para integración con Stripe
- Ver `platform-architecture` skill para tabla de límites completa
