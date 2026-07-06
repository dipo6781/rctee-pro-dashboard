---
name: platform-architecture
description: Arquitectura de R-C-T-E-E Pro — plataforma de consultoría empresarial aumentada por IA. Úsala al inicio de cualquier módulo, al tomar decisiones de stack, o al diseñar la estructura de datos. IMPORTANTE: esta plataforma NO es un constructor de chatbots; entrega diagnósticos y documentos profesionales a pymes.
---

# Platform Architecture — R-C-T-E-E Pro

**Qué es:** Fábrica de consultoría empresarial empaquetada en software. Convierte conocimiento experto en prompts estructurados (metodología R-C-T-E-E) que generan entregables de nivel consultor senior (PDF, Word, Excel) para pymes de 50-500 empleados en Colombia.

**NO es:** constructor de chatbots, plataforma de automatización, ni SaaS de atención al cliente.

**Compite con:** consultores tradicionales (McKinsey, Deloitte, independientes), no con Typebot/Botpress/n8n.

---

## Los 7 verticales (empresas/marcas de la plataforma)

| Marca | Vertical | Solución core |
|---|---|---|
| 💰 Reintegra AI | Financiero / Cobranza | Diagnóstico de cartera, guiones de cobranza |
| 🧘 Kineticorp Bienestar | RRHH / Bienestar laboral | Programas de bienestar, reducción de rotación |
| 🎨 Diseño Interiores | Interiorismo | Propuestas de diseño y presupuestos |
| 🎉 Toondimension | Eventos | Planificación y gestión de eventos |
| 🌱 Ecokineti | Sostenibilidad | Diagnóstico ESG, planes de sostenibilidad |
| 🏠 Reintegration | Inmobiliario | Valoración, estrategia de portafolio |
| ⚡ Kinetic Corp IA | Tecnología | Automatización y transformación digital |

Cada vertical tiene ~3 plantillas core (60+ en total).

---

## Metodología propietaria: R-C-T-E-E

Cada plantilla es un prompt estructurado con 5 componentes:
- **R**ol — qué experto es la IA en este contexto
- **C**ontexto — datos específicos de la pyme que usa la plantilla
- **T**area — qué debe producir exactamente
- **E**specificaciones — formato, extensión, estructura del entregable
- **E**jemplos — ejemplos del output esperado

Ver skill `rcte-methodology` para implementación detallada.

---

## Stack técnico

### Lo que YA existe (no recrear)
- **Supabase** — base de datos PostgreSQL + auth + storage
- **GitHub Pages** — frontend actual desplegado
- **Groq API** — LLM rápido (primario para generación)
- **Ollama** — LLM local (fallback / desarrollo)
- **60+ plantillas** en 7 verticales

### Stack para el panel de control central (nuevo)
- **Frontend:** React + Vite (TypeScript) — panel no-code, 100% visual
- **UI:** Tailwind CSS + shadcn/ui — debe sentirse como herramienta profesional, no startup tech
- **Backend:** Node.js + Express 5 (TypeScript) — API REST
- **DB:** Supabase (ya existente) vía Drizzle ORM
- **Auth:** Clerk (Replit-managed) o Supabase Auth (ya configurado — verificar cuál usa)
- **Generación de documentos:** PDFKit o Puppeteer (PDF), docxtemplater (Word), exceljs (Excel)
- **Cola de jobs:** Bull/BullMQ + Redis para generación de documentos (puede tardar 30-120s)
- **Pagos Colombia:** PayU Latam + PayPal (NO Stripe como prioritario — mercado colombiano)
- **Pagos internacional:** Stripe (secundario)

---

## Estructura de datos core

```
Organization (consultor / agencia que usa la plataforma)
  ├── Subscription (plan actual)
  ├── Templates (acceso a plantillas por vertical y plan)
  ├── Projects (engagement con un cliente pyme)
  │     ├── Client (la pyme cliente)
  │     ├── Deliverables (documentos generados)
  │     │     ├── template_id → plantilla usada
  │     │     ├── inputs (respuestas del formulario)
  │     │     ├── generated_content (texto generado por IA)
  │     │     └── file_url (PDF/Word/Excel final)
  │     └── Payments (cobros al cliente final)
  └── Analytics (métricas de uso y negocio)
```

---

## Flujo core de la plataforma

```
1. Usuario (consultor/pyme) selecciona plantilla del vertical correcto
2. Completa formulario guiado (datos de la empresa, problema, contexto)
3. Plataforma construye prompt R-C-T-E-E con los datos ingresados
4. IA (Groq) genera el contenido del entregable
5. Plataforma formatea y genera el documento (PDF/Word/Excel)
6. Usuario descarga o envía al cliente final
7. (Opcional) Usuario cobra al cliente final desde la plataforma
```

---

## Modelo de monetización (Estrategia Escalonada)

| Nivel | Qué es | Precio |
|---|---|---|
| **Freemium** | Acceso a 3 plantillas básicas, 2 usos/mes | Gratis |
| **Starter** | Acceso a 1 vertical completo, 10 usos/mes | $97/mes |
| **Pro** | Todos los verticales, usos ilimitados | $297/mes |
| **Por proyecto** | Entregable terminado entregado por el equipo | $4,500-$50,000 |
| **Agencia** | Reventa — white label para agencias | $497/mes |

---

## Principios de diseño

1. **No-code absoluto** — ningún usuario toca código. Todo es formularios, botones y plantillas.
2. **Entregable > conversación** — el output no es un chat, es un documento profesional descargable.
3. **Vertical-first** — la experiencia empieza eligiendo la industria, no la herramienta.
4. **Colombia-first** — idioma español, precios en COP/USD, pagos con PayU, WhatsApp como canal primario.
5. **Tiempo de entrega < 15 minutos** — desde completar el formulario hasta tener el PDF descargable.

---

## MVP (primera versión lanzable)

Ver contexto de MVP en `solution-templates-lib` skill. En resumen:
- 3 plantillas core funcionando perfectamente
- Landing page de ventas (1 página)
- Proceso de pago simple (transferencia + PayPal)
- Entrega del documento en PDF

**No incluir en MVP:** multi-tenancy complejo, todos los verticales, sistema de subscripciones completo.

---

## Referencias

- Ver `rcte-methodology` skill para la metodología de prompts
- Ver `solution-templates-lib` skill para estructura de plantillas y MVP
- Ver `platform-monetization` skill para pricing y pagos
- Ver `pyme-crm` skill para gestión de clientes
