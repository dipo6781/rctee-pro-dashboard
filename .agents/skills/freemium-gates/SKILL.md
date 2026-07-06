---
name: freemium-gates
description: Implementación del modelo freemium con límites por plan en R-C-T-E-E Pro. Úsala al agregar cualquier feature con cuotas, al verificar si un usuario puede usar una plantilla, o al mostrar mensajes de upgrade. Los límites son por número de usos de plantillas y clientes en CRM, no por features bloqueadas.
---

# Freemium Gates — R-C-T-E-E Pro

El modelo limita **cuántas veces** se puede usar la plataforma, no qué features están disponibles. El principio: el usuario SIEMPRE ve todas las plantillas — solo se bloquea al intentar generar el entregable si superó su cuota.

---

## Planes y límites

```typescript
export const PLAN_LIMITS = {
  free: {
    templates_access: ["diagnostico_financiero_basic", "cobranza_basic", "bienestar_basic"], // 3 específicas
    generations_per_month: 2,           // entregables generados
    clients_in_crm: 5,
    active_projects: 2,
    verticals_access: 1,                // solo el vertical que eligió al registrarse
    white_label: false,
    share_deliverable: false,           // no puede enviar docs al cliente desde la app
  },
  starter: {
    templates_access: "one_vertical",   // todas las plantillas de 1 vertical
    generations_per_month: 10,
    clients_in_crm: 50,
    active_projects: 10,
    verticals_access: 1,
    white_label: false,
    share_deliverable: true,
  },
  pro: {
    templates_access: "all",            // todos los verticales (60+ plantillas)
    generations_per_month: Infinity,
    clients_in_crm: Infinity,
    active_projects: Infinity,
    verticals_access: 7,
    white_label: false,
    share_deliverable: true,
  },
  agency: {
    templates_access: "all",
    generations_per_month: Infinity,
    clients_in_crm: Infinity,
    active_projects: Infinity,
    verticals_access: 7,
    white_label: true,                  // sin marca R-C-T-E-E en los documentos
    share_deliverable: true,
  },
} as const;

export type PlanTier = keyof typeof PLAN_LIMITS;
```

---

## Contador de generaciones (reset mensual)

```typescript
// Tabla: usage_counters
// organization_id | year_month | generations_count | created_at

async function checkGenerationLimit(orgId: string): Promise<{
  allowed: boolean;
  used: number;
  limit: number;
  plan: PlanTier;
}> {
  const yearMonth = new Date().toISOString().slice(0, 7); // "YYYY-MM"
  const sub = await getSubscription(orgId);
  const plan = (sub?.plan ?? "free") as PlanTier;
  const limit = PLAN_LIMITS[plan].generations_per_month;

  const counter = await db.query.usageCounters.findFirst({
    where: and(
      eq(usageCounters.organizationId, orgId),
      eq(usageCounters.yearMonth, yearMonth)
    ),
  });

  const used = counter?.generationsCount ?? 0;

  return {
    allowed: limit === Infinity || used < limit,
    used,
    limit: limit === Infinity ? -1 : limit,
    plan,
  };
}

async function incrementGenerationCount(orgId: string): Promise<void> {
  const yearMonth = new Date().toISOString().slice(0, 7);
  await db.insert(usageCounters)
    .values({ organizationId: orgId, yearMonth, generationsCount: 1 })
    .onConflictDoUpdate({
      target: [usageCounters.organizationId, usageCounters.yearMonth],
      set: { generationsCount: sql`${usageCounters.generationsCount} + 1` },
    });
}
```

---

## Verificación de acceso a plantilla

```typescript
async function canAccessTemplate(
  orgId: string,
  templateId: string
): Promise<{ allowed: boolean; reason?: string }> {
  const [sub, template] = await Promise.all([
    getSubscription(orgId),
    getTemplate(templateId),
  ]);

  const plan = (sub?.plan ?? "free") as PlanTier;
  const limits = PLAN_LIMITS[plan];

  // Plan free: solo 3 plantillas específicas
  if (plan === "free") {
    const freeTemplates = limits.templates_access as string[];
    if (!freeTemplates.includes(template.slug)) {
      return { allowed: false, reason: "template_not_in_free_plan" };
    }
  }

  // Plan starter: solo 1 vertical
  if (plan === "starter") {
    if (template.vertical !== sub?.selectedVertical) {
      return { allowed: false, reason: "vertical_not_in_plan" };
    }
  }

  return { allowed: true };
}
```

---

## Respuesta estándar cuando se supera un límite

HTTP **402 Payment Required**:

```json
{
  "error": "limit_reached",
  "resource": "generations_per_month",
  "used": 2,
  "limit": 2,
  "plan": "free",
  "reset_date": "2024-02-01",
  "upgrade_url": "/settings/billing",
  "message": "Alcanzaste el límite de 2 documentos este mes. Actualiza tu plan para generar más."
}
```

---

## Mensajes de upgrade en la UI (tono colombiano / cercano)

```typescript
const UPGRADE_MESSAGES: Record<string, string> = {
  generations_per_month:
    "¡Ya usaste tus 2 documentos del mes! Actualiza al plan Starter para generar hasta 10 al mes.",
  template_not_in_free_plan:
    "Esta plantilla está disponible desde el plan Starter. ¿Listo para sacarle más provecho a la plataforma?",
  vertical_not_in_plan:
    "Tienes acceso al vertical de Cobranza. Para usar plantillas de RRHH, actualiza al plan Pro.",
  clients_in_crm:
    "Llegaste al límite de 5 clientes en el plan gratuito. Con Starter puedes gestionar hasta 50.",
  white_label:
    "El white label (sin marca de la plataforma en tus documentos) está disponible en el plan Agencia.",
};
```

---

## Dónde aplicar los gates (checklist)

- [ ] **Antes de generar un entregable** → verificar `checkGenerationLimit` + `canAccessTemplate`
- [ ] **Al crear un cliente en CRM** → verificar `clients_in_crm`
- [ ] **Al crear un proyecto** → verificar `active_projects`
- [ ] **Al compartir un entregable** → verificar `share_deliverable`
- [ ] **Al acceder a una plantilla de otro vertical (starter)** → verificar `verticals_access`
- [ ] **Al generar sin marca (white label)** → verificar plan `agency`

---

## Tabla `subscriptions` (campos relevantes para gates)

```typescript
// Agregar a la tabla subscriptions:
selectedVertical: text("selected_vertical"),  // para plan starter: qué vertical eligió
```

Al registrarse en plan Starter, el usuario elige su vertical principal. Esto determina qué plantillas puede usar.

---

## Reglas clave

- El usuario SIEMPRE puede ver todas las plantillas — el lock solo aparece al intentar generar
- Los contadores de generación se resetean el día 1 de cada mes (UTC-5 Colombia)
- Al hacer downgrade, los proyectos/clientes existentes quedan en modo "solo lectura"
- Plan Free nunca expira — no hay trial con fecha límite
- Incrementar el contador DESPUÉS de que la generación sea exitosa, no antes

---

## Referencias

- Ver `platform-monetization` skill para los precios y el proceso de pago
- Ver `platform-architecture` skill para la tabla de planes completa
