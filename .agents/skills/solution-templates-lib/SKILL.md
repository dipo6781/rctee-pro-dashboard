---
name: solution-templates-lib
description: Biblioteca de plantillas de consultoría de R-C-T-E-E Pro. Úsala al construir el catálogo de plantillas, al implementar el flujo de selección y uso, o al agregar plantillas nuevas a los 7 verticales. Las plantillas son el producto core — no son "templates de chatbot", son metodologías de consultoría empaquetadas.
---

# Solution Templates Library

Las plantillas son el corazón del producto. Cada plantilla = una metodología de consultoría senior empaquetada en un formulario + prompt R-C-T-E-E + generador de documentos profesionales.

---

## Catálogo actual (60+ plantillas en 7 verticales)

### 💰 Reintegra AI — Financiero / Cobranza
| Plantilla | Output | Precio sugerido |
|---|---|---|
| Diagnóstico Financiero Express | PDF 30 pág | $4,500 USD |
| Plan de Cobranza Completo | Word + templates | $6,000 USD |
| Análisis de Cartera Vencida | Excel + PDF | $3,500 USD |

### 🧘 Kineticorp Bienestar — RRHH
| Plantilla | Output | Precio sugerido |
|---|---|---|
| Programa Bienestar Laboral 12 Meses | PDF + Excel calendario | $15,000 USD |
| Diagnóstico de Clima Organizacional | PDF 20 pág | $5,000 USD |
| Plan de Reducción de Rotación | PDF + plan de acción | $8,000 USD |

*(y más en los otros 5 verticales)*

---

## Estructura en DB

```typescript
// Ver rcte-methodology skill para el schema completo de la tabla templates
// Campos clave adicionales para la biblioteca:

interface TemplateCard {
  id: uuid;
  vertical: Vertical;
  name: string;
  description: string;        // Qué problema resuelve (lenguaje de la pyme)
  value_proposition: string;  // "Recupera $80K de cartera vencida en 90 días"
  output_format: OutputFormat;
  output_pages_approx: number;
  price_usd: number;          // Precio sugerido al cliente final
  duration_days: number;      // Días típicos de implementación
  required_plan: PlanTier;
  tags: string[];             // ["cobranza", "financiero", "pymes", "retail"]
  preview_pdf_url: string;    // PDF de muestra (primeras 3 páginas)
  is_featured: boolean;
  is_active: boolean;
}
```

---

## MVP: Las 3 plantillas que deben funcionar perfectamente

Antes de lanzar cualquier otra cosa, estas 3 deben generar documentos impecables:

### Plantilla 1: Diagnóstico Financiero Express
```
Vertical: Reintegra AI
Output: PDF ~30 páginas con portada de marca, índice, análisis, semáforo de colores
Formulario: 7 campos (ver rcte-methodology skill)
Precio: $4,500 USD
Tiempo generación: ~90s
Estado: ✅ Ya existe en el código — empaquetar como entregable
```

### Plantilla 2: Plan de Cobranza Completo
```
Vertical: Reintegra AI
Output: Documento Word (.docx) + 5 templates de mensajes WhatsApp/email
Formulario: 6 campos (empresa, cartera por tramos, sector, tamaño)
Precio: $6,000 USD
Tiempo generación: ~60s
Estado: ✅ Ya existe — migrar al nuevo pipeline
```

### Plantilla 3: Programa Bienestar Laboral 12 Meses
```
Vertical: Kineticorp Bienestar
Output: PDF programa (~20 pág) + archivo Excel con calendario y presupuesto
Formulario: 8 campos (empresa, empleados, problema principal, presupuesto disponible)
Precio: $15,000 USD
Tiempo generación: ~75s
Estado: ✅ Ya existe — migrar al nuevo pipeline
```

---

## Flujo de uso de una plantilla (UX no-code)

```
1. Usuario abre panel → ve galería de plantillas organizadas por vertical
2. Filtra por vertical (ej: "Financiero") o por tipo de problema
3. Selecciona plantilla → ve: descripción, entregable, precio sugerido, muestra PDF
4. Clic "Usar plantilla" → formulario guiado paso a paso
5. Completa campos con datos de su cliente pyme (con ejemplos en cada campo)
6. Clic "Generar documento" → barra de progreso (30-90s)
7. Documento listo → descarga PDF/Word/Excel
8. (Opcional) Comparte directamente con el cliente desde la plataforma
```

---

## Filtros de la galería

```typescript
// GET /api/templates?vertical=reintegra_ai&plan_ok=true&search=cobranza
interface TemplateFilters {
  vertical?: Vertical;
  output_format?: OutputFormat;
  max_price?: number;
  search?: string;          // busca en name, description, tags
  plan_ok?: boolean;        // solo mostrar las accesibles por el plan del usuario
  is_featured?: boolean;
}
```

---

## Previsualización (preview PDF)

Cada plantilla tiene un PDF de muestra con las primeras 3 páginas del entregable real (datos de ejemplo). Esto es clave para la conversión — el usuario ve exactamente qué va a recibir.

```
Almacenamiento: Supabase Storage en bucket "template-previews"
Formato: PDF, máx 2MB
URL: public (sin auth — sirven para marketing también)
Generar manualmente para cada plantilla nueva
```

---

## Acceso por plan

| Plan | Plantillas disponibles |
|---|---|
| Free | 3 plantillas básicas (1 por vertical core), 2 usos/mes |
| Starter | Todas las plantillas de 1 vertical elegido, 10 usos/mes |
| Pro | Todos los verticales (60+), usos ilimitados |
| Agencia | Todo + white label, sin marca de la plataforma en los docs |

---

## Agregar una plantilla nueva (checklist)

- [ ] Definir los 5 componentes R-C-T-E-E (ver `rcte-methodology` skill)
- [ ] Crear los `form_fields` con placeholders claros y ejemplos reales
- [ ] Definir `required_sections` para validar el output
- [ ] Generar 5 documentos de prueba con datos reales y validar calidad
- [ ] Crear el PDF de preview (3 páginas, datos de ejemplo)
- [ ] Definir `price_usd` y `required_plan`
- [ ] Subir a Supabase y activar (`is_active: true`)
- [ ] Agregar a la galería con `value_proposition` clara

---

## Nomenclatura de archivos generados

```
{vertical}_{template_slug}_{organization_id}_{timestamp}.pdf
// Ejemplo: reintegra_ai_diagnostico_financiero_abc123_20240115.pdf

Almacenamiento: Supabase Storage, bucket "deliverables"
Acceso: firmado (URL con expiración de 7 días), nunca público permanente
```

---

## Reglas clave

- Los prompts R-C-T-E-E son IP de la plataforma — **nunca exponer en la UI ni en el API response**
- El PDF de preview sí es público — úsalo en marketing y landing page
- Guardar el `generated_content` (texto crudo de la IA) además del PDF — permite regenerar el formato sin re-llamar a la IA
- Las plantillas "premium" tienen `required_plan: "pro"` — no desbloquear por error
- El nombre del archivo incluye `organization_id` para garantizar aislamiento de datos en storage

---

## Referencias

- Ver `rcte-methodology` skill para el schema de prompts y la implementación del builder
- Ver `ai-agent-config` skill para el pipeline de generación
- Ver `freemium-gates` skill para los límites de acceso por plan
- Ver `platform-architecture` skill para el stack y los verticales
