---
name: team-collaboration
description: Gestión de equipos y roles en R-C-T-E-E Pro. Úsala al construir el módulo de equipo, al implementar invitaciones, roles de acceso, o al controlar qué acciones puede realizar cada miembro. El "equipo" es el grupo de consultores/asistentes dentro de una agencia o firma consultora.
---

# Team Collaboration — R-C-T-E-E Pro

Cada organización puede tener múltiples consultores trabajando juntos. La gestión de membresías usa **Clerk Organizations** como fuente de verdad. No reimplementar invitaciones ni manejo de tokens en la plataforma.

---

## Roles

```typescript
type OrgRole =
  | "org:admin"     // Dueño de la firma/agencia — acceso total, billing, configuración
  | "org:member"    // Consultor — puede crear clientes, proyectos y generar entregables
  | "org:viewer"    // Asistente — solo puede ver proyectos y descargar entregables existentes
```

Los roles vienen en el JWT de Clerk. Leer `.local/skills/clerk-auth/SKILL.md` para extracción correcta.

---

## Middleware de autorización

```typescript
// src/middlewares/authorize.ts
import { getAuth } from "@clerk/express";

export function requireRole(...roles: OrgRole[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    const { orgRole } = getAuth(req);

    if (!orgRole || !roles.includes(orgRole as OrgRole)) {
      return res.status(403).json({
        error: "insufficient_permissions",
        required: roles,
        current: orgRole ?? "none",
      });
    }
    next();
  };
}

// Uso en rutas:
router.post("/clients",              requireRole("org:admin", "org:member"), createClient);
router.post("/deliverables/generate",requireRole("org:admin", "org:member"), generateDeliverable);
router.delete("/clients/:id",        requireRole("org:admin"),               deleteClient);
router.get("/analytics/summary",     requireRole("org:admin", "org:member", "org:viewer"), getAnalytics);
```

---

## Matriz de permisos

| Acción | Admin | Member | Viewer |
|---|---|---|---|
| Ver clientes y proyectos | ✅ | ✅ | ✅ |
| Crear/editar clientes | ✅ | ✅ | ❌ |
| Generar entregables | ✅ | ✅ | ❌ |
| Descargar entregables | ✅ | ✅ | ✅ |
| Compartir entregable por WhatsApp | ✅ | ✅ | ❌ |
| Ver analytics | ✅ | ✅ | ✅ |
| Configurar integraciones (WhatsApp, etc.) | ✅ | ❌ | ❌ |
| Invitar miembros | ✅ | ❌ | ❌ |
| Cambiar roles | ✅ | ❌ | ❌ |
| Gestionar billing / plan | ✅ | ❌ | ❌ |
| Eliminar clientes / proyectos | ✅ | ❌ | ❌ |

---

## Invitaciones (delegadas a Clerk)

```typescript
// Las invitaciones se envían desde el frontend usando el SDK de Clerk:
// organization.inviteMember({ emailAddress, role: "org:member" })
// NO crear endpoints propios para invitaciones — Clerk maneja el email y el token.

// Solo necesitas estos endpoints propios:
// GET /api/team/members     → listar miembros con nombre, email, rol
// GET /api/team/invitations → invitaciones pendientes (también viene de Clerk)
```

---

## Asignación de clientes y proyectos

```typescript
// Los clientes y proyectos pueden asignarse a un miembro específico del equipo
// Campos: assigned_to: string | null  (Clerk user ID)

// Filtro "mis clientes":
// GET /api/clients?assigned_to=me
// El backend reemplaza "me" con req.userId del token de Clerk
```

---

## Límites de equipo por plan

| Plan | Miembros |
|---|---|
| Free | 1 (solo el admin) |
| Starter | 3 |
| Pro | 10 |
| Agency | Ilimitados |

Al intentar invitar por encima del límite → HTTP 402 con `resource: "team_members"`.

---

## Caso de uso típico: Agencia con equipo

```
Admin (dueño de la agencia):
  - Configura las integraciones (WhatsApp, Drive)
  - Gestiona el billing y el plan
  - Asigna clientes a los consultores

Consultant (org:member):
  - Atiende sus clientes asignados
  - Genera entregables desde las plantillas
  - Envía documentos por WhatsApp al cliente

Viewer (asistente):
  - Revisa proyectos activos
  - Descarga entregables para revisión interna
  - No puede generar ni modificar
```

---

## Audit log de acciones sensibles

Registrar en `analytics_events`:

```typescript
| "team.member_invited"        // admin invitó un miembro
| "team.member_removed"        // admin removió un miembro
| "team.role_changed"          // cambio de rol
| "billing.plan_changed"       // upgrade o downgrade del plan
| "integration.configured"     // configuró WhatsApp u otra integración
```

---

## Reglas clave

- **Nunca reimplementar invitaciones** — usar Clerk Organizations exclusivamente
- El `org:admin` no puede ser removido si es el último admin (verificar antes de procesar la remoción)
- Al remover un miembro, sus clientes y proyectos asignados quedan con `assigned_to = null`
- Los datos de la organización persisten aunque el dueño original abandone — el rol de admin puede transferirse
- El Clerk `orgRole` viene en el JWT — no guardarlo en la DB, leerlo siempre del token

---

## Referencias

- Leer `.local/skills/clerk-auth/SKILL.md` — fuente de verdad para auth y roles
- Ver `freemium-gates` skill para límites de miembros por plan
- Ver `multi-tenancy` skill para scoping de datos por organización
