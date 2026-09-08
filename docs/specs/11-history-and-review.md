# Spec 11 — History and review

## Objetivo

Exponer el historial de auditoría de cada flag en la UI y surfacer el estado de review (`review_at`, indicadores de vencido), cerrando RF-18 (consulta), RF-20 y el ciclo de review a 90 días del PRD.

## Contexto y dependencias

- **PRD:** RF-18, RF-20; ciclo de vida / `review_at`; MVP #8.
- **Requiere:** Specs 01–08 (mutaciones ya escriben `audit_log` con actor, old/new, reason). Specs 09–10 no son bloqueantes pero el monorepo completo es el estado esperado.
- **Escritura de audit:** ya existe desde specs 04/05/08; esta spec se enfoca en **lectura API + UI** y pulido de review.

## Alcance

### In scope
- API (autenticada):
  - `GET /api/flags/:id/audit` → lista ordenada desc por `created_at`:
    `{ id, actor, createdAt, entity, field, oldValue, newValue, reason }`.
- UI en `/flags/[id]`:
  - Pestaña o sección “History” que lista los eventos de auditoría (fecha, actor, campo, old → new, reason).
  - Estado vacío si no hay eventos.
- Review:
  - Mostrar `review_at` en listado (spec 06 ya) y detalle.
  - Indicador visual si `review_at <= now` (“Review due”) en dashboard y/o detalle.
  - Acción opcional MVP: botón “Mark reviewed” que extiende `review_at` a now+90d vía `POST /api/flags/:id/review` con `{ reason }` y escribe audit — **incluir este endpoint** para cerrar el loop operativo del TTL.
- Tests API: tras crear/status/targeting, `GET audit` devuelve entradas; mark reviewed actualiza `review_at`.

### Out of scope
- Export CSV, webhooks, alertas email.
- Approval workflow.
- Soft-delete de audit (append-only).
- RBAC sobre quién ve historial.

## Tareas en orden

1. Implementar `GET /api/flags/:id/audit` con auth.
2. Implementar `POST /api/flags/:id/review` → set `review_at = now + 90d`, audit field `review_at`.
3. Construir sección History en detalle de flag.
4. Añadir badge “Review due” en dashboard cuando aplique.
5. Tests Vitest para audit list + review.
6. Verificación manual MVP #8: cambiar status/targeting y ver old/new + reason en UI.

## Criterios de aceptación verificables

1. Tras mutaciones (create, patch, status, targeting), `GET /api/flags/:id/audit` lista eventos con actor, timestamp, old/new, reason (RF-18, RF-20; MVP #8).
2. La UI de History muestra esos eventos sin autenticación inválida (401 si no hay sesión).
3. Flags con `review_at` en el pasado muestran indicador “Review due” en el dashboard.
4. “Mark reviewed” mueve `review_at` ~ +90 días desde ahora y añade entrada de audit.
5. El log es append-only: mark reviewed no borra eventos previos.
6. Tests API de audit/review pasan.

## Notas técnicas

- Orden: `created_at DESC`, límite default 100 (documentar query `?limit=` opcional).
- `oldValue`/`newValue` pueden ser JSON stringified; la UI puede mostrar texto plano.
- Actor siempre el usuario demo de sesión.
- No requiere evaluate ni simulador para estar completa, pero no debe romperlos.
- Con esta spec se considera cerrado el MVP documental del PRD a nivel funcional UI+API.

## Cobertura PRD (cierre)

Esta spec completa la trazabilidad de:

| Ítem | Cubierto en |
|------|-------------|
| RF-01 | Spec 05 |
| RF-02…06 | Specs 04, 07 |
| RF-07…09 | Spec 08 |
| RF-10…15 | Spec 09 |
| RF-16 | Specs 04 + 09 |
| RF-17 | Specs 06, 07, 08, 10 |
| RF-18 | Specs 04, 08 (write) + 11 (read) |
| RF-19 | Spec 03 |
| RF-20 | Spec 11 |
| MVP #1…#10 | Specs 05–11 según tabla del plan |
