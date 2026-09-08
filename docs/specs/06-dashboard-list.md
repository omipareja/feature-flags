# Spec 06 — Dashboard list

## Objetivo

Construir el dashboard autenticado que lista feature flags desde la API, mostrando información clave para operar el MVP (RF-17 parcial — listado).

## Contexto y dependencias

- **PRD:** RF-17 (listar flags en UI); criterio MVP #10 parcial.
- **Requiere:** Specs 01–05 (monorepo, API CRUD, login con cookie/proxy).
- **Estado esperado:** usuario puede autenticarse; `GET /api/flags` devuelve flags (seed + creadas).

## Alcance

### In scope
- Ruta protegida `/` o `/flags` como dashboard principal post-login.
- Tabla o lista (no hace falta design system elaborado) con columnas mínimas:
  - `key`
  - `status` (draft / active / archived)
  - `owner`
  - `review_at` (fecha visible)
  - `updated_at` (opcional pero recomendado)
- Estados de UI: loading, vacío (“No hay flags”), error de red/401.
- CTA visible “Create flag” que enlace a `/flags/new` (página puede ser placeholder hasta spec 07).
- Click en una fila o link “Edit” → `/flags/[id]` (placeholder permitido hasta spec 07).
- Fetch vía same-origin `/api/flags` con credentials (según proxy de spec 05).
- Estilos con Tailwind; layout simple con header mostrando usuario demo y logout.

### Out of scope
- Formulario create/edit completo (spec 07).
- Editor de targeting (spec 08).
- Simulador de evaluación (spec 10).
- Historial de auditoría (spec 11).
- Filtros avanzados, búsqueda full-text, paginación server (opcional client-side si hay pocas flags; no requerido).

## Tareas en orden

1. Crear layout autenticado (`apps/web/src/app/(app)/layout.tsx`) con nav: Flags, Logout; link Login fuera.
2. Implementar página lista que llame `GET /api/flags`.
3. Tipar respuesta según contrato API (`data: Flag[]`).
4. Renderizar tabla + empty/error/loading.
5. Añadir links a create y detalle.
6. Verificar manualmente con seed: tras login se ven ≥ 2 flags.
7. Test opcional: si hay Vitest en web, test de función formatStatus; no bloquear si no existe harness UI.

## Criterios de aceptación verificables

1. Con sesión válida, el dashboard muestra todas las flags devueltas por `GET /api/flags` con `key`, `status`, `owner`, `review_at`.
2. Sin sesión, al visitar el dashboard se redirige a `/login` (heredado spec 05).
3. Si la API devuelve `[]`, se muestra estado vacío (no crash).
4. Si la API falla, se muestra mensaje de error (no pantalla en blanco silenciosa).
5. Existe control visible para ir a crear flag y para abrir detalle de una flag existente.
6. Logout desde el header invalida el acceso al listado.

## Notas técnicas

- Preferir Server Components con fetch a `http://localhost:3001` solo si se reenvían cookies; con rewrite same-origin, fetch relativo `/api/flags` desde Client Component o Route Handler es más simple — **usar Client Component con useEffect o React Query no requerido; fetch en client tras mount es aceptable**.
- No inventar datos mock si la API está caído; mostrar error.
- Accesibilidad mínima: tabla con `<table>` o lista con headings; botones con texto claro.
- Spec 07 reemplazará placeholders de create/edit; no implementar formularios aquí más allá de links.
