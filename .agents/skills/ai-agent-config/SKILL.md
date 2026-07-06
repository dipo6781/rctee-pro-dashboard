---
name: ai-agent-config
description: Configuración del motor de IA en R-C-T-E-E Pro para generación de entregables de consultoría. Úsala al implementar la generación de documentos con Groq/Ollama, al construir el pipeline de prompts R-C-T-E-E, o al manejar la cola de generación de documentos.
---

# AI Agent Config — R-C-T-E-E Pro

El "agente" en esta plataforma NO es un chatbot. Es un **motor de generación de documentos profesionales** que toma un formulario completado por el usuario, construye un prompt R-C-T-E-E estructurado, llama a la IA, y produce el contenido de un entregable (PDF, Word, Excel).

---

## Proveedores de IA (ya configurados en el proyecto)

```typescript
// Proveedor primario: Groq (velocidad alta, ideal para docs largos)
// Proveedor fallback: Ollama (local, desarrollo y contingencia)

// Variables de entorno requeridas:
// GROQ_API_KEY — ya configurada
// OLLAMA_BASE_URL — http://localhost:11434 (dev)
```

### Cliente Groq

```typescript
import Groq from "groq-sdk";

const groq = new Groq({ apiKey: process.env.GROQ_API_KEY });

export async function generateWithGroq(
  systemPrompt: string,
  userContent: string,
  options?: { model?: string; maxTokens?: number; temperature?: number }
): Promise<string> {
  const response = await groq.chat.completions.create({
    model: options?.model ?? "llama-3.3-70b-versatile", // modelo recomendado para docs largos
    max_tokens: options?.maxTokens ?? 8000,
    temperature: options?.temperature ?? 0.3, // bajo = más consistente para documentos formales
    messages: [
      { role: "system", content: systemPrompt },
      { role: "user", content: userContent },
    ],
  });
  return response.choices[0].message.content ?? "";
}
```

### Cliente Ollama (fallback)

```typescript
export async function generateWithOllama(
  systemPrompt: string,
  userContent: string
): Promise<string> {
  const response = await fetch(`${process.env.OLLAMA_BASE_URL}/api/chat`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      model: "llama3.1:8b",
      stream: false,
      messages: [
        { role: "system", content: systemPrompt },
        { role: "user", content: userContent },
      ],
    }),
  });
  const data = await response.json();
  return data.message.content;
}
```

---

## Pipeline de generación de entregables

```
FormData (inputs del usuario)
  ↓
buildRCTEEPrompt(template, inputs)   ← ver rcte-methodology skill
  ↓
generateWithGroq(systemPrompt, userContent)
  ↓
validateOutput(rawContent)            ← verificar longitud y estructura mínima
  ↓
formatDocument(content, template.outputFormat)  ← PDF | Word | Excel
  ↓
uploadToStorage(file)                 ← Supabase Storage
  ↓
updateDeliverable(id, { file_url, status: "ready" })
```

---

## Cola de generación (jobs)

La generación de un documento toma 30-120 segundos. **Nunca bloquear el request HTTP.** Usar cola de jobs:

```typescript
// Usando BullMQ + Redis (o pg-boss si ya hay Supabase disponible)
// Alternativa simple sin Redis: tabla `generation_jobs` en Supabase + polling

// Tabla: generation_jobs
{
  id: uuid,
  deliverable_id: uuid,
  template_id: uuid,
  organization_id: uuid,
  inputs: jsonb,         // datos del formulario
  status: "queued" | "processing" | "completed" | "failed",
  attempts: integer,     // reintentos
  error: text | null,
  started_at: timestamp,
  completed_at: timestamp,
  created_at: timestamp,
}
```

### Flujo con polling (sin Redis — más simple para MVP)

```typescript
// 1. Frontend envía formulario → backend crea job → devuelve job_id inmediatamente
// 2. Frontend hace polling cada 3s: GET /api/jobs/:id/status
// 3. Cuando status = "completed", frontend descarga el documento

// Backend worker: cron cada 5s que toma jobs en "queued" y los procesa
```

---

## Modelos recomendados por tipo de entregable

| Entregable | Modelo Groq | Max tokens | Temperatura |
|---|---|---|---|
| Diagnóstico financiero (PDF 30 pág) | `llama-3.3-70b-versatile` | 8000 | 0.2 |
| Guión de cobranza (Word) | `llama-3.1-8b-instant` | 4000 | 0.4 |
| Plan de bienestar (PDF) | `llama-3.3-70b-versatile` | 6000 | 0.3 |
| Análisis de cartera (Excel) | `llama-3.1-8b-instant` | 3000 | 0.1 |
| Propuesta comercial | `llama-3.3-70b-versatile` | 5000 | 0.3 |

---

## Validación del output

Antes de formatear el documento, validar que el output sea usable:

```typescript
function validateOutput(content: string, template: Template): ValidationResult {
  const minLength = template.outputFormat === "pdf_30pages" ? 5000 : 1000;
  const hasRequiredSections = template.requiredSections.every(section =>
    content.toLowerCase().includes(section.toLowerCase())
  );

  if (content.length < minLength) return { valid: false, reason: "output_too_short" };
  if (!hasRequiredSections) return { valid: false, reason: "missing_sections" };
  return { valid: true };
}

// Si falla la validación: reintentar con temperature más alta (0.5) y prompt levemente modificado
// Máximo 3 reintentos antes de marcar como "failed" y notificar al usuario
```

---

## Generación de documentos

### PDF (PDFKit o Puppeteer)

```typescript
// Puppeteer: más fácil para documentos con formato complejo y gráficos
// PDFKit: más ligero, mejor para documentos estructurados simples

// Recomendado para R-C-T-E-E Pro: Puppeteer
// Flujo: contenido IA → plantilla HTML con estilos de marca → Puppeteer → PDF

import puppeteer from "puppeteer";

async function generatePDF(htmlContent: string, brand: BrandConfig): Promise<Buffer> {
  const browser = await puppeteer.launch({ headless: true, args: ["--no-sandbox"] });
  const page = await browser.newPage();
  await page.setContent(renderHTMLTemplate(htmlContent, brand));
  const pdf = await page.pdf({ format: "A4", printBackground: true });
  await browser.close();
  return pdf;
}
```

### Word (docxtemplater)

```typescript
// Usar plantilla .docx base por vertical + reemplazar variables con el contenido generado
import Docxtemplater from "docxtemplater";
import PizZip from "pizzip";
import fs from "fs";

async function generateWord(content: DocumentContent, templatePath: string): Promise<Buffer> {
  const template = fs.readFileSync(templatePath, "binary");
  const zip = new PizZip(template);
  const doc = new Docxtemplater(zip);
  doc.setData(content);
  doc.render();
  return doc.getZip().generate({ type: "nodebuffer" });
}
```

---

## Reglas clave

- **Temperatura baja (0.1-0.3)** para documentos formales — más consistente y profesional
- **Nunca exponer el prompt completo al usuario** — es propiedad intelectual de la plataforma
- Los jobs fallidos se reintentan automáticamente hasta 3 veces antes de notificar
- Guardar el `generated_content` (texto crudo) además del archivo final — permite regenerar sin llamar a la IA
- Para el MVP: polling simple sobre tabla de Supabase (no necesita Redis)

---

## Referencias

- Ver `rcte-methodology` skill para construcción de prompts
- Ver `solution-templates-lib` skill para estructura de plantillas
- Ver `platform-architecture` skill para el stack completo
