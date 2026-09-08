# Spec 01 — Monorepo setup

## Objetivo

Crear el monorepo vacío con la estructura de paquetes, tooling TypeScript, Turborepo, Tailwind en el frontend y scripts raíz para desarrollar `apps/web` y `apps/api` sin lógica de negocio todavía.

## Contexto y dependencias

- **Fuente de verdad de producto:** `docs/prds/feature-flags-mvp.md` (PRD MVP).
- **Dependencias de specs anteriores:** ninguna (primera spec).
- **Stack bloqueado:** pnpm workspaces + Turborepo; Next.js (App Router) en `apps/web`; Hono + `@hono/node-server` en `apps/api`; TypeScript estricto; Tailwind CSS en web; paquetes `packages/db` y `packages/domain` como shells (sin schema ni evaluador aún).

## Alcance

### In scope
- Root: `package.json`, `pnpm-workspace.yaml`, `turbo.json`, `tsconfig.base.json`, `.gitignore`, `.nvmrc` o engines Node ≥ 20, README mínimo de arranque.
- Workspaces: `apps/web`, `apps/api`, `packages/db`, `packages/domain`.
- `apps/web`: Next.js App Router + TypeScript + Tailwind; página placeholder (p. ej. “Feature Flags”); script `dev` en puerto 3000.
- `apps/api`: Hono con ruta `GET /health` → `{ "ok": true }`; servidor Node en puerto 3001; TypeScript.
- `packages/db`: `package.json` + `tsconfig` + export placeholder (`export {}` o `ping()`); dependencia lista para Drizzle/libSQL en spec 03 (puede declarar deps sin schema).
- `packages/domain`: `package.json` + `tsconfig` + export placeholder de tipos futuros.
- Scripts turbo: `build`, `dev`, `lint`, `typecheck` a nivel root.
- Referencias de workspace (`workspace:*`) entre apps y packages según necesidad mínima (web/api pueden depender de domain/db aunque aún vacíos).

### Out of scope
- Vitest (spec 02).
- Schema SQLite, migraciones, seed (spec 03).
- CRUD de flags, auth, UI de producto, evaluador.
- OAuth, Docker, CI cloud, deploy.

## Tareas en orden

1. Inicializar repo con pnpm: `pnpm-workspace.yaml` incluyendo `apps/*` y `packages/*`.
2. Añadir Turborepo (`turbo.json`) con pipelines `build`, `dev` (persistent), `lint`, `typecheck`.
3. Crear `tsconfig.base.json` (strict: `true`, `moduleResolution: bundler` o `node16` consistente en el monorepo).
4. Scaffold `packages/domain` y `packages/db` como librerías TS compilables o consumibles vía `exports` / `main`.
5. Scaffold `apps/api` con Hono: entry `src/index.ts`, `GET /health`, listen `3001`.
6. Scaffold `apps/web` con `create-next-app` (o equivalente manual): App Router, Tailwind, página raíz mínima.
7. Cablear scripts root: `pnpm dev` corre web + api vía turbo; `pnpm build` / `pnpm typecheck`.
8. Documentar en README raíz: cómo instalar (`pnpm install`) y arrancar (`pnpm dev`); puertos 3000/3001.
9. Verificar que `pnpm install`, `pnpm typecheck` y `curl localhost:3001/health` funcionan tras `pnpm --filter api dev` (o turbo).

## Criterios de aceptación verificables

1. Existen exactamente (como mínimo) las carpetas: `apps/web`, `apps/api`, `packages/db`, `packages/domain`.
2. `pnpm install` completa sin error desde la raíz.
3. `pnpm typecheck` (o equivalente turbo) pasa en todos los workspaces.
4. Con la API en marcha, `GET http://localhost:3001/health` responde `200` y body JSON con `ok: true`.
5. Con web en marcha, `GET http://localhost:3000` responde `200` y muestra el placeholder.
6. No hay endpoints de flags ni tablas de negocio en este entregable.
7. El README describe install + `pnpm dev` y los puertos.

## Notas técnicas

- Gestor de paquetes: **pnpm** únicamente (no mezclar npm/yarn).
- Nombres de package sugeridos: `@ff/web`, `@ff/api`, `@ff/db`, `@ff/domain` (o prefijo del repo; ser consistente).
- CORS: aún no obligatorio; se añadirá cuando web llame a api (spec 05+). Dejar comentario o TODO en api si se prefiere.
- Variables de entorno: crear `.env.example` vacío o con `API_PORT=3001` / `NEXT_PUBLIC_API_URL=http://localhost:3001` para specs siguientes; no secretos reales.
- Persistencia y auth quedan para specs posteriores; esta spec solo deja el esqueleto compilable.
