# Authentication

LeadFlow uses Laravel Sanctum **cookie-based SPA authentication**.

The SPA does not use bearer tokens.

## Local hosts

- SPA: `http://localhost:5173`
- API: `http://localhost:8000`

Use the same hostname on both sides. Do not mix `localhost` and `127.0.0.1`.

## Login sequence

1. `GET /sanctum/csrf-cookie`
2. `POST /api/v1/auth/login` with email and password
3. Subsequent requests send the session cookie and `X-XSRF-TOKEN`

Axios is configured with `withCredentials` and `withXSRFToken`.

## Logout sequence

1. `GET /sanctum/csrf-cookie` if needed
2. `POST /api/v1/auth/logout`
3. Session is invalidated and the CSRF token is regenerated

## Guards and middleware

- Default guard: `web` (session)
- Protected API routes: `auth:sanctum`
- Active-account check: `active`
- Role check (for later resources): `role:manager,staff`

`Gate::before` grants administrators every gate/policy ability. Route middleware still requires authentication.

## SPA client rules

- Store the current user in Pinia for UI only
- Never treat a frontend role check as security
- On 401, clear local auth state and send the user to login

## CSRF and CORS

- CORS origins are explicit (`FRONTEND_URL` / `CORS_ALLOWED_ORIGINS`)
- `supports_credentials` is true
- `SANCTUM_STATEFUL_DOMAINS` lists SPA hosts without a protocol

## Tokens

Sanctum publishes a `personal_access_tokens` table because that is the framework default. Phase 1 does not issue API tokens and does not expose token login endpoints.

If a non-SPA client is required later, token authentication can be added without changing the SPA cookie flow.
