# AGENTS.md

Punto de entrada para humanos y agentes. El detalle de implementación está en `.cursor/rules/` y en las specs.

## Producto

**Feature Flags Dashboard**: herramienta interna para crear, configurar y evaluar feature flags booleanas en runtime (sin redeploy). Targeting por ambiente (`dev` / `staging` / `prod`), empresa (`tenant_id`) y rollout porcentual. Persistencia local SQLite. Un único usuario demo.

Fuente de verdad de producto: `docs/prds/feature-flags-mvp.md`.

## Stack

pnpm workspaces + Turborepo. Next.js (App Router) + Tailwind en web. Hono en API. Drizzle + libSQL/SQLite. TypeScript estricto. Vitest.

Packages: `@ff/web`, `@ff/api`, `@ff/db`, `@ff/domain`.

## Monorepo

```
apps/web          UI (puerto 3000)
apps/api         HTTP (puerto 3001)
packages/db      schema, migraciones, seed
packages/domain  tipos y lógica pura (evaluador)
docs/prds        PRD
docs/specs       specs de ingeniería (01–11)
.cursor/rules    contrato de implementación
```

Quién depende de quién, qué vive en cada package y el contrato del evaluador: ver reglas en `.cursor/rules/` (arquitectura, TypeScript, React, Vitest, evaluador, estilo de agentes).

## Tests

Desde la raíz:

```sh
pnpm test
```

Vitest en `packages/domain`, `packages/db` y `apps/api`. Tests en `*.test.ts`. Criterios mínimos: regla `vitest-testing`.

Arranque local (cuando el monorepo esté scaffolded): `pnpm install` y `pnpm dev` (web 3000, api 3001).

## Flujo de trabajo

1. Leer este archivo y las reglas de `.cursor/rules/`.
2. Implementar **una spec a la vez**, en orden numérico, desde `docs/specs/`.
3. No adelantar specs posteriores ni trabajo fuera de su In scope / Out of scope.
4. Cerrar la spec activa (criterios de aceptación + tests que apliquen) antes de pasar a la siguiente.

Specs: `01-monorepo-setup` → `02-testing-setup` → `03-database-schema-and-seed` → `04-api-feature-flags` → `05-basic-login` → `06-dashboard-list` → `07-create-edit-flag` → `08-targeting-rules` → `09-flag-evaluator` → `10-simulator` → `11-history-and-review`.
