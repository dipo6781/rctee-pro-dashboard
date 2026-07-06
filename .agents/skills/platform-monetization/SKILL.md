---
name: platform-monetization
description: Implementación del modelo de monetización híbrido de R-C-T-E-E Pro. Úsala al construir el módulo de billing, al configurar planes y pagos, o al implementar el modelo de venta por proyecto. Incluye estrategia escalonada: freemium → suscripción → venta de proyectos. Mercado primario: Colombia.
---

# Platform Monetization — R-C-T-E-E Pro

Modelo híbrido con 3 capas de ingreso. El mercado primario es Colombia — priorizar PayU sobre Stripe.

---

## Estrategia Escalonada de Monetización

### Capa 1: Acceso a la plataforma (recurrente)

| Plan | Precio | Qué incluye |
|---|---|---|
| **Free** | $0 | 3 plantillas básicas, 2 usos/mes |
| **Starter** | $97 USD/mes | 1 vertical completo, 10 usos/mes |
| **Pro** | $297 USD/mes | Todos los verticales, usos ilimitados |
| **Agencia** | $497 USD/mes | Todo Pro + white label sin marca de plataforma |

### Capa 2: Venta de proyectos (por entregable)

El consultor/agencia vende el resultado a su cliente pyme. La plataforma puede facilitar el cobro:

| Entregable | Precio sugerido al cliente final |
|---|---|
| Diagnóstico Financiero Express | $4,500 USD |
| Plan de Cobranza Completo | $6,000 USD |
| Programa Bienestar Laboral | $15,000 USD |
| Paquete completo (3 entregables) | $20,000–$50,000 USD |

**Comisión de plataforma:** 5–10% si el cobro pasa por la plataforma (fase futura).

### Capa 3: Marketplace de plantillas (fase futura)

Consultores expertos venden sus propias plantillas R-C-T-E-E. Plataforma toma 20% de comisión.

---

## Proveedores de pago

### Colombia (primario)
**PayU Latam** — líder en Colombia, acepta PSE, tarjetas locales, efectivo (Efecty/Baloto)

```typescript
// PayU no tiene SDK TypeScript oficial — usar API REST directamente
// Documentación: https://developers.payulatam.com

const PAYU_CONFIG = {
  apiLogin: process.env.PAYU_API_LOGIN!,
  apiKey: process.env.PAYU_API_KEY!,
  merchantId: process.env.PAYU_MERCHANT_ID!,
  accountId: process.env.PAYU_ACCOUNT_ID!,
  baseUrl: process.env.NODE_ENV === "production"
    ? "https://api.payulatam.com/payments-api/4.0/service.cgi"
    : "https://sandbox.api.payulatam.com/payments-api/4.0/service.cgi",
};
```

### Internacional (secundario)
**PayPal** — para clientes fuera de Colombia o que prefieren USD.
**Stripe** — solo si el usuario lo pide explícitamente (no priorizar para Colombia).

### MVP (fase 0 — antes de integrar APIs de pago)
Para el MVP no integrar payment gateways. Usar proceso manual:
1. Usuario elige plan/plantilla
2. Plataforma muestra instrucciones: transferencia bancaria Bancolombia/Nequi o PayPal
3. Usuario envía comprobante → acceso manual o semiautomático
4. Esto permite validar demanda antes de invertir en integración de pagos

---

## Variables de entorno requeridas

```
# PayU (Colombia)
PAYU_API_LOGIN=
PAYU_API_KEY=
PAYU_MERCHANT_ID=
PAYU_ACCOUNT_ID=

# PayPal
PAYPAL_CLIENT_ID=
PAYPAL_CLIENT_SECRET=

# Stripe (secundario)
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=

# Precios (IDs de productos en cada gateway)
STARTER_PRICE_USD=97
PRO_PRICE_USD=297
AGENCY_PRICE_USD=497
```

