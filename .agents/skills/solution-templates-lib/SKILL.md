---
name: solution-templates-lib
description: Sistema de biblioteca de plantillas de soluciones para la plataforma. Úsala al construir el módulo de plantillas, al permitir que los usuarios exporten sus proyectos como plantillas, o al implementar el marketplace de plantillas públicas.
---

# Solution Templates Library

Las plantillas son proyectos "congelados" que otros usuarios pueden clonar como punto de partida. Cada plantilla incluye configuración de agentes, estructura del proyecto, integraciones y ajustes preconfigurados.

---

## Modelo de datos

```typescript
// Tabla: templates
{
  id: uuid,
  organization_id: uuid | null,   // null = plantilla de la plataforma
  created_by: uuid,               // usuario que la creó
  name: string,
  description: text,
  category: TemplateCategory,
  thumbnail_url: string,
  is_public: boolean,             // visible en marketplace
  is_featured: boolean,           // destacada por la plataforma
  use_count: integer,             // cuántas veces fue clonada
  tags: text[],
  snapshot: jsonb,                // snapshot del proyecto (ver abajo)
  required_plan: PlanTier,        // plan mínimo para usar esta plantilla
  price_cents: integer,           // 0 = gratis, >0 = de pago (marketplace futuro)
  created_at, updated_at
}
```

---

## Categorías de plantillas (`TemplateCategory`)

```typescript
type TemplateCategory =
  | "chatbot"           // atención al cliente, FAQ
  | "sales"             // CRM ligero, seguimiento de leads
  | "automation"        // flujos automáticos de tareas
  | "collections"       // cobranza y seguimiento de pagos
  | "events"            // registro y gestión de eventos
  | "onboarding"        // bienvenida y activación de clientes
  | "analytics"         // reportes y métricas
  | "custom"            // categoría personalizada por el usuario
```

---

## Snapshot de un proyecto (estructura del `jsonb`)

```typescript
interface ProjectSnapshot {
  version: "1.0",
  project: {
    name: string,
    description: string,
    settings: Record<string, unknown>,
  },
  agents: Array<{
    name: string,
    type: AgentType,
    model: string,
    system_prompt: string,
    temperature: number,
    tools_enabled: AgentTool[],
  }>,
  integrations: Array<{
    type: string,
    config: Record<string, unknown>, // sin credentials — solo estructura
  }>,
  pages?: Array<{   // para soluciones con UI pública
    slug: string,
    title: string,
    layout: string,
  }>,
}
```

**Nunca incluir en el snapshot:** API keys, tokens, datos de clientes, mensajes de conversaciones.

---

## Flujo: exportar proyecto como plantilla

```
1. Usuario clic "Guardar como plantilla"
2. Sistema genera snapshot del proyecto (sanitizado — sin credenciales)
3. Usuario completa: nombre, descripción, categoría, thumbnail, visibilidad
4. Sistema guarda en tabla `templates`
5. Si is_public=true y plan free → mostrar error (solo Starter/Pro pueden publicar)
```

---

## Flujo: clonar una plantilla

```
1. Usuario selecciona plantilla del catálogo
2. Sistema verifica: ¿el plan del usuario cubre `required_plan` de la plantilla?
3. Sistema verifica: ¿el usuario tiene cupo para un proyecto más? (freemium-gates)
4. Crear nuevo proyecto en la org del usuario
5. Crear agentes, integraciones y configuraciones desde el snapshot
6. Incrementar `use_count` en la plantilla original
7. Redirigir al nuevo proyecto
```

---

## Endpoints

```
GET  /api/templates                → listar plantillas (filtros: category, is_public, tags)
GET  /api/templates/:id            → detalle de plantilla
POST /api/templates                → crear plantilla desde proyecto
POST /api/templates/:id/clone      → clonar plantilla al workspace del usuario
PUT  /api/templates/:id            → actualizar plantilla propia
DELETE /api/templates/:id          → archivar plantilla propia
GET  /api/templates/featured       → plantillas destacadas (para home del marketplace)
```

---

## Reglas clave

- Las plantillas de la plataforma tienen `organization_id = null` y solo el sistema las puede crear/editar
- Un usuario en plan Free puede usar plantillas públicas pero NO publicar las suyas
- Al clonar, los agentes se crean con `is_active = false` hasta que el usuario configure sus credenciales
- El `system_prompt` de los agentes se clona tal cual — el usuario puede editarlo después
- Las plantillas con `price_cents > 0` son para el marketplace de pago (fase futura) — reservar el campo desde ahora

---

## Plantillas de la plataforma (seed inicial)

Crear estas plantillas base al hacer el seed de la DB:

| Nombre | Categoría | Descripción |
|---|---|---|
| Chatbot de Soporte | chatbot | Atiende preguntas frecuentes con IA |
| Seguimiento de Cobranza | collections | Recordatorios automáticos de pagos pendientes |
| Registro de Eventos | events | Formulario + confirmación por email |
| Calificación de Leads | sales | Chatbot que califica y agenda demos |
| Onboarding de Clientes | onboarding | Flujo de bienvenida paso a paso |

---

## Referencias

- Ver `platform-architecture` skill para estructura de datos completa
- Ver `freemium-gates` skill para restricciones por plan
- Ver `ai-agent-config` skill para el formato de los agentes en el snapshot
