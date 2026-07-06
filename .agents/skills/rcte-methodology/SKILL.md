---
name: rcte-methodology
description: Metodología propietaria R-C-T-E-E de R-C-T-E-E Pro para construir prompts de consultoría. Úsala al crear o editar plantillas, al implementar el motor de generación, o al agregar un nuevo vertical. Es la columna vertebral del producto — toda generación de documentos pasa por este framework.
---

# Metodología R-C-T-E-E

**R-C-T-E-E** = Rol · Contexto · Tarea · Especificaciones · Ejemplos

Es el framework propietario que transforma datos de una pyme en un prompt estructurado capaz de generar un entregable de nivel consultor senior. Esta metodología ES el moat competitivo de la plataforma.

---

## Los 5 componentes

### R — Rol
Define quién ES la IA en este contexto. Cuanto más específico, mejor el output.

```
❌ Malo:  "Eres un asistente financiero."
✅ Bueno: "Eres un consultor financiero senior con 15 años de experiencia en reestructuración 
          de cartera para empresas retail en Colombia, especializado en recuperación de cartera 
          vencida usando metodología de scoring crediticio y segmentación por tramos de mora."
```

### C — Contexto
Datos reales de la pyme inyectados desde el formulario del usuario.

```
"La empresa es: {company_name}
Industria: {industry}
Tamaño: {employee_count} empleados
Problema específico: {problem_description}
Datos clave: {key_metrics}
Situación actual: {current_situation}"
```

### T — Tarea
Qué debe producir exactamente la IA. Una sola tarea, muy específica.

```
❌ Malo:  "Analiza la situación financiera."
✅ Bueno: "Genera un diagnóstico completo de cartera vencida que incluya análisis de ratios 
          de liquidez, solvencia y rentabilidad, clasificados por semáforo de colores 
          (🔴 crítico / 🟡 atención / 🟢 saludable), con un plan de acción para 30 días."
```

### E — Especificaciones
Formato exacto del output: estructura, longitud, secciones obligatorias, estilo.

```
"El documento debe:
- Tener exactamente estas secciones: [lista de secciones]
- Extensión: 2,500-3,000 palabras
- Tono: formal, ejecutivo, directo (sin tecnicismos innecesarios)
- Formato: usar subtítulos en negrita, listas numeradas para recomendaciones
- Incluir al menos 3 métricas cuantificadas con números concretos
- Terminar con un 'Plan de Acción Inmediata' de 5 pasos con fechas"
```

### E — Ejemplos
Fragmentos del output esperado para que la IA calibre el nivel y estilo.

```
"Ejemplo del nivel de análisis esperado:
'La empresa presenta un ratio de liquidez corriente de 0.87, lo que indica que por cada 
peso de obligación corriente solo dispone de $0.87 para responder. Esto clasifica como 
CRÍTICO (🔴) según la metodología R-C-T-E-E. Acción recomendada: revisar política de 
cobro y renegociar plazos con proveedores en los próximos 15 días...'"
```

---

## Estructura de una plantilla en la DB

```typescript
// Tabla: templates
interface Template {
  id: uuid;
  vertical: Vertical;                    // "reintegra_ai" | "kineticorp" | etc.
  name: string;                          // "Diagnóstico Financiero Express"
  description: string;                   // Para mostrar al usuario
  output_format: OutputFormat;           // "pdf" | "word" | "excel"
  output_pages_approx: number;           // 30 para el diagnóstico financiero
  price_usd: number;                     // Precio sugerido al cliente final
  required_plan: PlanTier;              // Qué plan necesita para acceder

  // Componentes R-C-T-E-E (NUNCA exponer al usuario final)
  role_prompt: text;                     // El Rol fijo de la plantilla
  task_prompt: text;                     // La Tarea fija
  specifications_prompt: text;           // Las Especificaciones fijas
  examples_prompt: text;                 // Los Ejemplos fijos

  // Formulario de inputs (el Contexto lo completa el usuario)
  form_fields: FormField[];              // Campos que el usuario debe llenar
  required_sections: string[];           // Secciones que debe tener el output sí o sí

  is_active: boolean;
  created_at, updated_at;
}

type Vertical =
  | "reintegra_ai"          // 💰 Financiero/Cobranza
  | "kineticorp_bienestar"  // 🧘 RRHH/Bienestar
  | "diseno_interiores"     // 🎨 Interiorismo
  | "toondimension"         // 🎉 Eventos
  | "ecokineti"             // 🌱 Sostenibilidad
  | "reintegration"         // 🏠 Inmobiliario
  | "kinetic_corp_ia";      // ⚡ Tecnología

type OutputFormat = "pdf" | "word" | "excel" | "pdf_word_bundle";
```

