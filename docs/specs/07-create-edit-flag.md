# Spec 07 — Create and edit flag

## Objetivo

Permitir crear y editar feature flags desde la UI (metadatos + cambio de status), alineado con validaciones de API: `key`, `description`, `owner`, status `draft|active|archived`, `review_at` automático al crear (RF-02, RF-03, RF-05, RF-06; RF-17 parcial).

## Contexto y dependencias

- **PRD:** RF-02, RF-03, RF-05, RF-06; MVP #2; RF-17 create/edit (sin targeting).
- **Requiere:** Specs 01–06 (API CRUD+status, login, dashboard con links a `/flags/new` y `/flags/[id]`).

## Alcance

### In scope
- Página `/flags/new`:
  - Campos: `key`, `description`, `owner`.
  - Submit → `POST /api/flags`; on success redirect a detalle o lista.
  - Mostrar errores 400/409 del API en el formulario.
- Página `/flags/[id]`:
  - Carga `GET /api/flags/:id`.
  - Editar `description`, `owner` → `PATCH /api/flags/:id`.
  - `key` **solo lectura** tras creación.
  - Control de status: select o botones Draft / Active / Archived; exige campo `reason` obligatorio antes de llamar `POST /api/flags/:id/status`.
  - Mostrar `review_at`, `created_at`, `status` actual.
- Validación client-side opcional del formato `key` (misma regla que domain); la fuente de verdad sigue siendo el API.
- Tras create/edit/status exitoso, el dashboard refleja cambios al volver.

### Out of scope
- Edición de defaults por ambiente, overrides, % (spec 08).
- Simulador (spec 10).
- Panel de historial (spec 11) — puede mostrarse un link placeholder “History” sin implementar.
- Borrado de flags.

## Tareas en orden

1. Implementar formulario create en `apps/web` con manejo de errores.
2. Implementar página detalle/edit + cambio de status con `reason`.
3. Asegurar redirects y toasts/mensajes simples de éxito.
4. Probar flujo manual: crear `billing.new_checkout`, activar con reason, archivar (MVP #2).
5. Añadir test de domain ya existente para `isValidFlagKey` si faltan casos; test API ya cubre backend.
6. Actualizar links del dashboard si las rutas difieren.

## Criterios de aceptación verificables

1. Desde UI se crea flag con key `billing.new_checkout`, status inicial `draft`, y `review_at` ~ +90 días visible en detalle (RF-02, RF-06; MVP #2).
2. Crear con key inválida muestra error y no crea fila (RF-03).
3. Crear key duplicada muestra error de conflicto (RF-03).
4. Se puede editar `description` y `owner` y persisten tras reload (RF-04 parcial metadatos).
5. Se puede pasar `draft → active → archived` solo enviando `reason` no vacío; sin reason el UI bloquea submit (RF-05, RF-18 motivo).
6. `key` no es editable en la pantalla de edición.
7. Usuario no autenticado no accede a `/flags/new` ni `/flags/[id]`.

## Notas técnicas

- Reutilizar `apiClient` con `credentials: "include"`.
- Status transitions: permitir cualquier transición entre los tres estados (el PRD no restringe máquina de estados más allá de los tres valores).
- No enviar targeting en esta spec; defaults `false` ya los crea el API en POST (spec 04).
- Mantener UI simple; un solo job por página (crear vs editar).
