# Spec 10 — Simulator

## Objetivo

Añadir en la UI un simulador de evaluación que permita probar una flag con contexto (ambiente, empresa, user/session) y mostrar `enabled` + `reason` usando el endpoint público de evaluate (RF-17 evaluación de prueba; MVP #10).

## Contexto y dependencias

- **PRD:** RF-17 (ejecutar evaluación de prueba); MVP #10.
- **Requiere:** Specs 01–09 (detalle de flag, targeting configurable, `POST /api/evaluate`).

## Alcance

### In scope
- Sección o página “Simulator” accesible desde el detalle de flag `/flags/[id]` (y opcionalmente `/flags/[id]/simulate`).
- Formulario:
  - `environment`: select dev/staging/prod
  - `tenantId`: text opcional
  - `userId`: text opcional
  - `sessionId`: text opcional
  - `flagKey`: prellenado con la key de la flag actual (readonly)
- Botón “Evaluate” → `POST /api/evaluate` (via proxy same-origin; endpoint es público pero el uso desde UI autenticada es el flujo normal).
- Resultado visible: `enabled` (true/false destacado) y `reason` (código legible).
- Validación UX: si se espera probar %, avisar si faltan userId y sessionId (el API igual devolverá fail-safe).
- No persiste simulaciones (sin historial de sims).

### Out of scope
- Bulk simulation / CSV.
- Gráficos de distribución de rollout.
- Edición de targeting dentro del simulador (usar sección spec 08).
- Historial de auditoría (spec 11).

## Tareas en orden

1. Crear componente `FlagSimulator` en `apps/web`.
2. Integrarlo en la página de detalle de flag.
3. Cablear submit al API y render del resultado.
4. Manejar errores de red.
5. Verificación manual: escenarios MVP #3–#6 usando flags de seed/configuradas.
6. Test opcional del formatter de `reason` a label humano.

## Criterios de aceptación verificables

1. Desde el detalle de una flag `active`, el operador puede enviar ambiente + tenant + user y ver `enabled` y `reason` (RF-17).
2. Cambiar solo `environment` con defaults distintos cambia el resultado cuando aplica default (MVP #3 vía UI).
3. Con override acme, simular tenant `acme` vs otro muestra diferencia (MVP #4 vía UI).
4. Misma combinación de inputs → mismo resultado al re-evaluar (determinismo visible).
5. Flag no active: simulador muestra `enabled: false`.
6. La página del simulador/detalle sigue requiriendo login (aunque evaluate sea público).

## Notas técnicas

- Preferir llamar `/api/evaluate` (rewrite) no el endpoint de gestión.
- Mapear reasons a labels en español/inglés consistente con el resto de la UI (p. ej. `tenant_override` → “Override de empresa”).
- No reimplementar la lógica de evaluación en el cliente; solo mostrar respuesta del API.
- Mantener un solo bloque de resultado (evitar ruido visual).
