# PRD: Feature Flags internas (MVP)

## 1) Contexto y problema

Hoy, activar o desactivar funcionalidades por empresa, ambiente o porcentaje de tráfico requiere un deploy. Eso retrasa rollouts, complica rollbacks y aumenta el riesgo de cambios en producción.

Necesitamos una herramienta interna que permita controlar features en runtime, sin redeploy, con targeting por ambiente, empresa y porcentaje de tráfico, y con persistencia local simple.

## 2) Objetivo

Permitir a un operador (usuario demo) crear, configurar y evaluar feature flags booleanas con targeting por ambiente, empresa y rollout porcentual, persistidas en SQLite, sin necesidad de deploy para cambiar su estado.

**Éxito del MVP:** un flag se puede crear, configurar (ambiente / empresa / %) y evaluar de forma determinística vía UI o API, con el resultado correcto según las reglas definidas.

## 3) Público objetivo y usuarios

| Actor | Descripción | Capacidades en MVP |
|--------|-------------|-------------------|
| Operador (usuario demo) | Único usuario autenticado | CRUD de flags, definir targeting, consultar evaluación, ver historial básico de cambios |
| Sistema consumidor | Backend/servicio que consulta el evaluador | Evaluar si una flag está activa para un contexto dado |

No hay roles diferenciados (Product, Engineering, Support). Cualquiera con el usuario demo tiene acceso completo.

## 4) Alcance

### In scope
- Autenticación con un único usuario demo (credenciales fijas / hardcodeadas).
- Flags booleanas (`on` / `off`).
- Targeting por **ambiente** (`dev` \| `staging` \| `prod`), **empresa** (`tenant_id`) y **rollout porcentual** (0–100).
- Precedencia de evaluación: **override por empresa > porcentaje (si aplica) > default del ambiente**.
- Evaluación determinística por `user_id` o `session_id`.
- Persistencia en **SQLite** local.
- UI interna para gestionar flags y ver resultados de evaluación.
- API para CRUD y evaluación.
- Ciclo de vida básico: `draft → active → archived`, naming `area.feature_name`, owner, review/TTL a 90 días.
- Auditoría mínima: quién (usuario demo), cuándo, valor anterior/nuevo, motivo.

### Out of scope
- OAuth, SSO, RBAC, roles o permisos avanzados.
- Variantes A/B, experimentos, configs tipadas no booleanas.
- Sub-tenants / jerarquías de empresa.
- Approval workflows / gate de cambios en producción.
- SDKs multi-lenguaje maduros (más allá de lo mínimo para evaluar vía API).
- Alta disponibilidad, multi-región o sync entre instancias.
- Integraciones externas (Slack, Jira, feature stores comerciales).

## 5) Conceptos de dominio

### Feature flag
Entidad con:
- `key` única (`area.feature_name`)
- `description`, `owner`
- `status`: `draft` \| `active` \| `archived`
- `created_at` / `updated_at`
- `review_at` (default: creación + 90 días)

Una flag **active** puede evaluarse; `draft` y `archived` no se consideran activas en runtime (evaluación = `false`, salvo que se documente lo contrario de forma explícita en implementación: **MVP: solo `active` evaluable; resto → `false`**).

### Targeting rule
Conjunto de reglas asociadas a una flag que definen el resultado en un contexto. Incluye:
- **Default por ambiente:** valor booleano base en `dev` / `staging` / `prod`.
- **Override por empresa:** lista de `tenant_id` → `true` \| `false` (gana sobre % y default).
- **Rollout porcentual:** entero 0–100 aplicado cuando no hay override de empresa; usa hash determinístico de `user_id` o `session_id`.

### Evaluador
Componente que, dado:
- `flag_key`
- `environment`
- `tenant_id` (opcional)
- `user_id` o `session_id` (requerido si hay %)

devuelve `{ enabled: boolean, reason: string }` aplicando la precedencia:
1. Si existe override para `tenant_id` → ese valor.
2. Else si hay rollout % configurado → `enabled` si `bucket(user_id|session_id) < percentage`.
3. Else → default del ambiente.
4. Si la flag no existe o no está `active` → `enabled: false`.

## 6) Requerimientos funcionales

**RF-01.** El sistema debe autenticar un único usuario demo con credenciales configuradas; sin credenciales válidas no se accede a UI ni a endpoints de gestión.

**RF-02.** El operador debe poder crear una flag con `key` única en formato `area.feature_name`, `description`, `owner` y `status` inicial `draft`.

**RF-03.** El sistema debe rechazar la creación si `key` ya existe o no cumple el formato `area.feature_name`.

**RF-04.** El operador debe poder editar `description`, `owner`, defaults por ambiente, overrides por empresa y porcentaje de rollout (0–100).

**RF-05.** El operador debe poder cambiar el `status` entre `draft`, `active` y `archived`.

**RF-06.** Al crear una flag, el sistema debe fijar `review_at` = fecha de creación + 90 días.

**RF-07.** El operador debe poder definir, por flag, un valor default booleano independiente para `dev`, `staging` y `prod`.

