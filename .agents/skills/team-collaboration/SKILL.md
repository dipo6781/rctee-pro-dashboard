---
name: team-collaboration
description: Gestión de equipos, roles y permisos dentro de la plataforma. Úsala al construir el módulo de equipo, al implementar invitaciones, roles de acceso, o al controlar qué acciones puede realizar cada miembro de una organización.
---

# Team Collaboration

Cada organización puede tener múltiples miembros con roles distintos. La gestión de membresías usa Clerk Organizations como fuente de verdad — no reimplementar lógica de invitación/roles en la DB.

---

## Roles del equipo

```typescript
type OrgRole =
  | "org:admin"    // Dueño/admin — acceso total, gestión de billing y equipo
  | "org:member"   // Miembro — acceso a proyectos asignados
  | "org:viewer"   // Solo lectura — puede ver pero no editar ni publicar
```

Los roles son definidos en Clerk y reflejados en el token JWT del usuario. Leer `.local/skills/clerk-auth/SKILL.md` para setup de organizaciones.

---

## Middleware de autorización por rol

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
router.post("/projects", requireRole("org:admin", "org:member"), createProject);
router.delete("/projects/:id", requireRole("org:admin"), deleteProject);
router.get("/projects", requireRole("org:admin", "org:member", "org:viewer"), listProjects);
```

---

## Matriz de permisos

| Acción | Admin | Member | Viewer |
|---|---|---|---|
| Ver proyectos | ✅ | ✅ | ✅ |
| Crear/editar proyectos | ✅ | ✅ | ❌ |
| Eliminar proyectos | ✅ | ❌ | ❌ |
| Publicar soluciones | ✅ | ✅ | ❌ |
| Ver CRM | ✅ | ✅ | ✅ |
| Crear/editar clientes | ✅ | ✅ | ❌ |
| Exportar datos | ✅ | ✅ | ❌ |
| Gestionar integraciones | ✅ | ✅ | ❌ |
| Invitar miembros | ✅ | ❌ | ❌ |
| Cambiar roles | ✅ | ❌ | ❌ |
| Gestionar billing | ✅ | ❌ | ❌ |
| Ver analytics | ✅ | ✅ | ✅ |

---

## Invitaciones (delegadas a Clerk)

Clerk maneja el flujo de invitación por email. En la plataforma solo necesitas:

```typescript
// Mostrar el estado del equipo (datos de Clerk)
GET /api/team/members     → listar miembros de la org con sus roles
GET /api/team/invitations → listar invitaciones pendientes

// Las invitaciones se envían desde el frontend usando el SDK de Clerk:
// organization.inviteMember({ emailAddress, role })
// No crear endpoints propios para esto — Clerk lo maneja.
```

---

## Asignación de recursos por miembro

Los proyectos y clientes del CRM pueden asignarse a un miembro específico:

```typescript
// En las tablas projects y clients:
assigned_to: uuid | null  // user_id del miembro responsable

// El miembro puede filtrar "mis proyectos" o "mis clientes"
// GET /api/projects?assigned_to=me
// GET /api/clients?assigned_to=me
```

---

## Límites de equipo por plan

| Plan | Miembros máximos |
|---|---|
| Free | 1 (solo el admin) |
| Starter | 3 |
| Pro | Ilimitados |

Al intentar invitar a un miembro adicional que supere el límite, devolver HTTP 402 con `resource: "team_members"`.

---

## Audit log de acciones del equipo

Registrar en `analytics_events` las acciones sensibles:

```typescript
// Eventos a registrar
| "team.member_invited"
| "team.member_removed"
| "team.role_changed"
| "team.project_assigned"
| "billing.plan_changed"
```

Esto permite al admin ver quién hizo qué en la organización.

---

## Reglas clave

- **Nunca reimplementar invitaciones** — usar Clerk Organizations
- El `org:admin` no puede ser removido si es el único admin (verificar antes de procesar)
- Los `org:viewer` nunca reciben endpoints de escritura — usar `requireRole` en todas las rutas mutantes
- Al remover un miembro, reasignar sus proyectos/clientes a `assigned_to = null` (no al admin automáticamente)
- Los datos de la organización persisten aunque el dueño original abandone — el rol de admin puede transferirse

---

## Notificaciones de equipo

Cuando ocurren eventos relevantes, notificar a los miembros correspondientes:
- Nueva invitación aceptada → notificar al admin
- Proyecto asignado → notificar al miembro asignado
- Solución desplegada → notificar a todos los admins y members

Implementar con un sistema de notificaciones in-app (tabla `notifications`) o integración con email/Slack via `integration-connectors`.

---

## Referencias

- Leer `.local/skills/clerk-auth/SKILL.md` antes de implementar — es la fuente de verdad para auth
- Ver `freemium-gates` skill para límites de miembros por plan
- Ver `multi-tenancy` skill para scoping de datos por organización
- Ver `platform-analytics` skill para el audit log
