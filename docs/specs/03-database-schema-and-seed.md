# Spec 03 — Database schema and seed

## Objetivo

Definir el schema Drizzle para feature flags, reglas de targeting y auditoría en SQLite/libSQL local, con migraciones, cliente de DB reutilizable y seed de datos de ejemplo que sobrevivan reinicios (RF-06, RF-19).

## Contexto y dependencias

- **PRD:** secciones 5 (dominio), RF-06, RF-19; criterios MVP #9.
- **Requiere:** Specs 01 y 02 (monorepo + Vitest).
- **Paquete:** implementar en `packages/db`; consumible por `apps/api` (wiring mínimo de import permitido; CRUD HTTP es spec 04).

## Alcance

### In scope
- Tablas (nombres exactos a usar):
  - `feature_flags`: `id` (text PK uuid), `key` (text unique), `description` (text), `owner` (text), `status` (`draft` \| `active` \| `archived`), `created_at`, `updated_at`, `review_at` (ISO text o integer unix — documentar uno y usarlo siempre).
  - `environment_defaults`: `id`, `flag_id` FK, `environment` (`dev` \| `staging` \| `prod`), `enabled` (boolean/int), unique `(flag_id, environment)`.
  - `tenant_overrides`: `id`, `flag_id` FK, `tenant_id` (text), `enabled` (boolean), unique `(flag_id, tenant_id)`.
  - `rollouts`: `flag_id` PK/FK, `percentage` (integer 0–100 nullable o fila ausente = sin %), `enabled` (boolean “porcentaje activo”) — o columna `percentage` nullable en `feature_flags`; **elegir un modelo y documentarlo en notas de esta implementación**. Preferido: tabla `rollouts` con `percentage INTEGER NULL` donde `NULL` = porcentaje desactivado.
  - `audit_log`: `id`, `flag_id` (nullable si se borra), `actor` (text), `created_at`, `entity` (text), `field` (text), `old_value` (text/json), `new_value` (text/json), `reason` (text).
- Drizzle schema + generador de migraciones; archivo SQLite local (p. ej. `packages/db/data/ff.sqlite` o `apps/api/data/ff.sqlite` — path en env `DATABASE_URL=file:...`).
- Scripts: `db:generate`, `db:migrate`, `db:seed`.
- Seed ≥ 2 flags de ejemplo con defaults por ambiente, al menos un override de empresa y un rollout parcial; una flag `active`, otra `draft` o `archived`.
- Al crear flags en seed, `review_at` = `created_at + 90 days` (RF-06).
- Tests Vitest: migrar sobre DB temporal, insertar/leer una flag, verificar unique de `key`.

### Out of scope
- Endpoints HTTP CRUD (spec 04).
- Lógica del evaluador (spec 09).
- UI.
- Auth.
- Multi-instancia / réplicas (RNF-01: single instance).

## Tareas en orden

1. Añadir dependencias en `packages/db`: `drizzle-orm`, `@libsql/client`, `drizzle-kit`, uuid helper.
2. Definir schema TypeScript en `packages/db/src/schema/*.ts` y exportar desde el entry.
3. Configurar `drizzle.config.ts` con dialect sqlite/libsql y `DATABASE_URL`.
4. Generar migración inicial SQL y documentar comando `pnpm --filter @ff/db db:migrate`.
5. Implementar `createClient()` / `getDb()` que lea `DATABASE_URL` (default file local).
6. Implementar `seed.ts`: limpia o upsert idempotente; inserta flags de demo + reglas + (opcional) 1–2 filas de audit de ejemplo.
7. Exportar tipos inferidos de Drizzle útiles para api/domain.
8. Tests: DB en archivo temp o `:memory:` si libSQL lo permite; assert schema + seed mínimo.
9. Actualizar `.env.example` con `DATABASE_URL`.

## Criterios de aceptación verificables

1. `pnpm --filter @ff/db db:migrate` crea/actualiza el archivo SQLite sin error.
2. `pnpm --filter @ff/db db:seed` inserta ≥ 2 flags; re-ejecutar seed no rompe (idempotente o documentado “reset”).
3. Tras seed, consultar DB muestra defaults para `dev`/`staging`/`prod` en al menos una flag.
4. Al menos un `tenant_overrides` y un rollout con percentage entre 0 y 100 en seed.
5. Toda flag seeded tiene `review_at` ≈ created + 90 días.
6. Test Vitest de `packages/db` pasa: unique constraint en `key` o validación equivalentemente comprobada.
7. Reiniciar el proceso Node y reabrir el mismo `DATABASE_URL` sigue mostrando los datos (RF-19 / MVP #9 a nivel persistencia).

## Notas técnicas

- **libSQL:** usar `createClient({ url: process.env.DATABASE_URL })` con `file:./data/ff.sqlite`.
- Booleans en SQLite: integer 0/1 vía Drizzle `integer({ mode: "boolean" })`.
- No implementar aún escritura de audit desde API; solo tabla + seed opcional.
- Cascadas: `ON DELETE CASCADE` de reglas hijas al borrar flag (aunque delete flag puede no exponerse en MVP).
- El formato de `key` (`area.feature_name`) se valida en domain/API (specs 04/07); a nivel DB solo `UNIQUE`.
- Single-writer: documentar en README del package que una sola instancia de api debe abrir el archivo.
