# Architecture

## Applications

LeadFlow is a split-stack system:

```
leadflow-web  (Vue 3 SPA)
      │  cookie session + CSRF
      ▼
leadflow-api  (Laravel 13 REST API)
      │
      ▼
PostgreSQL
```

The SPA is the only first-party UI. The API is the system of record.

## Backend stack

- PHP 8.3+
- Laravel 13
- PostgreSQL
- Laravel Sanctum (SPA cookie authentication)
- Form Requests, Services, API Resources, Policies
- PHPUnit 12

API prefix: `/api/v1`

## Frontend stack

- Vue 3 Composition API
- TypeScript
- Pinia
- Vue Router
- Tailwind CSS
- Axios

The SPA authenticates with cookies. It does not store or send bearer tokens.

## Request flow

```
Controller → Form Request → Service → Model → API Resource
```

Controllers stay thin. Business rules belong in services. Validation belongs in Form Requests. Authorization belongs in policies, gates, and middleware. The frontend may hide UI, but the backend is authoritative.

## Layering rules

Prefer Laravel conventions.

Do **not** add repositories, interfaces, factories-as-architecture, or event buses unless a concrete need appears.

Do **not** put CRM, scheduling, or integration logic in controllers.

## Package layout

### API

- `app/Enums`
- `app/Http/Controllers/Api/V1`
- `app/Http/Requests/Api/V1`
- `app/Http/Resources/Api/V1`
- `app/Http/Middleware`
- `app/Services`
- `app/Models`
- `app/Policies` for User, Service, StaffAvailability, Lead, Pipeline, PipelineStage, and Customer
- `routes/api.php`

### SPA

- `src/lib/http.ts` — Axios client with credentials and CSRF
- `src/stores` — Pinia stores
- `src/views` — route-level pages
- `src/types` — shared TypeScript contracts
- `src/router` — auth-aware routing

## Extensibility

Later integrations (email, n8n, Google Calendar, webhooks) should attach to service-layer operations, not controllers. Phase 1 does not implement those integrations.
