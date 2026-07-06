---
name: ai-agent-config
description: Configuración, creación y ejecución de agentes IA dentro de la plataforma. Úsala al construir el módulo de agentes, al agregar un nuevo tipo de agente, o al conectar un agente a un proyecto de usuario.
---

# AI Agent Config

Los agentes IA son **registros en base de datos**, no código hardcodeado. Cada agente tiene un tipo, configuración, modelo, y prompt del sistema. Se ejecutan bajo demanda con el contexto del proyecto al que pertenecen.

---

## Modelo de datos del agente

```typescript
// Campos core de la tabla `agents`
id: uuid
organization_id: uuid          // tenant scope
project_id: uuid               // proyecto al que pertenece
name: string                   // nombre visible al usuario
type: AgentType                // ver tipos abajo
model: string                  // "gpt-4o", "claude-3-5-sonnet", etc.
system_prompt: text            // instrucciones del agente
temperature: float             // 0.0 - 2.0, default 0.7
max_tokens: integer            // límite de respuesta
tools_enabled: jsonb           // array de tools habilitadas
knowledge_base_ids: uuid[]     // documentos de contexto
is_active: boolean
created_at, updated_at
```

## Tipos de agente (`AgentType`)

| Tipo | Caso de uso |
|---|---|
| `chatbot` | Atención al cliente, soporte, FAQ |
| `automator` | Ejecuta flujos de trabajo paso a paso |
| `collector` | Recopila datos del usuario (formularios conversacionales) |
| `analyzer` | Analiza documentos, datos, tendencias |
| `notifier` | Envía alertas y resúmenes proactivos |
| `sales` | Calificación de leads, seguimiento comercial |

---

## Integración con Replit AI (sin API key del usuario)

Usar siempre el proxy de Replit AI Integrations. Leer `.local/skills/ai-integrations-openai/SKILL.md` o `.local/skills/ai-integrations-anthropic/SKILL.md` para setup exacto.

```typescript
// Patrón de ejecución de agente
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: process.env.REPLIT_AI_PROXY_URL,
  apiKey: process.env.REPLIT_AI_PROXY_TOKEN,
});

async function runAgent(agent: Agent, userMessage: string, history: Message[]) {
  const response = await client.chat.completions.create({
    model: agent.model,
    temperature: agent.temperature,
    max_tokens: agent.max_tokens,
    messages: [
      { role: "system", content: agent.system_prompt },
      ...history,
      { role: "user", content: userMessage },
    ],
  });
  return response.choices[0].message.content;
}
```

---

## Reglas clave

- **Nunca ejecutar agentes sincrónicamente en el request handler** si el tiempo de respuesta puede superar 30s. Usar streaming (`stream: true`) o jobs en cola.
- Los agentes con `type: automator` deben registrar cada paso ejecutado en tabla `agent_runs` para auditoría.
- El `system_prompt` siempre debe incluir el contexto de la organización y el proyecto (inyectado en runtime, no hardcodeado).
- Los límites de mensajes/mes se verifican ANTES de ejecutar el agente — ver `freemium-gates` skill.
- Guardar el historial de conversación por `session_id`, no por usuario, para permitir múltiples sesiones simultáneas.

---

## Tabla `agent_runs` (auditoría)

```
id, agent_id, organization_id, session_id,
input_tokens, output_tokens, duration_ms,
status (success|error|timeout), error_message,
created_at
```

---

## Tools habilitables por agente

```typescript
type AgentTool =
  | "web_search"          // búsqueda web
  | "send_email"          // envío de emails
  | "create_calendar_event" // eventos
  | "query_crm"           // consultar CRM de la org
  | "webhook_call"        // llamar webhook externo
  | "generate_document"   // crear PDF/doc
```

Solo habilitar las tools que el plan del usuario permite — ver `freemium-gates`.

---

## Streaming de respuestas al frontend

```typescript
// En el route handler (Express 5)
res.setHeader("Content-Type", "text/event-stream");
res.setHeader("Cache-Control", "no-cache");

const stream = await client.chat.completions.create({
  model: agent.model,
  messages,
  stream: true,
});

for await (const chunk of stream) {
  const text = chunk.choices[0]?.delta?.content || "";
  res.write(`data: ${JSON.stringify({ text })}\n\n`);
}
res.write("data: [DONE]\n\n");
res.end();
```

---

## Referencias

- Ver `platform-architecture` skill para estructura de datos completa
- Ver `freemium-gates` skill para límites de uso por plan
- Leer `.local/skills/ai-integrations-openai/SKILL.md` antes de implementar