**RF-08.** El operador debe poder agregar, editar y eliminar overrides por `tenant_id` (empresa) con valor booleano.

**RF-09.** El operador debe poder configurar un rollout porcentual entero entre 0 y 100 inclusive (o desactivarlo explícitamente).

**RF-10.** El evaluador debe aplicar la precedencia: override empresa > porcentaje > default de ambiente.

**RF-11.** Dado el mismo `flag_key`, `environment`, `tenant_id` y `user_id`/`session_id`, el evaluador debe devolver siempre el mismo resultado (determinismo).

**RF-12.** Si la flag no existe o su `status` ≠ `active`, el evaluador debe devolver `enabled: false`.

**RF-13.** Si hay override para el `tenant_id` del contexto, el evaluador debe ignorar el porcentaje y el default de ambiente.

**RF-14.** Si no hay override y hay porcentaje P, el evaluador debe asignar un bucket estable 0–99 (o 0–100 según implementación documentada) a partir de `user_id` o `session_id` y habilitar solo si el bucket está dentro del porcentaje.

**RF-15.** Si no hay override ni porcentaje activo, el evaluador debe usar el default del `environment` solicitado.

**RF-16.** La API debe exponer endpoints para: listar flags, obtener flag, crear, actualizar, cambiar status, y evaluar (`flag_key` + contexto).

**RF-17.** La UI debe permitir listar flags, crear/editar targeting y ejecutar una evaluación de prueba con contexto (ambiente, empresa, user/session).

**RF-18.** Cada cambio de configuración o status debe registrarse en auditoría con: actor (usuario demo), timestamp, campo/entidad afectada, valor anterior, valor nuevo y motivo (texto; obligatorio en cambios que alteren evaluación en runtime).

**RF-19.** Todos los datos de flags, reglas y auditoría deben persistirse en SQLite local y sobrevivir reinicios del proceso.

**RF-20.** El operador debe poder consultar el historial de auditoría de una flag.

## 7) Requerimientos no funcionales

**RNF-01.** Persistencia: SQLite en filesystem local; una sola instancia de la app es el caso soportado.

**RNF-02.** Latencia de evaluación (mismo proceso / API local): p95 < 50 ms para el volumen esperado del MVP (< 1k flags, < 10k overrides).

**RNF-03.** Disponibilidad: best-effort local; no se exige HA ni réplicas.

**RNF-04.** Fail-safe: ante error de lectura/evaluación no recuperable, el consumidor debe tratar el resultado como `enabled: false` (default seguro).

**RNF-05.** Consistencia: tras un cambio guardado, evaluaciones posteriores en la misma instancia reflejan el nuevo valor de forma inmediata (sin “eventual consistency” entre nodos; no hay multi-nodo en MVP).

**RNF-06.** Seguridad: credenciales del usuario demo no se exponen en respuestas de API; no hay OAuth/RBAC.

**RNF-07.** Observabilidad mínima: logs de errores de evaluación y de fallos de persistencia.

## 8) Criterios de aceptación del MVP

1. Login solo con usuario demo; credenciales inválidas bloquean acceso a gestión.
2. Se puede crear una flag `billing.new_checkout` en `draft`, activarla y archivarla.
3. Se configuran defaults distintos por `dev` / `staging` / `prod` y la evaluación respeta el ambiente enviado.
4. Con override `tenant_id=acme → true` y default `false`, evaluación para Acme es `true`; para otra empresa sin override sigue default/% .
5. Con 50% de rollout, sin override: ~50% de `user_id` distintos quedan enabled; el mismo `user_id` es estable entre llamadas.
6. Override de empresa gana sobre el porcentaje.
7. Flag `draft` o `archived` (o inexistente) → `enabled: false`.
8. Cambios quedan en auditoría consultable (anterior/nuevo + motivo).
9. Tras reiniciar la app, flags y reglas siguen disponibles desde SQLite.
10. UI permite crear/editar y probar evaluación; API permite lo mismo de forma programática.

## 9) Riesgos y supuestos

### Supuestos
- Un solo operador/proceso; no hay despliegue multi-instancia concurrente sobre el mismo SQLite.
- Los consumidores pasan `environment`, y `user_id` o `session_id` cuando usan %.
- `tenant_id` es un string estable conocido por el negocio.
- El usuario demo es aceptable porque la herramienta es interna y de alcance limitado.
- “Sin deploy” se refiere a no redeployar la app de producto; sí puede reiniciarse el servicio de flags si hace falta (no es requisito de zero-downtime).

### Riesgos
| Riesgo | Impacto | Mitigación |
|--------|---------|------------|
| SQLite + múltiples writers | Corrupción / locks | Documentar single-instance; una sola app escribe |
| Usuario demo compartido | Sin accountability real | Auditoría con actor fijo; asumir uso confiado en MVP |
| Flags sin limpieza | Deuda operativa | `review_at` 90 días + estado `archived` |
| Hash/% mal implementado | Rollouts no deterministas o sesgados | Test de estabilidad + distribución en criterios de aceptación |
| Default fail-open accidental | Features activas por error | Fail-safe = `false`; solo `active` evaluable |