---

## Campos del formulario (Context inputs)

```typescript
interface FormField {
  key: string;               // Nombre de la variable en el prompt
  label: string;             // Texto visible al usuario
  type: "text" | "number" | "textarea" | "select" | "date" | "currency";
  placeholder: string;       // Ejemplo orientador
  required: boolean;
  options?: string[];        // Para tipo "select"
  help_text?: string;        // Tooltip de ayuda
}
```

**Ejemplo para Diagnóstico Financiero:**
```typescript
[
  { key: "company_name", label: "Nombre de la empresa", type: "text", required: true },
  { key: "industry", label: "Industria / sector", type: "select", required: true,
    options: ["Retail", "Manufactura", "Servicios", "Tecnología", "Construcción", "Otro"] },
  { key: "employee_count", label: "Número de empleados", type: "number", required: true },
  { key: "annual_revenue", label: "Ingresos anuales aprox. (COP)", type: "currency", required: true },
  { key: "overdue_portfolio", label: "Cartera vencida actual (COP)", type: "currency", required: true },
  { key: "overdue_days_breakdown", label: "Distribución por tramos de mora",
    type: "textarea", placeholder: "Ej: 1-30 días: $50M, 31-60 días: $30M, 61-90 días: $20M",
    required: true },
  { key: "main_problem", label: "Describe el problema principal en tus palabras",
    type: "textarea", required: true },
]
```

---

## Constructor del prompt completo

```typescript
function buildRCTEEPrompt(template: Template, inputs: Record<string, string>): {
  systemPrompt: string;
  userContent: string;
} {
  // Construir el Contexto con los datos del formulario
  const contextBlock = template.form_fields
    .map(field => `${field.label}: ${inputs[field.key] ?? "No especificado"}`)
    .join("\n");

  const systemPrompt = `${template.role_prompt}

## ESPECIFICACIONES DEL ENTREGABLE
${template.specifications_prompt}

## EJEMPLOS DEL NIVEL ESPERADO
${template.examples_prompt}

IMPORTANTE: El documento debe ser completamente profesional, listo para presentar a un directivo. 
No incluyas frases como "como modelo de lenguaje" ni disculpas. Genera el documento directamente.`;

  const userContent = `## CONTEXTO DE LA EMPRESA
${contextBlock}

## TAREA
${template.task_prompt}`;

  return { systemPrompt, userContent };
}
```

---

## Las 3 plantillas MVP (implementar primero)

### 1. Diagnóstico Financiero Express — Reintegra AI
- **Output:** PDF ~30 páginas
- **Precio sugerido al cliente:** $4,500 USD
- **Secciones obligatorias:** Resumen ejecutivo, Análisis de ratios (liquidez/solvencia/rentabilidad), Semáforo de salud financiera, Top 3 problemas críticos, Plan de acción 30 días
- **Tiempo de generación:** ~90s con Groq llama-3.3-70b

### 2. Plan de Cobranza Completo — Reintegra AI
- **Output:** Word (~15 páginas) + templates WhatsApp/email
- **Precio sugerido al cliente:** $6,000 USD
- **Secciones obligatorias:** Segmentación de cartera, Guiones por tramo de mora (1-30, 31-60, 61-90 días), Templates de mensajes, Cronograma de gestión, KPIs de seguimiento
- **Tiempo de generación:** ~60s

### 3. Programa Bienestar Laboral 12 Meses — Kineticorp Bienestar
- **Output:** PDF (~20 páginas) + calendario Excel
- **Precio sugerido al cliente:** $15,000 USD
- **Secciones obligatorias:** Diagnóstico de clima laboral, Programa mensual de actividades, Presupuesto desglosado, Métricas de seguimiento, Plan de comunicación interna
- **Tiempo de generación:** ~75s

---

## Reglas clave

- Los prompts (`role_prompt`, `task_prompt`, etc.) **NUNCA se exponen al usuario** — son IP de la plataforma
- El Contexto es el único bloque dinámico — el resto es fijo por plantilla
- Temperatura ≤ 0.3 para documentos formales (más determinístico = más profesional)
- Si el output no contiene todas las `required_sections`, reintentar automáticamente
- El prompt completo para el diagnóstico financiero suele tener ~1,200 tokens de sistema + ~400 de usuario

---

## Referencias

- Ver `ai-agent-config` skill para la implementación técnica de la generación
- Ver `solution-templates-lib` skill para la gestión del catálogo de plantillas
- Ver `platform-architecture` skill para el contexto completo del producto
