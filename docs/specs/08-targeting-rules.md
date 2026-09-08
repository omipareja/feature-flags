# Spec 08 — Targeting rules

## Objetivo

Permitir configurar y persistir reglas de targeting por flag: defaults por ambiente (`dev`/`staging`/`prod`), overrides por `tenant_id`, y rollout porcentual 0–100 (o desactivado), vía API y UI (RF-04, RF-07, RF-08, RF-09).

## Contexto y dependencias

- **PRD:** RF-04, RF-07, RF-08, RF-09; tablas de spec 03; mutaciones auditan (RF-18).
- **Requiere:** Specs 01–07 (detalle de flag en UI, API flags, auth).
- **Evaluación runtime:** no implementar aquí (spec 09); solo persistencia + UI de configuración.

## Alcance

### In scope
- **API** (protegida por sesión):
  - `PUT /api/flags/:id/targeting` (o `PATCH`) con body:
    ```json
    {
      "reason": "string-obligatorio",
      "environmentDefaults": {
        "dev": false,
        "staging": true,
        "prod": false
      },
      "tenantOverrides": [
        { "tenantId": "acme", "enabled": true }
      ],
      "rollout": { "percentage": 50 }
    }
    ```
  - `rollout: null` o `{ "percentage": null }` = porcentaje **desactivado**.
  - Validar environments solo `dev|staging|prod`; percentage entero 0–100; `tenantId` string no vacío; lista de overrides reemplaza el set completo (replace semantics) o documentar merge — **usar replace completo** del array de overrides.
  - `GET /api/flags/:id` debe devolver targeting actual embebido.
  - Escribir `audit_log` por cambio (al menos un entry resumen o por campo) con reason.
- **UI** en `/flags/[id]` sección “Targeting”:
  - Tres toggles/checkboxes de default por ambiente.
  - Lista editable de overrides (tenant id + enabled); add/remove rows.
  - Input percentage 0–100 + control “Disable percentage”.
  - Campo reason + botón Save targeting.
- Tests API: put targeting, get refleja, percentage 101 → 400, audit row created.

### Out of scope
- Evaluador / precedencia runtime (spec 09).
- Simulador UI (spec 10).
- Variantes A/B, segmentación por país, etc.
- Sub-tenants.

## Tareas en orden

1. Extender servicio DB/API para leer/escribir `environment_defaults`, `tenant_overrides`, `rollouts`.
2. Implementar endpoint PUT targeting con validación y auditoría.
3. Extender respuesta `GET /api/flags/:id` con bloque `targeting`.
4. Construir sección UI y cablear save.
5. Tests Vitest API para casos felices y de validación.
6. Verificar con seed: editar override `acme` y percentage y recargar página.

## Criterios de aceptación verificables

1. Se pueden guardar defaults distintos por `dev`/`staging`/`prod` y persisten tras reload (RF-07; MVP #3 datos listos).
2. Se puede agregar, editar y eliminar override por `tenant_id` booleano (RF-08).
3. Se puede setear percentage 0–100 y desactivarlo explícitamente (RF-09).
4. Percentage fuera de rango o environment inválido → `400`.
5. Save sin `reason` → `400` / UI bloquea.
6. Tras save, `GET` detalle devuelve exactamente los overrides enviados (replace).
7. Queda registro en `audit_log` del cambio (RF-18).

## Notas técnicas

- Precedencia **no** se calcula en esta spec; solo se almacena.
- Si `rollouts.percentage` es `NULL` o fila ausente → “sin % activo” para el evaluador futuro.
- Transacción recomendada al reemplazar overrides (delete all + insert).
- Actor de audit = username de sesión.
- No cambiar el contrato de create flag de spec 04 más allá de lo necesario.
