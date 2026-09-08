# Spec 04 — API feature flags

## Objetivo

Exponer en Hono los endpoints de gestión de feature flags (listar, obtener, crear, actualizar, cambiar status) persistiendo en SQLite vía `@ff/db`, validando `key`, fijando `review_at` (+90 días) y registrando auditoría en cada mutación (RF-02…05, RF-06, RF-16 parcial, RF-18 escritura).

## Contexto y dependencias

- **PRD:** RF-02, RF-03, RF-04 (campos básicos; targeting completo en spec 08), RF-05, RF-06, RF-16 (sin evaluate), RF-18 (escritura), RF-19.
- **Requiere:** Specs 01–03 (API esqueleto, Vitest, schema + migrate + seed).
- **Auth:** aún no implementada (spec 05). En esta spec los endpoints de gestión quedan **abiertos**; spec 05 los protegerá. Documentar TODO claro en rutas.

## Alcance

### In scope
- Módulo rutas `apps/api/src/routes/flags.ts` montado en `/api/flags`.
- Contratos:

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/api/flags` | Lista flags (incluye status, owner, key, review_at; puede incluir resumen de targeting si ya está en DB) |
| `GET` | `/api/flags/:id` | Detalle por id (o por key si se documenta `?key=`; preferir **id** en path y opcional `GET /api/flags/by-key/:key`) |
| `POST` | `/api/flags` | Crea flag: body `{ key, description, owner }`; status inicial `draft`; `review_at = now + 90d`; crear defaults de ambiente en `false` si no se envían |
| `PATCH` | `/api/flags/:id` | Actualiza `description`, `owner` (no key); body parcial |
| `POST` | `/api/flags/:id/status` | Body `{ status: "draft"\|"active"\|"archived", reason: string }`; `reason` obligatorio |

- Validación `key`: regex `^[a-z][a-z0-9_]*\.[a-z][a-z0-9_]*$` (área.feature en minúsculas); preferible función en `packages/domain`.
- Conflictos: `key` duplicada → `409`; key inválida → `400`; no encontrada → `404`.
- Cada `POST`/`PATCH`/cambio de status escribe fila(s) en `audit_log` con `actor: "demo"` (placeholder hasta login), timestamp, field/entity, old/new, reason (reason obligatorio en status; en create reason puede ser `"create"`).
- Tests Vitest de rutas con DB temporal: create → list → get → patch → status active → archived; reject invalid key; reject duplicate key.
- CORS básico permitiendo `http://localhost:3000` (preparación web).

### Out of scope
- `POST /api/flags/evaluate` o `/api/evaluate` (spec 09).
- Endpoints dedicados de overrides/rollout (pueden ir en PATCH anidado en spec 08; **no** requerir UI).
- Login/cookies (spec 05).
- Frontend.

### Targeting en esta spec
- Al crear, insertar tres filas `environment_defaults` (`dev`/`staging`/`prod`) en `false`.
- No exponer aún CRUD de overrides/% (spec 08); si el GET detalle devuelve defaults vacíos/false, está bien.

## Tareas en orden

1. Mover validación de `key` a `packages/domain` (`isValidFlagKey` + tests).
2. Implementar repositorio/servicio `flagsService` en api (o en db) usando Drizzle.
3. Montar rutas CRUD + status; respuestas JSON consistentes `{ data }` / `{ error: { code, message } }`.
4. Escribir `audit_log` en el mismo request de mutación (transacción si libSQL/Drizzle lo permite).
5. Tests de integración API + DB temp.
6. Asegurar seed sigue funcionando; list endpoint devuelve seeded flags.

## Criterios de aceptación verificables

1. `POST /api/flags` con `key: "billing.new_checkout"` crea status `draft` y `review_at` ~ +90 días (RF-02, RF-06; MVP #2 parcial).
2. `POST` con `key: "InvalidKey"` o `"no-dot"` → `400` (RF-03).
3. Segundo `POST` con misma key → `409` (RF-03).
4. `POST .../status` con `{ status: "active", reason: "rollout" }` cambia status; sin `reason` → `400` (RF-05, RF-18).
5. `GET /api/flags` y `GET /api/flags/:id` reflejan cambios inmediatos (RNF-05).
6. Tras mutación, existe fila en `audit_log` con old/new y actor (RF-18).
7. Suite Vitest de api para estos casos pasa con `pnpm --filter @ff/api test`.

## Notas técnicas

- IDs: `crypto.randomUUID()`.
- Timestamps: ISO-8601 strings UTC.
- No borrar flags en MVP (no `DELETE` requerido).
- Actor fijo `"demo"` hasta spec 05; entonces usar identidad de sesión.
- Evaluate y precedencia de targeting no se implementan aquí.
