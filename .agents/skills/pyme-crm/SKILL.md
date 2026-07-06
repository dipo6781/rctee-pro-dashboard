---
name: pyme-crm
description: CRM de clientes de R-C-T-E-E Pro — gestión de las pymes que contratan servicios de consultoría. Úsala al construir el módulo de clientes, al rastrear proyectos y entregables por cliente, o al implementar seguimiento del pipeline de ventas. El "cliente" aquí es la pyme compradora, no el usuario final del chatbot.
---

# Pyme CRM — R-C-T-E-E Pro

El CRM registra las **pymes clientes** que contratan los servicios del consultor/agencia que usa la plataforma. No es un CRM de atención al cliente masivo — es una agenda de proyectos de consultoría con historial de entregables y pagos.

---

## Modelo de datos

```typescript
// Tabla: clients (pymes que contratan consultoría)
{
  id: uuid,
  organization_id: uuid,          // consultor/agencia dueño de este cliente
  company_name: string,
  industry: Industry,             // sector de la pyme
  company_size: CompanySize,      // "50-100" | "100-250" | "250-500"
  contact_name: string,           // nombre del dueño/gerente
  contact_email: string,
  contact_phone: string,          // formato colombiano: +57 300 XXX XXXX
  contact_whatsapp: string | null,
  city: string,                   // Bogotá, Medellín, Cali, etc.
  status: ClientStatus,
  pipeline_stage: PipelineStage,
  total_billed_usd: decimal,      // suma de todos los proyectos
  notes: text | null,
  source: string | null,          // "linkedin", "referido", "landing", "evento"
  assigned_to: uuid | null,
  created_at, updated_at
}

type Industry =
  | "retail" | "manufactura" | "servicios" | "tecnologia"
  | "construccion" | "salud" | "educacion" | "agro" | "otro"

type CompanySize = "50-100" | "100-250" | "250-500"

type ClientStatus = "prospect" | "active" | "completed" | "paused" | "lost"

type PipelineStage =
  | "lead"              // contacto inicial
  | "diagnostico"       // llamada de diagnóstico agendada/realizada
  | "propuesta"         // propuesta enviada
  | "negociacion"       // en negociación de precio/alcance
  | "anticipo"          // pago de anticipo recibido
  | "en_ejecucion"      // proyecto en curso
  | "entregado"         // entregables entregados
  | "completado"        // pago final recibido, proyecto cerrado
```

---

## Proyectos por cliente

```typescript
// Tabla: client_projects
{
  id: uuid,
  organization_id: uuid,
  client_id: uuid,
  name: string,                   // "Diagnóstico Financiero - Q1 2024"
  vertical: Vertical,             // qué empresa/marca de la plataforma se usó
  status: ProjectStatus,
  start_date: date,
  delivery_date: date | null,
  price_usd: decimal,
  payment_status: PaymentStatus,
  advance_paid_usd: decimal,      // anticipo recibido
  balance_due_usd: decimal,       // saldo pendiente (computed)
  notes: text | null,
  created_at, updated_at
}

type ProjectStatus = "pending" | "in_progress" | "delivered" | "completed" | "canceled"
type PaymentStatus = "pending" | "advance_received" | "fully_paid" | "overdue"
```

---

## Entregables por proyecto

```typescript
// Tabla: deliverables (documentos generados para cada proyecto)
{
  id: uuid,
  organization_id: uuid,
  project_id: uuid,
  client_id: uuid,
  template_id: uuid,              // qué plantilla R-C-T-E-E se usó
  template_name: string,          // copia del nombre (por si se edita la plantilla)
  inputs: jsonb,                  // datos del formulario que generó este doc
  generated_content: text | null, // texto crudo de la IA (para regenerar formato)
  file_url: string | null,        // URL firmada en Supabase Storage
  file_format: "pdf" | "word" | "excel",
  file_size_bytes: integer | null,
  generation_status: "queued" | "processing" | "completed" | "failed",
  generation_attempts: integer,
  shared_with_client: boolean,    // si se envió el link al cliente
  shared_at: timestamp | null,
  created_at, updated_at
}
```

---

## Pipeline de ventas (vista Kanban sugerida)

```
Lead → Diagnóstico → Propuesta → Anticipo → En Ejecución → Entregado → Completado
```

