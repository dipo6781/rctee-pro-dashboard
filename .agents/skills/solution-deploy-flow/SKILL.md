---
name: solution-deploy-flow
description: Flujo de entrega de entregables en R-C-T-E-E Pro. Úsala al construir el pipeline de generación y descarga de documentos, al implementar el envío de entregables al cliente pyme, o al gestionar el historial de documentos generados. NOTA: en esta plataforma "deploy" = entregar un PDF/Word/Excel profesional al cliente, no publicar una app.
---

# Solution Deploy Flow — R-C-T-E-E Pro

En R-C-T-E-E Pro, "publicar una solución" significa **entregar un documento profesional (PDF/Word/Excel) al cliente pyme**. No hay apps que desplegar — el entregable ES el producto.

---

## Flujo completo de entrega

```
1. Consultor selecciona plantilla y cliente
2. Completa formulario con datos de la pyme
3. Sistema valida datos y límites del plan (freemium-gates)
4. Se crea un `deliverable` en status "queued" → respuesta inmediata al usuario
5. Job worker toma el deliverable y:
   a. Construye prompt R-C-T-E-E (rcte-methodology)
   b. Llama a Groq API para generar contenido (~30-90s)
   c. Valida el output (secciones mínimas, longitud)
   d. Genera el archivo (PDF via Puppeteer / Word via docxtemplater / Excel via exceljs)
   e. Sube el archivo a Supabase Storage
   f. Actualiza deliverable a status "completed" con file_url
6. Frontend detecta el cambio (polling o realtime) → muestra botón de descarga
7. Consultor descarga o envía directamente al cliente
```

---

## Generación del archivo (por formato)

### PDF — Puppeteer + plantilla HTML de marca

```typescript
// Cada vertical tiene su plantilla HTML con colores y logo de marca
// El contenido de la IA se inyecta en los slots de la plantilla

async function generatePDF(
  content: GeneratedContent,
  template: Template,
  client: Client
): Promise<Buffer> {
  const html = renderDeliverableHTML({
    content,
    brand: VERTICAL_BRANDS[template.vertical],  // colores, logo, tipografía del vertical
    client,
    generatedAt: new Date(),
  });

  const browser = await puppeteer.launch({
    headless: true,
    args: ["--no-sandbox", "--disable-setuid-sandbox"],
  });
  const page = await browser.newPage();
  await page.setContent(html, { waitUntil: "networkidle0" });
  const pdf = await page.pdf({
    format: "A4",
    printBackground: true,
    margin: { top: "20mm", bottom: "20mm", left: "15mm", right: "15mm" },
  });
  await browser.close();
  return pdf;
}
```

### Word — docxtemplater con plantilla .docx base

```typescript
// Cada vertical tiene su archivo .docx base con estilos de marca
// Variables {{like_this}} en el .docx son reemplazadas con el contenido generado

async function generateWord(
  content: GeneratedContent,
  templateDocxPath: string
): Promise<Buffer> {
  const templateBuffer = await fs.readFile(templateDocxPath);
  const zip = new PizZip(templateBuffer);
  const doc = new Docxtemplater(zip, { paragraphLoop: true });

  doc.setData({
    company_name: content.client.companyName,
    date: formatDate(new Date()),
    ...content.sections,  // secciones mapeadas a variables del template .docx
  });

  doc.render();
  return doc.getZip().generate({ type: "nodebuffer" });
}
```

### Excel — exceljs con estructura por plantilla

```typescript
async function generateExcel(
  content: GeneratedContent,
  template: Template
): Promise<Buffer> {
  const workbook = new ExcelJS.Workbook();
  // Cada plantilla Excel tiene su propia función de construcción
  const builder = EXCEL_BUILDERS[template.slug];
  await builder(workbook, content);

  const buffer = await workbook.xlsx.writeBuffer();
  return Buffer.from(buffer);
}
```

---

## Almacenamiento en Supabase Storage

```typescript
// Bucket: "deliverables" — privado (no acceso público)
// Path: {organization_id}/{client_id}/{deliverable_id}.{ext}

async function uploadDeliverable(
  buffer: Buffer,
  orgId: string,
  clientId: string,
  deliverableId: string,
  format: "pdf" | "docx" | "xlsx"
): Promise<string> {
  const path = `${orgId}/${clientId}/${deliverableId}.${format}`;

  const { error } = await supabase.storage
    .from("deliverables")
    .upload(path, buffer, {
      contentType: FORMAT_MIME_TYPES[format],
      upsert: false,
    });

  if (error) throw error;
  return path; // guardar en DB, no la URL directa
}

// Generar URL firmada (válida 7 días para descargas del consultor)
async function getDownloadUrl(storagePath: string): Promise<string> {
  const { data } = await supabase.storage
    .from("deliverables")
    .createSignedUrl(storagePath, 60 * 60 * 24 * 7);
  return data!.signedUrl;
}

// URL de más larga duración para compartir con el cliente (30 días)
async function getClientShareUrl(storagePath: string): Promise<string> {
  const { data } = await supabase.storage
    .from("deliverables")
    .createSignedUrl(storagePath, 60 * 60 * 24 * 30);
  return data!.signedUrl;
}
```

