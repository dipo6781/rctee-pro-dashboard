---
name: integration-connectors
description: Integraciones externas de R-C-T-E-E Pro. Úsala al construir el módulo de integraciones, al conectar WhatsApp para envío de entregables, al integrar PayU para pagos en Colombia, o al agregar webhooks de entrada/salida. Canal primario en Colombia: WhatsApp Business.
---

# Integration Connectors — R-C-T-E-E Pro

Las integraciones conectan la plataforma con los canales de comunicación y pago que usan las pymes colombianas. Canal dominante: WhatsApp. Pago dominante: transferencia bancaria / PayU.

---

## Integraciones prioritarias para Colombia

| Integración | Prioridad | Caso de uso |
|---|---|---|
| WhatsApp Business Cloud API | 🔴 Alta | Enviar entregables al cliente pyme, notificaciones |
| Email (SendGrid / SMTP) | 🔴 Alta | Respaldo a WhatsApp, facturas, notificaciones |
| PayU Latam | 🔴 Alta | Cobros en Colombia (PSE, tarjetas, efectivo) |
| PayPal | 🟡 Media | Clientes internacionales o que prefieren USD |
| Google Drive | 🟡 Media | Guardar entregables en el Drive del consultor |
| Calendly | 🟢 Baja | Agendar llamada de diagnóstico desde landing page |
| Stripe | 🟢 Baja | Solo si el usuario lo solicita explícitamente |
| n8n / Zapier / Make | 🟢 Baja | Automatizaciones avanzadas (plan Pro+) |

---

## WhatsApp Business Cloud API (Meta)

Canal primario para Colombia. Permite enviar documentos PDF directamente al cliente pyme.

### Setup

```typescript
// Variables de entorno:
// WHATSAPP_PHONE_NUMBER_ID — ID del número de WhatsApp Business
// WHATSAPP_ACCESS_TOKEN — token permanente de Meta Business
// WHATSAPP_WEBHOOK_VERIFY_TOKEN — token para verificar el webhook

const WHATSAPP_API_URL = `https://graph.facebook.com/v18.0/${process.env.WHATSAPP_PHONE_NUMBER_ID}/messages`;
```

### Enviar documento al cliente

```typescript
async function sendWhatsAppDocument(
  recipientPhone: string, // formato E.164: +573001234567
  documentUrl: string,    // URL firmada del Supabase Storage (30 días)
  filename: string,       // "Diagnostico_Financiero_EmpresaXYZ.pdf"
  caption: string         // "Aquí está su diagnóstico financiero. Con gusto resolvemos sus dudas."
): Promise<void> {
  const response = await fetch(WHATSAPP_API_URL, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${process.env.WHATSAPP_ACCESS_TOKEN}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      messaging_product: "whatsapp",
      recipient_type: "individual",
      to: recipientPhone,
      type: "document",
      document: {
        link: documentUrl,
        filename,
        caption,
      },
    }),
  });

  if (!response.ok) {
    const error = await response.json();
    throw new Error(`WhatsApp send failed: ${JSON.stringify(error)}`);
  }
}
```

### Enviar mensaje de texto

```typescript
async function sendWhatsAppText(
  recipientPhone: string,
  message: string
): Promise<void> {
  await fetch(WHATSAPP_API_URL, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${process.env.WHATSAPP_ACCESS_TOKEN}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      messaging_product: "whatsapp",
      to: recipientPhone,
      type: "text",
      text: { body: message },
    }),
  });
}
```

### Webhook de WhatsApp (recibir mensajes)

```typescript
// GET /api/webhooks/whatsapp — verificación del webhook
app.get("/api/webhooks/whatsapp", (req, res) => {
  const mode = req.query["hub.mode"];
  const token = req.query["hub.verify_token"];
  const challenge = req.query["hub.challenge"];

  if (mode === "subscribe" && token === process.env.WHATSAPP_WEBHOOK_VERIFY_TOKEN) {
    res.status(200).send(challenge);
  } else {
    res.sendStatus(403);
  }
});

// POST /api/webhooks/whatsapp — recibir mensajes entrantes (SIN auth)
app.post("/api/webhooks/whatsapp", (req, res) => {
  res.sendStatus(200); // responder 200 rápido a Meta
  processWhatsAppWebhook(req.body).catch(err =>
    logger.error({ err }, "WhatsApp webhook processing failed")
  );
});
```

---

## Email (SendGrid)

Respaldo a WhatsApp para entrega de entregables y notificaciones.

```typescript
// Variables de entorno: SENDGRID_API_KEY, FROM_EMAIL

import sgMail from "@sendgrid/mail";
sgMail.setApiKey(process.env.SENDGRID_API_KEY!);

