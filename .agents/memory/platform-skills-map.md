---
name: AI Platform Skills Map
description: Mapa de skills creados para R-C-T-E-E Pro — plataforma de consultoría empresarial aumentada por IA para pymes colombianas. Modelo freemium, mercado Colombia primario.
---

# R-C-T-E-E Pro — Skills Map

## Qué es el producto (CORREGIDO)

**R-C-T-E-E Pro NO es un constructor de chatbots.**

Es una **fábrica de consultoría empresarial empaquetada en software**. Convierte metodología experta en prompts estructurados (framework R-C-T-E-E) que generan entregables profesionales (PDF/Word/Excel) de nivel consultor senior para pymes colombianas de 50–500 empleados.

**Compite con:** consultores tradicionales (McKinsey, Deloitte, independientes).  
**NO compite con:** Typebot, Botpress, Tidio, n8n.  
**Mercado primario:** Colombia.  
**Canal de comunicación dominante:** WhatsApp Business.  
**Pago primario Colombia:** PayU Latam (no Stripe).

## Stack tecnológico

- **Ya existe:** Supabase (DB + auth + storage), GitHub Pages, Groq API (LLM), Ollama (fallback), 60+ plantillas en 7 verticales
- **Nuevo panel:** React + Vite, Node.js + Express 5, Drizzle ORM sobre Supabase, Clerk auth, PayU + PayPal
- **Generación docs:** Puppeteer (PDF), docxtemplater (Word), exceljs (Excel)

## Los 7 verticales

| Marca | Vertical |
|---|---|
| 💰 Reintegra AI | Financiero / Cobranza |
| 🧘 Kineticorp Bienestar | RRHH / Bienestar laboral |
| 🎨 Diseño Interiores | Interiorismo |
| 🎉 Toondimension | Eventos |
| 🌱 Ecokineti | Sostenibilidad |
| 🏠 Reintegration | Inmobiliario |
| ⚡ Kinetic Corp IA | Tecnología |

## MVP (3 plantillas core)

1. Diagnóstico Financiero Express → PDF 30 pág → $4,500 USD
2. Plan de Cobranza Completo → Word + templates WhatsApp → $6,000 USD
3. Programa Bienestar Laboral 12 Meses → PDF + Excel → $15,000 USD

## Skills creados (todos en `.agents/skills/`)

| Skill | Cuándo usarlo |
|---|---|
| `platform-architecture` | Inicio de cualquier módulo; stack; verticales; MVP scope |
| `rcte-methodology` | Construir o editar prompts; agregar plantillas; motor de generación |
| `ai-agent-config` | Pipeline de generación Groq/Ollama; jobs asíncronos; PDF/Word/Excel |
| `freemium-gates` | Límites por plan; verificar acceso a plantillas; mensajes de upgrade |
| `multi-tenancy` | Crear tablas; queries; Clerk org_id como tenant key (text, no uuid) |
| `platform-monetization` | PayU Colombia; PayPal; planes; estrategia escalonada |
| `solution-templates-lib` | Catálogo de plantillas; MVP; flujo de uso; acceso por plan |
| `platform-analytics` | Métricas de uso; embudo de conversión; dashboard del consultor |
| `integration-connectors` | WhatsApp Business API; SendGrid; Google Drive; credenciales encriptadas |
| `solution-deploy-flow` | Generación asíncrona de documentos; Supabase Storage; polling frontend |
| `pyme-crm` | Clientes pyme; pipeline de ventas; proyectos; compartir entregables |
| `team-collaboration` | Roles Clerk; invitaciones; permisos por rol; límites de equipo por plan |

## Orden de construcción para bootstrapping

1. `platform-architecture` + `multi-tenancy` → base sólida con Supabase existente
2. `rcte-methodology` + `ai-agent-config` → motor de generación funcional
3. `solution-templates-lib` + `solution-deploy-flow` → MVP con 3 plantillas
4. `pyme-crm` → gestión de clientes y pipeline
5. `platform-monetization` + `freemium-gates` → cobrar (empezar con pago manual)
6. `integration-connectors` → WhatsApp para envío de entregables
7. `platform-analytics` + `team-collaboration` → escalar

**Why:** El MVP es: formulario → documento PDF → descarga. Todo lo demás es accesorio.

## Gotchas importantes

- `organization_id` es un Clerk Org ID (string `"org_2abc..."`) — NO uuid de Postgres. Usar `text` en Drizzle.
- Groq ya está integrado — reusar cliente existente, no crear uno nuevo.
- Supabase Storage existe — usar buckets `deliverables` y `template-previews`.
- La generación de documentos tarda 30-120s — siempre asíncrona con polling, nunca bloquear el HTTP request.
- Los prompts R-C-T-E-E son IP de la plataforma — nunca exponer al usuario en la UI.
- PayU es el proveedor de pago primario para Colombia, no Stripe.
- Para el MVP no integrar payment gateway — proceso manual (transferencia + comprobante) hasta validar demanda.
