---
name: solution-deploy-flow
description: Flujo completo de deploy y publicación de soluciones construidas por usuarios en la plataforma. Úsala al construir el módulo de deploy, al implementar la publicación de proyectos, o al gestionar versiones y dominios de soluciones.
---

# Solution Deploy Flow

Cada proyecto construido en la plataforma puede ser "desplegado" para que los clientes finales del usuario accedan a él. El deploy genera una URL pública con la interfaz de la solución (chatbot, formulario, app ligera, etc.).

---

## Modelo de datos

```typescript
// Tabla: deployments
{
  id: uuid,
  organization_id: uuid,
  project_id: uuid,
  version: integer,                  // autoincremental por proyecto
  slug: string,                      // URL única de la solución
  custom_domain: string | null,      // dominio personalizado (plan Pro)
  status: DeploymentStatus,
  snapshot_hash: string,             // hash del snapshot para detectar cambios
  config: jsonb,                     // configuración de la solución publicada
  published_at: timestamp | null,
  created_by: uuid,
  created_at, updated_at
}

type DeploymentStatus =
  | "draft"        // en construcción, no accesible públicamente
  | "building"     // proceso de deploy en curso
  | "live"         // activo y accesible
  | "paused"       // temporalmente inaccesible
  | "archived"     // versión anterior, reemplazada
```

---

## URL de acceso público

Las soluciones desplegadas son accesibles en:
- **Default:** `https://<plataforma>.replit.app/s/<slug>`
- **Dominio personalizado (Pro):** `https://<custom-domain>` vía CNAME

El slug se genera automáticamente desde el nombre del proyecto (kebab-case + sufijo aleatorio) pero el usuario puede personalizarlo.

---

## Flujo de deploy

```
1. Usuario clic "Publicar"
2. Sistema verifica:
   - ¿Tiene cupo de deployments según su plan? (freemium-gates)
   - ¿Tiene al menos un agente activo en el proyecto?
3. Crear registro en `deployments` con status "building"
4. Generar snapshot del proyecto (igual que templates — sin credenciales)
5. Configurar la ruta pública `/s/<slug>`
6. Actualizar deployment status a "live"
7. Devolver URL pública al usuario
```

---

## Endpoints

```
POST /api/projects/:id/deploy           → iniciar deploy del proyecto
GET  /api/projects/:id/deployments      → historial de deployments
PUT  /api/deployments/:id/pause         → pausar deployment activo
PUT  /api/deployments/:id/resume        → reanudar deployment pausado
DELETE /api/deployments/:id             → archivar deployment
POST /api/deployments/:id/custom-domain → configurar dominio personalizado (Pro)

GET  /s/:slug                           → interfaz pública de la solución (SIN auth)
POST /s/:slug/chat                      → endpoint del chatbot público (SIN auth interna)
```

---

## Interfaz pública de la solución (`/s/:slug`)

La interfaz pública es una página React ligera (no el panel completo) que carga la configuración del deployment y renderiza el tipo de solución correspondiente:

```typescript
// Tipos de interfaz pública según el tipo de proyecto
type PublicInterface =
  | "chat_widget"     // chatbot embebible o página completa
  | "form"            // formulario conversacional
  | "landing"         // página de captura con chatbot
  | "dashboard_view"  // vista de métricas (solo lectura, para clientes finales)
```

La interfaz pública nunca expone credenciales ni configuración interna del agente.

---

## Rate limiting para soluciones públicas

```typescript
// Limitar mensajes por IP para soluciones públicas (evitar abuso)
// Usar un contador en memoria o Redis con ventana deslizante

const PUBLIC_RATE_LIMIT = {
  requests_per_minute: 20,      // por IP
  requests_per_hour: 200,       // por IP
  daily_org_budget: "from_plan" // usa el límite mensual del plan del dueño
};
```

---

## Límites por plan

| Plan | Deployments activos | Custom domain |
|---|---|---|
| Free | 1 | No |
| Starter | 5 | No |
| Pro | Ilimitados | Sí |

---

## Widget embebible

Para que los usuarios puedan incrustar la solución en su propio sitio web:

```html
<!-- Código que la plataforma genera para el usuario -->
<script src="https://<plataforma>.replit.app/embed.js" 
        data-slug="<slug>" 
        data-theme="light">
</script>
```

El script inyecta un iframe o un chat widget flotante configurado con el slug del deployment.

---

## Reglas clave

- La ruta `/s/:slug` y `/s/:slug/chat` son públicas — SIN autenticación de la plataforma
- El agente que se ejecuta en la solución pública usa las credenciales de la org dueña (no del visitante)
- Los mensajes enviados desde soluciones públicas sí cuentan para el límite mensual de IA del plan del dueño
- Al pausar un deployment, la URL devuelve una página de "temporalmente inactivo" (no 404)
- Nunca reutilizar un slug archivado — marcar como `archived` y dejar reservado

---

## Métricas de la solución desplegada

Registrar en `analytics_events` con `project_id`:
- `deployment.view` — visita a la URL pública
- `deployment.chat_message` — mensaje enviado en el chatbot público
- `deployment.form_submitted` — formulario completado
- `deployment.widget_loaded` — widget embebido cargado en sitio externo

---

## Referencias

- Ver `platform-architecture` skill para estructura de datos completa
- Ver `freemium-gates` skill para límites de deployments por plan
- Ver `ai-agent-config` skill para ejecución de agentes en modo público
- Ver `platform-analytics` skill para métricas de soluciones desplegadas
