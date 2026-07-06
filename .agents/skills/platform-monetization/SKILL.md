---
name: platform-monetization
description: Implementación completa de monetización freemium con Stripe para la plataforma AI. Úsala al construir el módulo de billing, al configurar planes y suscripciones, o al manejar webhooks de pago. Cubre checkout, upgrades, downgrades y cancelaciones.
---

# Platform Monetization

Modelo freemium con upgrades a planes pagos vía Stripe Subscriptions. Los límites de cada plan están en `freemium-gates` skill.

---

## Setup inicial

Leer la skill de Stripe ANTES de implementar: `.local/skills/stripe/SKILL.md`
Leer la skill de monetización: `.local/skills/monetization/SKILL.md`

---

## Flujo de suscripción

```
Usuario free → clic "Upgrade" → Stripe Checkout Session → pago → webhook → actualizar plan en DB → usuario en plan Starter/Pro
```

---

## Productos en Stripe (crear una sola vez en el dashboard)

| Plan | Precio | Stripe Price ID (guardar en env) |
|---|---|---|
| Starter | $29/mes | `STRIPE_STARTER_MONTHLY_PRICE_ID` |
| Starter | $290/año | `STRIPE_STARTER_YEARLY_PRICE_ID` |
| Pro | $99/mes | `STRIPE_PRO_MONTHLY_PRICE_ID` |
| Pro | $990/año | `STRIPE_PRO_YEARLY_PRICE_ID` |

---

## Endpoints requeridos

```
POST /api/billing/checkout          → crear Stripe Checkout Session
POST /api/billing/portal            → abrir Stripe Customer Portal
GET  /api/billing/subscription      → estado actual del plan
POST /api/billing/webhooks          → recibir eventos de Stripe (público, sin auth)
```

---

## Crear Checkout Session

```typescript
// POST /api/billing/checkout
import Stripe from "stripe";
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

app.post("/api/billing/checkout", requireAuth, async (req, res) => {
  const { priceId, billingInterval } = req.body; // "monthly" | "yearly"
  const { tenantId } = req;

  // Obtener o crear Stripe customer
  let sub = await db.query.subscriptions.findFirst({
    where: eq(subscriptions.organizationId, tenantId),
  });

  let customerId = sub?.stripeCustomerId;
  if (!customerId) {
    const customer = await stripe.customers.create({
      metadata: { organizationId: tenantId },
    });
    customerId = customer.id;
    await db.update(subscriptions)
      .set({ stripeCustomerId: customerId })
      .where(eq(subscriptions.organizationId, tenantId));
  }

  const session = await stripe.checkout.sessions.create({
    customer: customerId,
    payment_method_types: ["card"],
    line_items: [{ price: priceId, quantity: 1 }],
    mode: "subscription",
    success_url: `${process.env.APP_URL}/settings/billing?success=true`,
    cancel_url: `${process.env.APP_URL}/settings/billing?canceled=true`,
    metadata: { organizationId: tenantId },
  });

  res.json({ url: session.url });
});
```

---

## Webhook handler (eventos críticos)

```typescript
// POST /api/billing/webhooks — SIN middleware de auth
app.post("/api/billing/webhooks", express.raw({ type: "application/json" }), async (req, res) => {
  const sig = req.headers["stripe-signature"]!;
  let event: Stripe.Event;

  try {
    event = stripe.webhooks.constructEvent(req.body, sig, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch {
    return res.status(400).send("Webhook signature failed");
  }

  switch (event.type) {
    case "checkout.session.completed": {
      const session = event.data.object as Stripe.CheckoutSession;
      const orgId = session.metadata?.organizationId;
      const sub = await stripe.subscriptions.retrieve(session.subscription as string);
      await updateSubscription(orgId!, sub);
      break;
    }
    case "customer.subscription.updated":
    case "customer.subscription.deleted": {
      const sub = event.data.object as Stripe.Subscription;
      const customer = await stripe.customers.retrieve(sub.customer as string) as Stripe.Customer;
      const orgId = customer.metadata.organizationId;
      await updateSubscription(orgId, sub);
      break;
    }
  }

  res.json({ received: true });
});

async function updateSubscription(orgId: string, stripeSub: Stripe.Subscription) {
  const priceId = stripeSub.items.data[0].price.id;
  const plan = getPlanFromPriceId(priceId); // mapear price ID → "starter" | "pro"

  await db.update(subscriptions).set({
    plan,
    stripeSubscriptionId: stripeSub.id,
    currentPeriodEnd: new Date(stripeSub.current_period_end * 1000),
    status: stripeSub.status,
    updatedAt: new Date(),
  }).where(eq(subscriptions.organizationId, orgId));
}
```

---

## Variables de entorno requeridas

```
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
STRIPE_STARTER_MONTHLY_PRICE_ID=price_...
STRIPE_STARTER_YEARLY_PRICE_ID=price_...
STRIPE_PRO_MONTHLY_PRICE_ID=price_...
STRIPE_PRO_YEARLY_PRICE_ID=price_...
APP_URL=https://tu-dominio.replit.app
```

Usar `.local/skills/environment-secrets/SKILL.md` para configurar estos secrets.

---

## Reglas clave

- El webhook endpoint NUNCA lleva middleware de autenticación — Stripe lo llama directamente
- Usar `express.raw()` en el webhook, NO `express.json()` — la firma falla si el body es parseado
- Siempre verificar la firma del webhook antes de procesar (`constructEvent`)
- Al downgrade, no borrar datos — solo cambiarlos a "read-only" si superan el límite del plan nuevo
- Stripe Customer Portal permite al usuario gestionar su suscripción sin código adicional

---

## Referencias

- Ver `freemium-gates` skill para los límites por plan
- Leer `.local/skills/stripe/SKILL.md` antes de implementar
- Ver `.local/skills/environment-secrets/SKILL.md` para configurar secrets
