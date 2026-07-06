---
name: platform-analytics
description: Sistema de analytics y métricas de negocio para la plataforma AI. Úsala al construir el dashboard de métricas, al instrumentar eventos de uso, o al implementar reportes de actividad por organización.
---

# Platform Analytics

Dos capas de analytics: **métricas de plataforma** (qué hacen los usuarios dentro de la plataforma) y **métricas de solución** (qué hacen los clientes finales dentro de las soluciones construidas por los usuarios).

---

## Eventos a instrumentar (tabla `analytics_events`)

```typescript
// Tabla: analytics_events (append-only, nunca UPDATE)
{
  id: uuid,
  organization_id: uuid,
  project_id: uuid | null,
  agent_id: uuid | null,
  event_type: AnalyticsEventType,
  properties: jsonb,           // datos adicionales del evento
  user_id: uuid | null,        // usuario de la plataforma (no cliente final)
  session_id: string | null,
  ip_country: string | null,   // país (no IP — privacy)
  created_at: timestamp,
}
```

---

## Tipos de evento (`AnalyticsEventType`)

```typescript
// Eventos de plataforma
type AnalyticsEventType =
  // Proyectos
  | "project.created"
  | "project.deployed"
  | "project.archived"
  // Agentes IA
  | "agent.created"
  | "agent.message_sent"
  | "agent.error"
  // Monetización
  | "subscription.upgraded"
  | "subscription.downgraded"
  | "subscription.canceled"
  // Templates
  | "template.cloned"
  | "template.published"
  // CRM
  | "client.created"
  | "client.contacted"
  // Integraciones
  | "integration.connected"
  | "integration.webhook_received"
  // Límites
  | "limit.reached"            // qué límite se alcanzó (para medir fricción)
```

---

## Helper para registrar eventos

```typescript
// src/lib/analytics.ts
export async function track(
  event: AnalyticsEventType,
  orgId: string,
  properties: Record<string, unknown> = {},
  ctx?: { projectId?: string; agentId?: string; userId?: string }
) {
  // Fire-and-forget — no bloquear el request
  db.insert(analyticsEvents).values({
    organizationId: orgId,
    projectId: ctx?.projectId ?? null,
    agentId: ctx?.agentId ?? null,
    eventType: event,
    properties,
    userId: ctx?.userId ?? null,
  }).catch((err) => logger.error({ err }, "Failed to track event"));
}

// Uso en un route handler:
track("agent.message_sent", req.tenantId, { model: agent.model }, { agentId: agent.id });
```

---

## Métricas del dashboard (queries SQL recomendadas)

### Resumen de la organización (últimos 30 días)
```sql
-- Mensajes IA enviados
SELECT COUNT(*) FROM analytics_events
WHERE organization_id = $1
  AND event_type = 'agent.message_sent'
  AND created_at >= NOW() - INTERVAL '30 days';

-- Proyectos activos
SELECT COUNT(*) FROM projects
WHERE organization_id = $1 AND status = 'active';

-- Clientes en CRM
SELECT COUNT(*) FROM clients
WHERE organization_id = $1;
```

### Serie temporal de actividad (para gráficos)
```sql
SELECT
  DATE_TRUNC('day', created_at) AS day,
  event_type,
  COUNT(*) AS count
FROM analytics_events
WHERE organization_id = $1
  AND created_at >= NOW() - INTERVAL '30 days'
GROUP BY 1, 2
ORDER BY 1;
```

---

## Endpoints del dashboard

```
GET /api/analytics/summary          → métricas clave del período
GET /api/analytics/timeseries       → serie temporal por evento
GET /api/analytics/agents           → uso por agente
GET /api/analytics/projects         → actividad por proyecto
GET /api/analytics/limits           → cuántas veces se alcanzaron límites (conversión)
```

Todos requieren `?from=YYYY-MM-DD&to=YYYY-MM-DD` como parámetros opcionales (default: últimos 30 días).

---

## Métricas de negocio clave (para el dueño de la plataforma)

Para el panel de administración interno (no accesible por usuarios normales):

| Métrica | Query |
|---|---|
| MRR | SUM de suscripciones activas |
| Churn mensual | Cancelaciones / total activos inicio del mes |
| Activación | % orgs con ≥1 proyecto desplegado en primeros 7 días |
| Conversión free→paid | Upgrades / total orgs free |
| Feature más bloqueada | event_type = 'limit.reached' GROUP BY properties->>'resource' |

---

## Retención de datos

- Eventos de analytics: retener 12 meses en tabla caliente
- Datos más antiguos: agregar a tabla mensual y eliminar raw (reducir costos de DB)
- Plan Free: solo 30 días de historial visible en el dashboard
- Plan Starter/Pro: 12 meses de historial visible

---

## Reglas clave

- Los eventos son **append-only** — nunca UPDATE ni DELETE en `analytics_events`
- `track()` es fire-and-forget — nunca `await` en el flujo principal
- No guardar IPs completas — solo país (`ip_country`) para privacidad
- Separar analytics de la plataforma de analytics de las soluciones desplegadas (tablas distintas)

---

## Referencias

- Ver `platform-architecture` skill para estructura de datos completa
- Ver `multi-tenancy` skill para filtrar correctamente por organización
- Ver `freemium-gates` skill para los límites de historial por plan
