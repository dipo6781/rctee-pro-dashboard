---
name: skill-creator
description: Genera nuevas skills personalizadas de forma dinámica según la necesidad del proyecto o del usuario. Úsala cuando el usuario pida crear una skill, automatizar un flujo repetitivo, o cuando detectes que un patrón de trabajo debería guardarse para reutilizarse en el futuro.
---

# Skill Creator

Esta skill te enseña a **analizar una necesidad y generar una skill nueva, bien estructurada y reutilizable**, sin que el usuario tenga que especificar cada detalle técnico.

---

## Cuándo activar esta skill

- El usuario dice "crea una skill para X"
- El usuario dice "quiero que recuerdes cómo hacer Y"
- Detectas un flujo repetitivo que debería automatizarse
- El usuario pide un asistente especializado en algún dominio (ej: "una skill para manejar migraciones de base de datos")
- El usuario dice "enséñate a hacer Z"

---

## Proceso de creación

### Paso 1 — Entender la necesidad

Antes de escribir nada, responde estas preguntas (usa contexto del proyecto o pregunta al usuario si es ambiguo):

1. **¿Cuál es el dominio?** (ej: pagos, autenticación, generación de imágenes, migraciones de DB)
2. **¿Cuál es el disparador?** ¿Cuándo debería un agente futuro usar esta skill?
3. **¿Qué pasos o reglas son no-obvios?** Solo incluye lo que no se puede inferir del código.
4. **¿Hay ejemplos concretos?** Input/output o comandos exactos.
5. **¿Depende de otras skills o archivos del proyecto?** Referéncialos.

Si la necesidad es vaga, pregunta UNA sola pregunta al usuario antes de continuar.

---

### Paso 2 — Elegir el nombre y ubicación

- Nombre: `lowercase-con-guiones`, máx 64 caracteres
- Ubicación: `.agents/skills/<nombre>/SKILL.md`
- Si la skill necesita archivos de referencia: `.agents/skills/<nombre>/references/<archivo>.md`

**Convenciones de nombre:**
| Dominio | Ejemplo de nombre |
|---|---|
| Integración externa | `stripe-payments`, `github-sync` |
| Flujo de datos | `csv-import`, `pdf-export` |
| Infraestructura | `db-migrations`, `env-setup` |
| Dominio de negocio | `invoice-generator`, `user-onboarding` |
| Meta (skills sobre skills) | `skill-creator` ← esta misma |

---

### Paso 3 — Escribir la skill

Usa esta plantilla como base y ajústala al dominio:

```markdown
---
name: <nombre>
description: <Qué hace. Cuándo usarla. Máx 1024 chars.>
---

# <Título legible>

## Cuándo usar

- <Trigger 1>
- <Trigger 2>

## Proceso

1. <Paso concreto>
2. <Paso concreto>
3. <Paso concreto>

## Reglas clave

- <Regla no-obvia 1>
- <Regla no-obvia 2>

## Ejemplos

\`\`\`
Input: ...
Output: ...
\`\`\`

## Referencias

- Ver `references/api.md` para detalles del contrato
- Depende de la skill `<otra-skill>`
```

---

### Paso 4 — Validar antes de guardar

Antes de escribir el archivo, verifica:

- [ ] El `description` explica CUÁNDO usarla, no solo QUÉ hace
- [ ] No duplica una skill que ya existe (revisa `.agents/skills/` y `.local/skills/`)
- [ ] No incluye secretos, tokens, credenciales ni PII
- [ ] No repite información derivable del código actual
- [ ] Está bajo 500 líneas (si es más, separa en archivos de referencia)
- [ ] Está escrita en el idioma que usa el usuario

---

### Paso 5 — Confirmar al usuario

Después de crear la skill, informa:
- El nombre y ubicación del archivo creado
- En una oración: qué hace la skill y cuándo se activará en el futuro
- Si hay algo que el usuario debería agregar o refinar

---

## Skills que puedes generar (ejemplos por dominio)

### Infraestructura y base de datos
- `db-migrations` — flujo para aplicar migraciones con Drizzle
- `env-setup` — cómo solicitar y configurar variables de entorno del proyecto
- `seed-data` — cómo poblar la base de datos con datos de prueba

### Integraciones externas
- `stripe-checkout` — flujo completo de pago con Stripe
- `sendgrid-emails` — envío de emails transaccionales
- `openai-streaming` — respuestas en streaming con OpenAI

### Flujos de contenido
- `pdf-export` — generar PDFs desde HTML o React
- `csv-import` — parsear y validar CSVs del usuario
- `image-optimizer` — redimensionar y comprimir imágenes

### Dominio de negocio (específico del proyecto)
- Cualquier flujo repetitivo que el usuario describa

---

## Reglas de calidad

1. **Concisa pero completa**: Incluye solo lo que un agente futuro NO puede inferir del código.
2. **Orientada a triggers**: El `description` es lo que activa la skill — escríbelo pensando en búsqueda semántica.
3. **Ejemplos > explicaciones**: Un ejemplo concreto vale más que un párrafo de prosa.
4. **Actualizable**: Las skills en `.agents/skills/` son mutables. Actualízalas si descubres mejores patrones.
5. **Composición**: Una skill puede referenciar otras skills para flujos complejos.

---

## Detección automática de necesidades

Además de crear skills cuando el usuario lo pide explícitamente, detecta estas señales implícitas:

| Señal | Acción |
|---|---|
| El usuario corrige el mismo error por tercera vez | Crea una skill con la regla correcta |
| Un flujo requirió más de 3 pasos no obvios | Crea una skill con ese flujo |
| El usuario dice "siempre que hagas X, haz Y" | Crea una skill de convención |
| Una integración tiene comportamiento no documentado | Crea una skill con las quirks |

En estos casos, **crea la skill sin esperar a que el usuario lo pida** y menciona brevemente que lo hiciste.
