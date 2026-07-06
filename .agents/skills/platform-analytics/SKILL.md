---
name: platform-analytics
description: Analytics y métricas de negocio para R-C-T-E-E Pro. Úsala al construir el dashboard de métricas, al instrumentar eventos de uso, o al implementar reportes de actividad. Las métricas core son: documentos generados, ingresos por vertical, conversión freemium→pago, y pipeline de clientes.
---

# Platform Analytics — R-C-T-E-E Pro

Dos tipos de métricas: **métricas de plataforma** (qué hacen los consultores/agencias que usan el panel) y **métricas de negocio del consultor** (su pipeline, ingresos, clientes).

---

## Eventos a instrumentar

```typescript
// Tabla: analytics_events (append-only, nunca UPDATE)
{
  id: uuid,
  organization_id: uuid,
  event_type: AnalyticsEventType,
  properties: jsonb,
  user_id: uuid | null,
  created_at: timestamp,
}

type AnalyticsEventType =
  // Uso de plantillas
  | "template.viewed"              // vio la descripción de una plantilla
  | "template.form_started"        // empezó a llenar el formulario
  | "template.generation_started"  // disparó la generación
  | "template.generation_completed"// documento generado exitosamente
  | "template.generation_failed"   // falló la generación
  | "template.downloaded"          // descargó el documento
  | "template.shared_with_client"  // envió el doc al cliente pyme
  // Límites (señales de conversión)
  | "limit.generations_reached"    // llegó al límite mensual
  | "limit.template_locked"        // intentó usar plantilla de plan superior
  | "limit.clients_reached"        // llegó al límite de CRM
  // Monetización
  | "billing.upgrade_clicked"      // hizo clic en "Actualizar plan"
  | "billing.checkout_started"     // inició el proceso de pago
  | "billing.payment_completed"    // pago exitoso
  | "billing.payment_failed"       // pago fallido
  // CRM / Pipeline
  | "client.created"
  | "project.created"
  | "project.stage_advanced"       // el proyecto avanzó de etapa en el pipeline
  | "deliverable.shared"           // entregable compartido con cliente pyme
  // Retención
  | "session.started"              // usuario inició sesión
  | "vertical.switched"            // cambió de vertical en el panel
```

---

## Helper de tracking (fire-and-forget)

```typescript
// src/lib/analytics.ts
export function track(
  event: AnalyticsEventType,
  orgId: string,
  properties: Record<string, unknown> = {},
  userId?: string
): void {
  // Sin await — nunca bloquear el flujo principal
  db.insert(analyticsEvents).values({
    organizationId: orgId,
    eventType: event,
    properties,
    userId: userId ?? null,
  }).catch(err => logger.error({ err }, "Analytics track failed"));
}
```

---

## Dashboard del consultor — métricas clave

```typescript
// GET /api/analytics/summary?period=30d
interface AnalyticsSummary {
  // Producción
  documents_generated: number;          // este período
  documents_by_vertical: Record<Vertical, number>;
  avg_generation_time_seconds: number;

  // Negocio
  revenue_this_month_usd: number;
  revenue_by_vertical: Record<Vertical, number>;
  pipeline_value_usd: number;           // suma de proyectos activos

  // Clientes
  new_clients: number;
  clients_by_stage: Record<PipelineStage, number>;
  avg_deal_size_usd: number;

  // Uso de la plataforma
  most_used_templates: Array<{ template_name: string; count: number }>;
  plan: PlanTier;
  generations_used_this_month: number;
  generations_limit: number;            // -1 = ilimitado
}
```

---

## Métricas de la plataforma (admin interno)

Solo accesible por el equipo de R-C-T-E-E Pro — NO por usuarios normales:

```typescript
// GET /api/admin/platform-metrics (requiere rol admin de plataforma, no de org)
interface PlatformMetrics {
  mrr_usd: number;                      // Monthly Recurring Revenue
  arr_usd: number;                      // Annualized
  users_by_plan: Record<PlanTier, number>;
  free_to_paid_conversion_rate: number; // % de free → cualquier plan pago
  churn_rate_monthly: number;           // % cancelaciones este mes
  most_used_verticals: Array<{ vertical: Vertical; count: number }>;
  top_converting_features: Array<{      // qué features triggean upgrades
    event: string;
    upgrade_rate: number;
  }>;
  documents_generated_total: number;
  avg_documents_per_org_monthly: number;
}
```

---

## Serie temporal (para gráficos)

```sql
-- Documentos generados por día, últimos 30 días
SELECT
  DATE_TRUNC('day', created_at AT TIME ZONE 'America/Bogota') AS day,
  COUNT(*) AS count
FROM analytics_events
WHERE organization_id = $1
  AND event_type = 'template.generation_completed'
  AND created_at >= NOW() - INTERVAL '30 days'
GROUP BY 1
ORDER BY 1;
```

---

## Embudo de conversión (métrica clave para el negocio)

Traquear el embudo completo por plantilla:

```
template.viewed → form_started → generation_started → generation_completed → downloaded → shared_with_client
```

```sql
-- Tasa de conversión por etapa del embudo
SELECT
  event_type,
  COUNT(DISTINCT organization_id) AS orgs,
  COUNT(*) AS events
FROM analytics_events
WHERE event_type IN (
  'template.viewed', 'template.form_started',
  'template.generation_started', 'template.generation_completed',
  'template.downloaded', 'template.shared_with_client'
)
  AND created_at >= NOW() - INTERVAL '30 days'
GROUP BY 1
ORDER BY 2 DESC;
```

---

## Métricas de conversión freemium → pago

```sql
-- ¿Qué feature triggeó más upgrades? (señal de valor real)
SELECT
  properties->>'resource' AS blocked_resource,
  COUNT(*) AS times_blocked,
  COUNT(*) FILTER (WHERE event_type = 'billing.upgrade_clicked') AS upgrades
FROM analytics_events
WHERE event_type IN ('limit.generations_reached', 'limit.template_locked', 'billing.upgrade_clicked')
  AND created_at >= NOW() - INTERVAL '30 days'
GROUP BY 1
ORDER BY 2 DESC;
```

---

## Endpoints

```
GET /api/analytics/summary            → métricas clave del período (dashboard consultor)
GET /api/analytics/timeseries         → serie temporal de documentos generados
GET /api/analytics/funnel             → embudo por plantilla
GET /api/analytics/pipeline           → métricas del pipeline de clientes
GET /api/admin/platform-metrics       → métricas globales (solo admin de plataforma)
```

Parámetros opcionales: `?period=7d|30d|90d` (default: 30d), `?vertical=reintegra_ai`.

---

## Retención de datos

| Plan | Historial visible |
|---|---|
| Free | 7 días |
| Starter | 90 días |
| Pro / Agency | 12 meses |

Los datos raw se retienen 12 meses en la tabla caliente. Datos más antiguos se agregan a tabla mensual y se eliminan del raw.

---

## Reglas clave

- `track()` es SIEMPRE fire-and-forget — nunca `await` en el flujo principal del request
- Los eventos son append-only — nunca UPDATE ni DELETE en `analytics_events`
- El dashboard admin de plataforma (`/api/admin/*`) requiere validación explícita de rol de plataforma, separada de los roles de organización de Clerk
- El evento `limit.generations_reached` es la señal más valiosa de intención de compra — trackear con máxima prioridad

---

## Referencias

- Ver `pyme-crm` skill para métricas de pipeline y revenue
- Ver `freemium-gates` skill para los eventos de límite
- Ver `multi-tenancy` skill para filtrar analytics por organización
