# Spec 02 — Testing setup

## Objetivo

Configurar Vitest en el monorepo de forma que `packages/domain`, `packages/db` y `apps/api` puedan ejecutar tests unitarios/integración locales con un comando raíz único, listo para las specs de schema, API y evaluador.

## Contexto y dependencias

- **PRD:** `docs/prds/feature-flags-mvp.md` (soporte a verificación continua de RFs).
- **Requiere completada:** Spec 01 — monorepo con `apps/web`, `apps/api`, `packages/db`, `packages/domain`, pnpm + turbo.
- **Estado esperado al empezar:** `pnpm typecheck` pasa; `GET /health` existe en la API.

## Alcance

### In scope
- Vitest como runner único del monorepo.
- Configuración por workspace que lo necesite: como mínimo `packages/domain`, `packages/db`, `apps/api`.
- Script root `pnpm test` (turbo `test`) que ejecute todos los workspaces con tests.
- Un test smoke por workspace configurado (p. ej. `expect(true).toBe(true)` o test de `/health` con `app.request` de Hono).
- `apps/web`: opcional en esta spec; si no se configura Vitest en web, documentarlo como diferido (UI se validará manualmente / en specs UI). Preferir **no** bloquear: o configurar Vitest mínimo en web o excluirlo explícitamente del pipeline `test` con comentario en turbo.

### Out of scope
- Coverage gates estrictos (umbral %) — opcional documentar `coverage` sin fallar CI.
- E2E Playwright/Cypress.
- Tests de negocio de flags/evaluador (specs 03–11).
- Snapshot testing de UI.

## Tareas en orden

1. Añadir `vitest` (y tipos si aplica) como dep de desarrollo en root o en cada package que testee.
2. Crear `vitest.config.ts` (o `.mts`) en `packages/domain`, `packages/db`, `apps/api` con `environment: "node"` y resolución de path aliases del package.
3. Añadir script `"test": "vitest run"` y `"test:watch": "vitest"` en cada uno de esos `package.json`.
4. Registrar task `test` en `turbo.json` (dependsOn opcional de `^build` solo si los packages exportan dist; preferir test sobre source TS).
5. Escribir smoke test:
   - `packages/domain`: importa el entry y aserta que el módulo carga.
   - `packages/db`: igual.
   - `apps/api`: `GET /health` vía `app.request("http://localhost/health")` sin levantar puerto real.
6. Añadir `pnpm test` en root que invoque turbo test.
7. Actualizar README con cómo correr tests.

## Criterios de aceptación verificables

1. `pnpm test` desde la raíz termina con exit code 0.
2. Existen al menos un archivo `*.test.ts` (o `*.spec.ts`) en `packages/domain`, `packages/db` y `apps/api`.
3. El test de API valida status 200 y `ok: true` en `/health` sin abrir socket en 3001.
4. `pnpm test` es determinista (sin flaky por timers/red).
5. No se introducen dependencias de testing de producto (flags) todavía.

## Notas técnicas

- Usar Vitest 2.x (o la estable actual del ecosistema al implementar); API compatible con `describe`/`it`/`expect`.
- Para Hono: exportar `app` desde un módulo (`src/app.ts`) separado del `listen` en `src/index.ts` para poder testear sin bind de puerto — refactor mínimo permitido en esta spec.
- SQLite en tests de db llegará en spec 03 (DB en memoria o archivo temp); aquí solo el runner.
- No exigir tests en `apps/web` para cerrar esta spec; si se añaden, no deben fallar el pipeline.
