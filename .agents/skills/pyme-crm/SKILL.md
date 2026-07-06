---
name: pyme-crm
description: CRM ligero para gestión de clientes finales dentro de la plataforma. Úsala al construir el módulo de CRM, al conectar clientes a proyectos/soluciones, o al implementar seguimiento de interacciones para pymes.
---

# Pyme CRM

CRM orientado a pymes: simple, rápido y conectado a los agentes IA de la plataforma. El objetivo es que el usuario de la plataforma pueda gestionar a sus propios clientes finales y que los agentes puedan consultar y actualizar estos datos.

---

## Modelo de datos

```typescript
// Tabla: clients
{
  id: uuid,
  organization_id: uuid,
  name: string,
  email: string | null,
  phone: string | null,               // formato E.164 (+5491112345678)
  company: string | null,
  tags: text[],                       // etiquetas libres (ej: ["lead", "vip", "deudor"])
  status: ClientStatus,
  assigned_to: uuid | null,           // miembro del equipo responsable
  source: string | null,              // de dónde llegó ("chatbot", "formulario", "manual")
  custom_fields: jsonb,               // campos personalizados por la org
  notes: text | null,
  last_contacted_at: timestamp | null,
  created_at, updated_at
}

type ClientStatus =
  | "lead"          // posible cliente, no confirmado
  | "active"        // cliente activo
  | "inactive"      // sin actividad reciente
  | "at_risk"       // señales de churn
  | "lost"          // perdido / canceló

// Tabla: client_interactions (historial de contacto)
{
  id: uuid,
  organization_id: uuid,
  client_id: uuid,
  type: InteractionType,
  summary: text,                      // resumen del contacto
  agent_id: uuid | null,              // si fue generado por un agente IA
  created_by: uuid | null,            // si fue creado manualmente por un usuario
  metadata: jsonb,                    // datos adicionales (ej: monto cobrado, evento asistido)
  created_at: timestamp,
}

type InteractionType =
  | "chat"          // conversación con agente IA
  | "email"         // email enviado
  | "call"          // llamada registrada
  | "note"          // nota manual
  | "payment"       // pago registrado
  | "event"         // asistencia a evento
  | "form"          // formulario completado
```

---

## Endpoints

```
GET    /api/clients                     → listar clientes (filtros: status, tags, assigned_to, search)
GET    /api/clients/:id                 → detalle del cliente + historial de interacciones
POST   /api/clients                     → crear cliente
PUT    /api/clients/:id                 → actualizar cliente
DELETE /api/clients/:id                 → archivar cliente
POST   /api/clients/:id/interactions    → registrar interacción manual
GET    /api/clients/export              → exportar a CSV (Starter/Pro)
POST   /api/clients/import              → importar desde CSV (Starter/Pro)
```

---

## Integración con agentes IA

Los agentes con la tool `query_crm` pueden leer y escribir en el CRM:

```typescript
// Tool: query_crm
// El agente puede hacer:
// - Buscar clientes por email, teléfono o nombre
// - Ver el historial de interacciones de un cliente
// - Crear un cliente nuevo (cuando el chatbot captura un lead)
// - Registrar una interacción
// - Actualizar el status de un cliente

// Siempre scoped a organization_id del proyecto — nunca acceso cross-tenant
```

Cuando un agente crea un cliente nuevo (ej: un lead capturado por chatbot), registrar:
- `source = "chatbot"`
- `agent_id` en la interacción
- `status = "lead"`

---

## Campos personalizados (`custom_fields`)

Cada organización puede definir sus propios campos:

```typescript
// Tabla: client_field_definitions
{
  id: uuid,
  organization_id: uuid,
  field_key: string,          // "fecha_vencimiento", "monto_deuda"
  field_label: string,        // "Fecha de vencimiento"
  field_type: "text" | "number" | "date" | "boolean" | "select",
  options: string[] | null,   // para tipo "select"
  is_required: boolean,
}
```

Los valores se guardan en `clients.custom_fields` como jsonb: `{ "fecha_vencimiento": "2024-03-01" }`.

---

## Segmentación y filtros

Filtros disponibles en la lista de clientes:
- `status` — lead / active / inactive / at_risk / lost
- `tags` — filtro por etiqueta (AND o OR)
- `assigned_to` — por miembro del equipo
- `search` — búsqueda por nombre, email, teléfono, empresa
- `source` — de dónde proviene el cliente
- `last_contacted_before` — para encontrar clientes sin contacto reciente
- `created_after` / `created_before`

---

## Cobranza (caso de uso pyme frecuente)

Para pymes que usan la plataforma para cobranza, los campos clave son:
- `status: "at_risk"` → clientes con pagos vencidos
- `custom_fields.monto_deuda` → deuda pendiente
- `interactions` con `type: "payment"` → historial de pagos
- Agente con `type: "collector"` → automatiza recordatorios

Flujo típico:
```
1. Importar deudores desde CSV
2. Agente "collector" envía WhatsApp/email con recordatorio
3. Registrar interacción con resultado
4. Si el cliente paga → registrar interaction type "payment" + actualizar status
```

---

## Límites por plan

| Plan | Clientes en CRM | Exportación CSV | Campos personalizados |
|---|---|---|---|
| Free | 50 | No | 0 |
| Starter | 500 | Sí | 5 |
| Pro | Ilimitados | Sí | Ilimitados |

---

## Reglas clave

- Los clientes del CRM son de la organización, no del usuario individual — toda la org puede verlos
- `last_contacted_at` se actualiza automáticamente al registrar cualquier interacción
- Al archivar un cliente, no borrar — cambiar `status = "lost"` y ocultar de la lista por defecto
- Los teléfonos se guardan en formato E.164 para compatibilidad con WhatsApp API
- La búsqueda full-text debe indexar `name`, `email`, `company` (índice GIN en PostgreSQL)

---

## Índices recomendados

```sql
-- Búsqueda full-text
CREATE INDEX idx_clients_search ON clients
  USING GIN(to_tsvector('spanish', name || ' ' || COALESCE(email, '') || ' ' || COALESCE(company, '')));

-- Filtros frecuentes
CREATE INDEX idx_clients_org_status ON clients(organization_id, status);
CREATE INDEX idx_clients_org_tags ON clients USING GIN(organization_id, tags);
```

---

## Referencias

- Ver `platform-architecture` skill para estructura de datos completa
- Ver `multi-tenancy` skill para aislamiento por organización
- Ver `ai-agent-config` skill para la tool `query_crm`
- Ver `freemium-gates` skill para límites por plan
- Ver `integration-connectors` skill para WhatsApp/email desde el CRM