Usar `.local/skills/environment-secrets/SKILL.md` para configurar.

---

## Tabla `subscriptions` en DB

```typescript
export const subscriptions = pgTable("subscriptions", {
  id: uuid("id").primaryKey().defaultRandom(),
  organizationId: uuid("organization_id").notNull().unique(),
  plan: text("plan").notNull().default("free"),     // free | starter | pro | agency
  billingInterval: text("billing_interval"),         // monthly | annual
  paymentProvider: text("payment_provider"),         // payu | paypal | stripe | manual
  externalSubscriptionId: text("external_subscription_id"),
  externalCustomerId: text("external_customer_id"),
  currentPeriodEnd: timestamp("current_period_end"),
  status: text("status").notNull().default("active"), // active | past_due | canceled
  currency: text("currency").notNull().default("USD"),
  amountCents: integer("amount_cents"),
  createdAt: timestamp("created_at").defaultNow(),
  updatedAt: timestamp("updated_at").defaultNow(),
});
```

---

## Endpoints

```
GET  /api/billing/subscription        → plan y estado actual
POST /api/billing/checkout/payu       → iniciar pago con PayU
POST /api/billing/checkout/paypal     → iniciar pago con PayPal
POST /api/billing/checkout/manual     → registrar intención de pago manual
POST /api/billing/webhooks/payu       → confirmar pagos PayU (sin auth)
POST /api/billing/webhooks/paypal     → confirmar pagos PayPal (sin auth)
GET  /api/billing/invoices            → historial de pagos
```

---

## Flujo PayU (Colombia)

```typescript
// PayU usa un modelo de redirect — el usuario va a la página de PayU y vuelve
async function createPayUCheckout(orgId: string, plan: PlanTier): Promise<string> {
  const amount = PLAN_PRICES[plan];
  const referenceCode = `${orgId}_${plan}_${Date.now()}`;

  // Generar firma MD5
  const signature = md5(`${PAYU_CONFIG.apiKey}~${PAYU_CONFIG.merchantId}~${referenceCode}~${amount}~USD`);

  // Construir URL del formulario PayU (redirección)
  const params = new URLSearchParams({
    merchantId: PAYU_CONFIG.merchantId,
    accountId: PAYU_CONFIG.accountId,
    description: `R-C-T-E-E Pro - Plan ${plan}`,
    referenceCode,
    amount: amount.toString(),
    currency: "USD",
    signature,
    responseUrl: `${process.env.APP_URL}/billing/response`,
    confirmationUrl: `${process.env.APP_URL}/api/billing/webhooks/payu`,
  });

  return `https://checkout.payulatam.com/ppp-web-gateway-payu/?${params}`;
}
```

---

## Precios en COP (Colombia)

Mostrar precios en COP además de USD para usuarios colombianos:

```typescript
// Usar tasa de cambio aproximada — actualizar mensualmente
const USD_TO_COP = 4100; // Actualizar con API de tasa de cambio si se quiere dinámico

const PRICES_COP = {
  starter: 97 * USD_TO_COP,  // ~$397,700 COP
  pro: 297 * USD_TO_COP,     // ~$1,217,700 COP
  agency: 497 * USD_TO_COP,  // ~$2,037,700 COP
};
```

---

## Reglas clave

- **MVP:** Empezar con proceso manual (transferencia + comprobante) — no invertir en integración hasta tener los primeros 5 clientes pagados
- PayU es el proveedor primario para Colombia, no Stripe
- Los webhooks de pago van SIN middleware de autenticación (PayU/PayPal los llaman directamente)
- Al confirmar un pago, actualizar el plan del usuario ANTES de responder 200 al webhook
- Ofrecer descuento del 20% en facturación anual

---

## Referencias

- Ver `freemium-gates` skill para los límites por plan
- Ver `.local/skills/environment-secrets/SKILL.md` para configurar secrets
- Ver `platform-architecture` skill para la estrategia de monetización completa
