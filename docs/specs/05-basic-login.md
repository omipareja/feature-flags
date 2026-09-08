# Spec 05 — Basic login

## Objetivo

Añadir autenticación con un único usuario demo (credenciales en env), sesión por cookie firmada, protección de UI de gestión y de endpoints de gestión de la API; sin OAuth ni RBAC (RF-01, RNF-06).

## Contexto y dependencias

- **PRD:** RF-01, RNF-06; criterio MVP #1; out of scope OAuth/roles.
- **Requiere:** Specs 01–04 (web + api + CRUD flags abierto).
- **Evaluate:** el endpoint de evaluación (spec 09) quedará **público**; esta spec solo protege gestión. Si evaluate aún no existe, documentar la excepción futura en middleware (allowlist de paths públicos: `/health`, futuro `/api/evaluate`).

## Alcance

### In scope
- Env vars (documentar en `.env.example`):
  - `DEMO_USER=demo`
  - `DEMO_PASSWORD=demo` (o similar; nunca hardcodear en responses)
  - `SESSION_SECRET=` (string largo)
- API:
  - `POST /api/auth/login` body `{ username, password }` → setea cookie httpOnly `ff_session` (firmada/HMAC o token opaco firmado); `200` `{ data: { username } }`.
  - `POST /api/auth/logout` → limpia cookie.
  - `GET /api/auth/me` → `200` con usuario si sesión válida; `401` si no.
  - Middleware: todas las rutas `/api/flags*` requieren sesión válida → si no, `401`. `/health` y `/api/auth/login` públicas.
- Web (`apps/web`):
  - Página `/login` con formulario usuario/password.
  - Tras login OK, redirect a `/` (dashboard placeholder o lista si ya existe spec 06; si no, home).
  - Layout o middleware Next que redirija a `/login` si no hay sesión (llamar `GET /api/auth/me` con credentials include, o BFF proxy).
  - Cliente fetch a API con `credentials: "include"`.
- CORS: `origin: http://localhost:3000`, `credentials: true`.
- Tests API: login OK, login fail, acceso a `GET /api/flags` sin cookie → 401, con cookie → 200.
- Actor en audit: usar username de sesión (`demo`) en mutaciones (actualizar spec 04 wiring).

### Out of scope
- OAuth, SSO, refresh tokens rotativos, roles, permisos por recurso.
- Captcha, rate limit avanzado, 2FA.
- Multi-usuario real.

## Tareas en orden

1. Definir helper de firma/verificación de sesión en `apps/api` (p. ej. HMAC-SHA256 de `username|exp` en cookie).
2. Implementar rutas `/api/auth/*` y middleware de protección.
3. Aplicar middleware a rutas de flags; mantener `/health` público.
4. Configurar CORS con credentials.
5. Construir página `/login` en Next + helper `apiClient`.
6. Gate de rutas autenticadas en web (middleware.ts o check en layout servidor/cliente).
7. Tests Vitest auth + flags 401.
8. Actualizar README con credenciales demo de `.env.example`.

## Criterios de aceptación verificables

1. Login con `DEMO_USER` / `DEMO_PASSWORD` correctos → cookie httpOnly set; `GET /api/auth/me` → 200 (RF-01).
2. Login con password incorrecto → 401; no cookie de sesión válida (MVP #1).
3. `GET /api/flags` sin cookie → 401; con sesión → 200 (RF-01).
4. Logout invalida la sesión; requests siguientes a flags → 401.
5. Respuestas de API **nunca** incluyen `DEMO_PASSWORD` ni `SESSION_SECRET` (RNF-06).
6. Desde el browser: visitar ruta protegida sin sesión redirige a `/login`; tras login se accede al área app.
7. Tests Vitest de auth pasan.

## Notas técnicas

- Cookie: `HttpOnly; Path=/; SameSite=Lax` en local; `Secure` solo en HTTPS (no forzar en localhost).
- Next.js no debe guardar el password; solo consume cookie vía dominio/puerto — si web:3000 y api:3001, la cookie debe setearse de forma usable: opciones válidas:
  1. **Proxy rewrites** de Next (`/api/*` → `http://localhost:3001/api/*`) para same-origin cookies (preferido), o
  2. Cookie Domain/`NEXT_PUBLIC_API_URL` + CORS credentials.
- Elegir **opción 1 (rewrites)** en `apps/web/next.config` para simplificar; documentarlo en la spec de implementación.
- No introducir librerías OAuth.
- Spec 09 añadirá `/api/evaluate` a la allowlist pública del middleware.
