---
name: AI Platform Skills Map
description: Mapa de skills creados para la plataforma AI multi-tenant con panel central para construcción de soluciones para pymes, modelo freemium bootstrapping.
---

# AI Platform Skills Map

## Contexto del proyecto
Plataforma AI con panel de control central para construir soluciones (apps, chatbots, automatizaciones, cobranza, eventos) orientadas a pymes. Usuarios mixtos. Modelo freemium. Stack sugerido: React+Vite / Node.js+Express / PostgreSQL+Drizzle / Clerk / Stripe.

## Skills creados (todos en `.agents/skills/`)

| Skill | Cuándo usarlo |
|---|---|
| `platform-architecture` | Inicio de cualquier módulo; decisiones de stack; estructura de datos |
| `ai-agent-config` | Módulo de agentes; agregar tipos de agente; streaming |
| `freemium-gates` | Cualquier feature con cuotas; upgrades; límites por plan |
| `multi-tenancy` | Crear tablas; escribir queries; middleware de tenant |
| `platform-monetization` | Billing; Stripe; webhooks de pago; checkout |
| `solution-templates-lib` | Biblioteca de plantillas; clonar; exportar proyectos |
| `platform-analytics` | Dashboard de métricas; instrumentar eventos; reportes |
| `integration-connectors` | n8n/Zapier/WhatsApp; webhooks; credenciales encriptadas |
| `solution-deploy-flow` | Publicar soluciones; URLs públicas; widget embebible |
| `pyme-crm` | CRM de clientes finales; cobranza; campos personalizados |
| `team-collaboration` | Roles; invitaciones Clerk; permisos; audit log |

## Orden de construcción sugerido (bootstrapping)
1. `platform-architecture` + `multi-tenancy` → base de datos y auth
2. `freemium-gates` + `platform-monetization` → cobrar desde el día 1
3. `ai-agent-config` + `solution-templates-lib` → valor core
4. `pyme-crm` + `integration-connectors` → retención
5. `solution-deploy-flow` + `platform-analytics` → escalar

**Why:** El orden maximiza tiempo-a-primer-ingreso en modelo bootstrapping: cobrar temprano, entregar valor core, luego crecer.