---

## Worker de generación (polling sobre Supabase)

Para el MVP sin Redis, usar polling sobre la tabla `generation_jobs`:

```typescript
// Cron cada 10 segundos — toma el siguiente job en cola y lo procesa
async function processNextJob(): Promise<void> {
  const job = await db.query.generationJobs.findFirst({
    where: eq(generationJobs.status, "queued"),
    orderBy: asc(generationJobs.createdAt),
  });

  if (!job) return;

  // Marcar como "processing" (evita doble procesamiento)
  await db.update(generationJobs)
    .set({ status: "processing", startedAt: new Date() })
    .where(and(
      eq(generationJobs.id, job.id),
      eq(generationJobs.status, "queued") // condition race-safe
    ));

  try {
    const { systemPrompt, userContent } = buildRCTEEPrompt(job.template, job.inputs);
    const content = await generateWithGroq(systemPrompt, userContent);
    const validated = validateOutput(content, job.template);

    if (!validated.valid) throw new Error(`Invalid output: ${validated.reason}`);

    const buffer = await generateDocument(content, job.template, job.client);
    const storagePath = await uploadDeliverable(buffer, job.organizationId, job.clientId, job.deliverableId, job.outputFormat);

    await db.update(deliverables).set({
      fileUrl: storagePath,
      generatedContent: content,
      generationStatus: "completed",
    }).where(eq(deliverables.id, job.deliverableId));

    await db.update(generationJobs).set({
      status: "completed",
      completedAt: new Date(),
    }).where(eq(generationJobs.id, job.id));

    await incrementGenerationCount(job.organizationId);

  } catch (err) {
    const attempts = job.attempts + 1;
    await db.update(generationJobs).set({
      status: attempts >= 3 ? "failed" : "queued",
      attempts,
      error: String(err),
    }).where(eq(generationJobs.id, job.id));
  }
}
```

---

## Polling del frontend (sin WebSockets para MVP)

```typescript
// El frontend hace polling cada 3 segundos hasta que el status cambia
async function pollDeliverableStatus(deliverableId: string): Promise<Deliverable> {
  while (true) {
    const res = await fetch(`/api/deliverables/${deliverableId}/status`);
    const { status, fileUrl } = await res.json();

    if (status === "completed" || status === "failed") {
      return { status, fileUrl };
    }

    await new Promise(resolve => setTimeout(resolve, 3000));
  }
}
```

---

## Identidad visual por vertical (branding en los documentos)

```typescript
const VERTICAL_BRANDS = {
  reintegra_ai: {
    primaryColor: "#1B4F72",     // azul marino financiero
    accentColor: "#F39C12",      // dorado
    logo: "/assets/reintegra-logo.png",
    footerText: "Reintegra AI — Consultoría Financiera Especializada",
  },
  kineticorp_bienestar: {
    primaryColor: "#196F3D",     // verde bienestar
    accentColor: "#A9DFBF",
    logo: "/assets/kineticorp-logo.png",
    footerText: "Kineticorp Bienestar — Soluciones de RRHH",
  },
  // ... un objeto por cada vertical
};
```

---

## Endpoints

```
POST /api/deliverables                  → crear job de generación (respuesta inmediata)
GET  /api/deliverables/:id/status       → estado del job (para polling)
GET  /api/deliverables/:id/download     → generar URL firmada de descarga
POST /api/deliverables/:id/share        → generar URL de 30 días para el cliente
GET  /api/projects/:id/deliverables     → todos los entregables de un proyecto
```

---

## Reglas clave

- La generación es **siempre asíncrona** — nunca bloquear el HTTP request
- Guardar `generated_content` (texto crudo) siempre — permite regenerar el formato sin re-llamar a Groq
- Las URLs de storage son **firmadas con expiración**, nunca públicas permanentes
- Máximo 3 reintentos por job antes de marcar como "failed" y notificar al usuario
- El white label (sin logo/marca del vertical) solo está disponible en plan Agency

---

## Referencias

- Ver `rcte-methodology` skill para la construcción del prompt
- Ver `ai-agent-config` skill para la llamada a Groq y validación del output
- Ver `pyme-crm` skill para la relación proyecto → entregable → cliente
- Ver `freemium-gates` skill para verificar límites antes de crear el job
