---
name: integration-connectors
description: Sistema de integraciones externas para la plataforma AI. Úsala al construir el módulo de integraciones, al agregar un conector nuevo (n8n, Zapier, email, WhatsApp, etc.), o al implementar webhooks entrantes/salientes.
---

# Integration Connectors

Las integraciones conectan los proyectos de los usuarios con herramientas externas. Hay dos tipos: **webhooks** (la plataforma envía/recibe eventos) y **conectores** (integraciones OAuth o API key con servicios específicos).

---

## Modelo de datos

```typescript
// Tabla: integrations
{
  id: uuid,
  organization_id: uuid,
  project_id: uuid,
  type: IntegrationType,
  name: string,              // nombre visible al usuario
  config: jsonb,             // configuración no-sensible (URL, channel, etc.)
  credentials: jsonb,        // ENCRIPTADO — API keys, tokens
  status: "active" | "error" | "disabled",
  last_triggered_at: timestamp,
  error_message: text | null,
  created_at, updated_at
}

// Tabla: webhook_logs (auditoría de webhooks)
{
  id: uuid,
  integration_id: uuid,
  organization_id: uuid,
  direction: "inbound" | "outbound",
  payload: jsonb,
  status_code: integer,
  duration_ms: integer,
  created_at: timestamp,
}
```

---

## Tipos de integración (`IntegrationType`)

```typescript
type IntegrationType =
  // Automatización
  | "n8n_webhook"           // trigger a n8n workflow
  | "zapier_webhook"        // trigger a Zapier zap
  | "make_webhook"          // trigger a Make.com scenario
  // Comunicación
  | "whatsapp_cloud"        // WhatsApp Business API (Meta)
  | "twilio_sms"            // SMS via Twilio
  | "sendgrid_email"        // Email via SendGrid
  | "slack_webhook"         // Notificaciones a Slack
  // Datos
  | "google_sheets"         // Leer/escribir hojas de cálculo
  | "airtable"              // Base de datos Airtable
  // Pagos
  | "stripe_webhook"        // Recibir eventos de Stripe del usuario
  | "mercadopago"           // Pagos en LATAM
  // Custom
  | "custom_webhook"        // URL personalizada (outbound)
  | "inbound_webhook"       // Endpoint generado para recibir datos
```

---

## Webhooks salientes (outbound)

```typescript
// src/lib/integrations/webhook.ts
export async function triggerWebhook(
  integration: Integration,
  payload: Record<string, unknown>
): Promise<void> {
  const start = Date.now();
  let statusCode = 0;

  try {
    const res = await fetch(integration.config.url, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "X-Platform-Event": "true",
        "X-Organization-Id": integration.organizationId,
        ...(integration.config.secret
          ? { "X-Webhook-Secret": integration.config.secret }
          : {}),
      },
      body: JSON.stringify({
        event: payload.event,
        timestamp: new Date().toISOString(),
        data: payload.data,
      }),
      signal: AbortSignal.timeout(10000), // 10s timeout
    });
    statusCode = res.status;
  } catch (err) {
    // Registrar error pero no bloquear el flujo principal
    logger.error({ err, integrationId: integration.id }, "Webhook failed");
  } finally {
    await logWebhook(integration.id, integration.organizationId, "outbound", payload, statusCode, Date.now() - start);
  }
}
```

---

## Webhooks entrantes (inbound)

Cada integración inbound genera una URL única:
`POST /api/webhooks/inbound/:token`

```typescript
// El token es un UUID opaco — no expone IDs internos
// Mapear token → integration_id en tabla `inbound_webhook_tokens`

app.post("/api/webhooks/inbound/:token", async (req, res) => {
  const integration = await resolveInboundToken(req.params.token);
  if (!integration) return res.status(404).end();

  // Disparar el agente asociado con el payload recibido
  await enqueueAgentJob(integration.agentId, req.body);

  res.json({ received: true });
});
```

---

## Encriptación de credenciales

Las API keys y tokens de terceros se guardan encriptados con AES-256-GCM:

```typescript
import { createCipheriv, createDecipheriv, randomBytes } from "crypto";

const ENCRYPTION_KEY = Buffer.from(process.env.CREDENTIALS_ENCRYPTION_KEY!, "hex"); // 32 bytes

export function encryptCredentials(data: Record<string, string>): string {
  const iv = randomBytes(16);
  const cipher = createCipheriv("aes-256-gcm", ENCRYPTION_KEY, iv);
  const encrypted = Buffer.concat([cipher.update(JSON.stringify(data), "utf8"), cipher.final()]);
  const tag = cipher.getAuthTag();
  return Buffer.concat([iv, tag, encrypted]).toString("base64");
}

export function decryptCredentials(encrypted: string): Record<string, string> {
  const buf = Buffer.from(encrypted, "base64");
  const iv = buf.subarray(0, 16);
  const tag = buf.subarray(16, 32);
  const data = buf.subarray(32);
  const decipher = createDecipheriv("aes-256-gcm", ENCRYPTION_KEY, iv);
  decipher.setAuthTag(tag);
  return JSON.parse(decipher.update(data) + decipher.final("utf8"));
}
```

**Variable de entorno requerida:** `CREDENTIALS_ENCRYPTION_KEY` — 32 bytes aleatorios en hex.

---

## Límites por plan (ver freemium-gates)

| Plan | Integraciones activas |
|---|---|
| Free | 0 |
| Starter | 3 |
| Pro | Ilimitadas |

---

## Checklist al agregar un conector nuevo

- [ ] Agregar el tipo a `IntegrationType`
- [ ] Crear un schema de validación Zod para su `config` y `credentials`
- [ ] Implementar un test de conexión (`POST /api/integrations/:id/test`)
- [ ] Documentar los eventos que puede emitir/recibir
- [ ] Agregar al UI del catálogo de integraciones con ícono y descripción

---

## Reglas clave

- **Nunca** loguear credenciales o payloads de webhook con datos sensibles
- Los webhooks salientes tienen timeout de 10s — no usar para operaciones largas
- Guardar los últimos 100 logs de webhook por integración (rotar automáticamente)
- Al desactivar una integración, mantener los logs históricos
- `CREDENTIALS_ENCRYPTION_KEY` debe estar en secrets, nunca en código

---

## Referencias

- Ver `platform-architecture` skill para estructura general
- Ver `freemium-gates` skill para límites por plan
- Ver `multi-tenancy` skill para aislamiento por organización
- Leer `.local/skills/environment-secrets/SKILL.md` para configurar `CREDENTIALS_ENCRYPTION_KEY`