Cada columna muestra: nombre de la empresa, vertical, monto del proyecto, fecha estimada.

---

## Endpoints

```
GET    /api/clients                       → lista con filtros (status, pipeline_stage, industry)
GET    /api/clients/:id                   → detalle + proyectos + entregables + pagos
POST   /api/clients                       → crear cliente
PUT    /api/clients/:id                   → actualizar
POST   /api/clients/:id/projects          → crear proyecto para el cliente
GET    /api/clients/:id/projects          → proyectos del cliente
POST   /api/projects/:id/deliverables     → generar entregable (dispara job de IA)
GET    /api/projects/:id/deliverables     → listar entregables
POST   /api/deliverables/:id/share        → generar link de descarga para el cliente
GET    /api/pipeline                      → vista Kanban de todos los proyectos activos
GET    /api/crm/summary                   → métricas resumen (ingresos, proyectos activos, etc.)
```

---

## Compartir entregable con el cliente

Cuando el consultor termina un entregable, puede enviarlo al cliente pyme directamente:

```typescript
// POST /api/deliverables/:id/share
// Genera URL firmada de Supabase con expiración de 30 días
// Envía WhatsApp/email al cliente con el link

async function shareDeliverable(deliverableId: string, orgId: string) {
  const deliverable = await getDeliverable(deliverableId, orgId);
  const signedUrl = await supabase.storage
    .from("deliverables")
    .createSignedUrl(deliverable.storagePath, 60 * 60 * 24 * 30); // 30 días

  // Enviar por WhatsApp si tiene número configurado
  await sendWhatsAppMessage(deliverable.client.whatsapp, {
    type: "document",
    url: signedUrl,
    filename: deliverable.filename,
    caption: `Aquí está su ${deliverable.templateName}. Cualquier pregunta, con gusto.`,
  });

  await db.update(deliverables).set({
    sharedWithClient: true,
    sharedAt: new Date(),
  }).where(eq(deliverables.id, deliverableId));
}
```

---

## Métricas de negocio del consultor

```typescript
// GET /api/crm/summary
interface CRMSummary {
  revenue: {
    total_billed_usd: number;       // total facturado histórico
    collected_usd: number;          // efectivamente cobrado
    pending_usd: number;            // por cobrar (saldo)
    this_month_usd: number;         // facturado este mes
  };
  pipeline: {
    active_projects: number;
    projects_by_stage: Record<PipelineStage, number>;
    avg_deal_size_usd: number;
  };
  deliverables: {
    total_generated: number;
    by_vertical: Record<Vertical, number>;
    this_month: number;
  };
  clients: {
    total: number;
    active: number;
    new_this_month: number;
  };
}
```

---

## Límites por plan

| Plan | Clientes | Proyectos activos | Entregables/mes |
|---|---|---|---|
| Free | 5 | 2 | 2 |
| Starter | 50 | 10 | 10 |
| Pro | Ilimitados | Ilimitados | Ilimitados |
| Agencia | Ilimitados | Ilimitados | Ilimitados |

---

## WhatsApp como canal primario (Colombia)

WhatsApp es el canal de comunicación dominante en Colombia. Integrar envío de entregables vía WhatsApp Business API (Meta) desde el primer release:

```
Casos de uso:
- Enviar PDF/Word al cliente cuando el entregable está listo
- Recordatorios de pago de anticipo o saldo
- Notificación de inicio de proyecto
```

Ver `integration-connectors` skill para implementación de WhatsApp Cloud API.

---

## Reglas clave

- El campo `total_billed_usd` se actualiza automáticamente al crear/completar proyectos
- `balance_due_usd = price_usd - advance_paid_usd` — computado en el modelo, no en la DB
- Al compartir un entregable, guardar `shared_at` para saber si el cliente ya lo recibió
- Los archivos en Supabase Storage usan paths con `organization_id` para aislamiento
- El teléfono se guarda en formato E.164 (+57 300...) para WhatsApp API

---

## Referencias

- Ver `platform-architecture` skill para los 7 verticales
- Ver `solution-templates-lib` skill para la relación plantilla → entregable
- Ver `integration-connectors` skill para WhatsApp y envío de documentos
- Ver `freemium-gates` skill para límites por plan
