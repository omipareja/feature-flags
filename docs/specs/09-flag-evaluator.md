# Spec 09 — Flag evaluator

## Objetivo

Implementar el evaluador determinístico de flags (dominio puro + endpoint público) con precedencia **override empresa > porcentaje > default de ambiente**, fail-safe `false`, y tests exhaustivos (RF-10…15, RF-16 evaluate, RNF-02/04/05).

## Contexto y dependencias

- **PRD:** §5 Evaluador; RF-10…15; RF-16 evaluate; RNF-02, RNF-04, RNF-05; MVP #3–#7.
- **Requiere:** Specs 01–04 y 08 (datos de targeting en DB). Spec 05 auth: añadir `/api/evaluate` a allowlist **pública**.
- **Paquete principal:** `packages/domain` (función pura); `apps/api` carga flag y llama al dominio.

## Alcance

### In scope
- Tipos de contexto:
  ```ts
  type EvaluateInput = {
    flagKey: string
    environment: "dev" | "staging" | "prod"
    tenantId?: string
    userId?: string
    sessionId?: string
  }
  type EvaluateResult = { enabled: boolean; reason: string }
  ```
- Función `evaluateFlag(flagSnapshot, input): EvaluateResult` en `packages/domain`:
  1. Si flag missing o `status !== "active"` → `{ enabled: false, reason: "flag_inactive_or_missing" }` (RF-12).
  2. Si `tenantId` tiene override → ese boolean; reason `tenant_override` (RF-13).
  3. Else si rollout activo (percentage number 0–100) → bucket; requiere `userId` o `sessionId`; si faltan ambos → `enabled: false`, reason `missing_subject_for_rollout` (fail-safe).
  4. Bucket: hash estable del subject string → entero `0..99`; `enabled = bucket < percentage` (RF-14). Documentar algoritmo (p. ej. FNV-1a o primeros bytes hex de SHA-256).
  5. Else default del `environment`; si environment sin default → `false` (RF-15).
- Endpoint público:
  - `POST /api/evaluate` body = EvaluateInput → `{ data: EvaluateResult }`.
  - No requiere cookie (consumidor).
  - Ante error de DB inesperado: responder `enabled: false` + reason `error` o 500 con fail-safe documentado; preferir **200 + enabled false** para fail-safe del consumidor (RNF-04) **o** 500 y documentar que el SDK trata error como false — **elegir 200 `{ enabled: false, reason: "evaluation_error" }`** en errores recuperables de lectura.
- Middleware auth: allowlist `/api/evaluate`.
- Tests unitarios domain: matriz de casos MVP #3–#7 + determinismo mismo input (RF-11).
- Test de distribución aproximada: 1000 userIds distintos con 50% → enabled count en rango razonable (p. ej. 400–600) opcional pero recomendado.
- Carga desde DB inmediata tras cambios (RNF-05): evaluate lee estado actual, sin cache stale en MVP (cache opcional desactivada).

### Out of scope
- UI simulador (spec 10).
- SDKs multi-lenguaje.
- Cache distribuida / TTL (no necesaria).
- Latencia p95 medida en prod; test de performance no obligatorio si la función es O(1) en memoria — cumplir RNF-02 por diseño (evaluación in-process).

## Tareas en orden

1. Implementar `hashToBucket(subject: string): number` (0–99) + tests de estabilidad.
2. Implementar `evaluateFlag` con todas las ramas y reasons estables (strings constantes).
3. En API: cargar flag por key + targeting; mapear a snapshot; llamar domain.
4. Exponer `POST /api/evaluate` público.
5. Actualizar auth middleware allowlist.
6. Tests domain + un test integración API con DB seed/fixture.
7. Exportar tipos desde `@ff/domain` para web (spec 10).

## Criterios de aceptación verificables

1. Flag `draft`/`archived`/inexistente → `enabled: false` (RF-12; MVP #7).
2. Defaults distintos por ambiente: mismo tenant/user, cambia `environment` → respeta default (RF-15; MVP #3).
3. Override `acme=true` con default false → acme true; otro tenant sin override sigue default/% (RF-13; MVP #4).
4. Con 50% y sin override, mismo `userId` siempre igual resultado (RF-11; MVP #5 estabilidad).
5. Override gana sobre percentage (RF-10, RF-13; MVP #6).
6. `POST /api/evaluate` sin sesión retorna 200 con evaluación (no 401).
7. Suite Vitest domain cubre RF-10…15 y pasa en `pnpm --filter @ff/domain test`.

## Notas técnicas

- Subject para hash: `userId ?? sessionId` (preferir userId si ambos).
- Percentage `0` → nadie; `100` → todos (`bucket < 100` siempre).
- No usar `Math.random`.
- Reason strings: usar union type documentada para el simulador.
- Snapshot de flag pasado al domain no debe incluir secretos.
