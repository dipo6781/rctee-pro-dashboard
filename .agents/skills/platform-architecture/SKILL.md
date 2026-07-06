---
name: platform-architecture
description: Arquitectura recomendada para la plataforma AI multi-tenant con panel de control central para construcción de soluciones para pymes. Úsala al inicio de cualquier módulo nuevo, al tomar decisiones de stack, o al diseñar la estructura de datos multi-tenant.
---

# Platform Architecture

Plataforma AI de control central para construir soluciones (apps, chatbots, automatizaciones, cobranza, eventos) orientadas a pymes. Modelo freemium, usuarios mixtos (solopreneurs, agencias, empresas). Fase: desde cero.

---

## Stack Recomendado

### Frontend
- **React + Vite** (TypeScript) — panel de control central
- **Tailwind CSS + shadcn/ui** — componentes UI consistentes
- **TanStack Query** — estado del servidor
- **React Router v6** — navegación SPA

### Backend
- **Node.js + Express 5** (TypeScript) — API REST
- **OpenAPI + Orval** — contrato API generado, hooks React autogenerados
- **Zod** — validación en servidor y cliente

### Base de datos
- **PostgreSQL + Drizzle ORM** — datos relacionales (proyectos, usuarios, orgs, plantillas)
- **Multi-tenancy por `organization_id`** en cada tabla — aislamiento de datos por organización

### IA
- **Replit AI Integrations** (OpenAI/Anthropic vía proxy) — no requiere API key del usuario
- Agentes configurados como registros en DB, ejecutados en runtime

### Autenticación
- **Clerk** (Replit-managed) — auth, organizaciones, roles

### Pagos / Freemium
- **Stripe** — suscripciones, upgrades, gestión de plan

### Integraciones externas
- Webhooks salientes nativos
- Conectores para n8n/Zapier vía API keys generadas por la plataforma

### Deploy de soluciones de usuario
- Soluciones publicadas como sub-rutas o subdominios del hosting de la plataforma

---

## Principios de diseño

1. **Multi-tenant desde el día 1** — cada registro lleva `organization_id`, nunca datos globales sin scope
2. **Contrato OpenAPI primero** — toda feature nueva empieza con un endpoint en `lib/api-spec/openapi.yaml`
3. **Freemium por límites de recursos** — no features bloqueadas, sino cuotas (ej: max 3 proyectos, max 1000 mensajes/mes)
4. **Plantillas como ciudadanos de primera clase** — cada solución construida puede exportarse como plantilla
5. **Agentes IA como configuración** — los agentes son registros en DB, no código hardcodeado

---

## Estructura de datos core

```
Organization (tenant)
  ├── Users (miembros con roles)
  ├── Projects (soluciones en construcción)
  │     ├── Agents (agentes IA configurados)
  │     ├── Integrations (webhooks, APIs externas)
  │     └── Deployments (versiones publicadas)
  ├── Templates (plantillas reutilizables)
  ├── Clients (CRM — clientes finales de la org)
  ├── Subscription (plan actual, límites)
  └── Analytics Events (métricas de uso)
```

---

## Tabla de límites Freemium

| Recurso | Free | Starter | Pro |
|---|---|---|---|
| Proyectos activos | 3 | 10 | Ilimitado |
| Agentes IA | 1 por proyecto | 5 por proyecto | Ilimitado |
| Mensajes IA / mes | 500 | 5.000 | 50.000 |
| Miembros del equipo | 1 | 3 | Ilimitado |
| Integraciones externas | 0 | 3 | Ilimitado |
| Clientes en CRM | 50 | 500 | Ilimitado |

---

## Reglas de arquitectura

- Nunca exponer `organization_id` en URLs públicas — usar slugs o UUIDs opacos
- Todas las queries en la DB deben filtrar por `organization_id` del usuario autenticado
- Los agentes IA nunca se ejecutan sincrónicamente en el request handler — usar cola de jobs
- Las plantillas tienen un flag `is_public` para el marketplace futuro
- Los eventos de analytics se guardan en tabla append-only, nunca se actualizan

---

## Referencias

- Ver `pnpm-workspace` skill para estructura de monorepo
- Ver `freemium-gates` skill para implementar límites por plan
- Ver `multi-tenancy` skill para patrones de aislamiento de datos
- Ver `ai-agent-config` skill para configuración de agentes