async function sendDeliverableEmail(
  to: string,
  clientName: string,
  deliverableName: string,
  downloadUrl: string
): Promise<void> {
  await sgMail.send({
    to,
    from: { email: process.env.FROM_EMAIL!, name: "R-C-T-E-E Pro" },
    subject: `Su ${deliverableName} está listo`,
    html: `
      <p>Estimado/a ${clientName},</p>
      <p>Su <strong>${deliverableName}</strong> está listo para descargarse.</p>
      <p><a href="${downloadUrl}" style="padding:12px 24px;background:#1B4F72;color:white;text-decoration:none;border-radius:4px;">
        Descargar documento
      </a></p>
      <p>El enlace es válido por 30 días.</p>
    `,
  });
}
```

---

## Google Drive (guardar entregables automáticamente)

```typescript
// OAuth con Google — el consultor conecta su Drive
// Al generar un entregable, opcionalmente subirlo a su carpeta de Drive

// Variables de entorno (por org, encriptadas):
// GOOGLE_ACCESS_TOKEN, GOOGLE_REFRESH_TOKEN, GOOGLE_DRIVE_FOLDER_ID

async function uploadToDrive(
  buffer: Buffer,
  filename: string,
  mimeType: string,
  folderId: string,
  accessToken: string
): Promise<string> {
  const form = new FormData();
  form.append("metadata", JSON.stringify({
    name: filename,
    parents: [folderId],
  }), { contentType: "application/json" });
  form.append("file", buffer, { contentType: mimeType });

  const response = await fetch(
    "https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart",
    {
      method: "POST",
      headers: { "Authorization": `Bearer ${accessToken}`, ...form.getHeaders() },
      body: form,
    }
  );

  const data = await response.json();
  return data.id; // Google Drive file ID
}
```

---

## Modelo de datos de integraciones

```typescript
// Tabla: integrations
{
  id: uuid,
  organization_id: uuid,
  type: IntegrationType,           // "whatsapp" | "email" | "google_drive" | etc.
  name: string,                    // "WhatsApp de la empresa"
  is_active: boolean,
  config: jsonb,                   // datos no-sensibles (phone_number_id, folder_id)
  credentials_encrypted: text,     // tokens y API keys encriptados (AES-256-GCM)
  last_used_at: timestamp | null,
  created_at, updated_at
}
```

**Encriptación de credenciales:** Ver `multi-tenancy` skill para el patrón de encriptación.

---

## Mensajes predefinidos por vertical (plantillas de WhatsApp)

Cada vertical tiene mensajes predefinidos en español colombiano:

```typescript
const WHATSAPP_TEMPLATES = {
  reintegra_ai: {
    deliverable_ready: (clientName: string, docName: string) =>
      `Hola ${clientName} 👋\n\nSu *${docName}* está listo. Lo encontrará adjunto.\n\nCualquier pregunta sobre los resultados, con mucho gusto le ayudamos. 🤝`,
    payment_reminder: (clientName: string, amount: string, dueDate: string) =>
      `Hola ${clientName}, le recordamos que el saldo de *${amount}* vence el *${dueDate}*. ¿Cómo podemos ayudarle a gestionar este pago?`,
  },
  kineticorp_bienestar: {
    deliverable_ready: (clientName: string, docName: string) =>
      `Hola ${clientName} 😊\n\nSu *${docName}* está listo. Con él tiene todo lo necesario para iniciar su programa de bienestar este mes.\n\n¡Éxito en la implementación! 🌿`,
  },
};
```

---

## Límites por plan

| Plan | Integraciones activas |
|---|---|
| Free | Solo email básico |
| Starter | WhatsApp + Email |
| Pro | Todas las integraciones |
| Agency | Todas + API propia para conectar sistemas del cliente |

---

## Variables de entorno requeridas

```
WHATSAPP_PHONE_NUMBER_ID=
WHATSAPP_ACCESS_TOKEN=
WHATSAPP_WEBHOOK_VERIFY_TOKEN=
SENDGRID_API_KEY=
FROM_EMAIL=noreply@rcteepro.com
CREDENTIALS_ENCRYPTION_KEY=  # 32 bytes hex para encriptar tokens de terceros
```

---

## Reglas clave

- **Nunca loguear** tokens, API keys ni contenido de mensajes de WhatsApp
- Los webhooks de entrada (WhatsApp, PayU) responden **200 inmediatamente** y procesan en background
- Las credenciales de integraciones (tokens de Google, etc.) se guardan **siempre encriptadas** en DB
- WhatsApp requiere que los números estén en formato E.164 (+57...)
- Las plantillas de mensaje de WhatsApp deben aprobarse previamente en Meta Business Manager para mensajes iniciados por el negocio

---

## Referencias

- Ver `solution-deploy-flow` skill para el flujo de compartir entregables por WhatsApp
- Ver `pyme-crm` skill para los datos del cliente (teléfono, email)
- Ver `platform-monetization` skill para la integración de PayU
- Ver `.local/skills/environment-secrets/SKILL.md` para configurar secrets
