# API conventions

## Prefix

All application JSON endpoints live under:

```
/api/v1
```

Sanctum CSRF cookie:

```
GET /sanctum/csrf-cookie
```

Health check:

```
GET /up
```

## Layering

```
Controller → Form Request → Service → Model → API Resource
```

- Controllers: HTTP only
- Form Requests: authorization for the request shape + validation
- Services: business operations
- Models: persistence and relationships
- API Resources: response shape
- Policies: resource authorization

## Success responses

Single resource:

```json
{
  "data": {
    "id": 1
  }
}
```

Paginated collection (later phases):

```json
{
  "data": [],
  "links": {},
  "meta": {}
}
```

Action without a resource body:

```json
{
  "message": "Logged out successfully."
}
```

## Error responses

Validation (`422`):

```json
{
  "message": "The email field is required.",
  "errors": {
    "email": ["The email field is required."]
  }
}
```

Unauthenticated (`401`):

```json
{
  "message": "Unauthenticated."
}
```

Forbidden (`403`):

```json
{
  "message": "This action is unauthorized."
}
```

API routes always render JSON, including validation and authentication failures.

## Current endpoints

| Method | Path | Auth | Purpose |
| --- | --- | --- | --- |
| GET | `/api/v1/public/services` | Guest | Active services (public-safe fields) |
| POST | `/api/v1/public/leads` | Guest, `throttle:public-leads` | Create a website lead |
| POST | `/api/v1/auth/login` | Guest, throttled | Create session |
| POST | `/api/v1/auth/logout` | Sanctum + active | Destroy session |
| GET | `/api/v1/auth/user` | Sanctum + active | Current user |
| GET | `/api/v1/users` | Sanctum + active, manager/admin | List users |
| POST | `/api/v1/users` | Sanctum + active, manager/admin | Create user |
| GET | `/api/v1/users/{user}` | Sanctum + active | View user (staff: self only) |
| PATCH | `/api/v1/users/{user}` | Sanctum + active, manager/admin | Update user |
| POST | `/api/v1/users/{user}/activate` | Sanctum + active, manager/admin | Activate user |
| POST | `/api/v1/users/{user}/deactivate` | Sanctum + active, manager/admin | Deactivate user |
| GET | `/api/v1/services` | Sanctum + active | List services |
| POST | `/api/v1/services` | Sanctum + active, manager/admin | Create service |
| GET | `/api/v1/services/{service}` | Sanctum + active | View service |
| PATCH | `/api/v1/services/{service}` | Sanctum + active, manager/admin | Update service |
| POST | `/api/v1/services/{service}/activate` | Sanctum + active, manager/admin | Activate service |
| POST | `/api/v1/services/{service}/deactivate` | Sanctum + active, manager/admin | Deactivate service |
| GET | `/api/v1/users/{user}/availabilities` | Sanctum + active | List availability |
| POST | `/api/v1/users/{user}/availabilities` | Sanctum + active | Create availability |
| GET | `/api/v1/users/{user}/availabilities/{availability}` | Sanctum + active | View availability |
| PATCH | `/api/v1/users/{user}/availabilities/{availability}` | Sanctum + active | Update availability |
| POST | `/api/v1/users/{user}/availabilities/{availability}/activate` | Sanctum + active | Activate availability |
| POST | `/api/v1/users/{user}/availabilities/{availability}/deactivate` | Sanctum + active | Deactivate availability |
| GET | `/api/v1/leads` | Sanctum + active | List leads (staff: assigned only) |
| POST | `/api/v1/leads` | Sanctum + active | Create lead |
| GET | `/api/v1/leads/{lead}` | Sanctum + active | View lead |
| PATCH | `/api/v1/leads/{lead}` | Sanctum + active | Update lead |
| PATCH | `/api/v1/leads/{lead}/assignment` | Sanctum + active, manager/admin | Assign lead |
| PATCH | `/api/v1/leads/{lead}/stage` | Sanctum + active | Move pipeline stage |
| GET | `/api/v1/leads/{lead}/notes` | Sanctum + active | List notes |
| POST | `/api/v1/leads/{lead}/notes` | Sanctum + active | Add note |
| GET | `/api/v1/leads/{lead}/activities` | Sanctum + active | List activities, newest first |
| POST | `/api/v1/leads/{lead}/convert` | Sanctum + active | Convert lead to customer |
| GET | `/api/v1/customers` | Sanctum + active | List customers (staff: related only) |
| POST | `/api/v1/customers` | Sanctum + active, manager/admin | Create customer |
| GET | `/api/v1/customers/{customer}` | Sanctum + active | View customer |
| PATCH | `/api/v1/customers/{customer}` | Sanctum + active | Update customer |
| GET | `/api/v1/customers/{customer}/notes` | Sanctum + active | List notes |
| POST | `/api/v1/customers/{customer}/notes` | Sanctum + active | Add note |
| GET | `/api/v1/customers/{customer}/activities` | Sanctum + active | List activities, newest first |
| GET | `/api/v1/appointments` | Sanctum + active | List appointments (staff: scoped) |
| POST | `/api/v1/appointments` | Sanctum + active | Create appointment |
| GET | `/api/v1/appointments/slots` | Sanctum + active | Suggested available start times |
| GET | `/api/v1/appointments/{appointment}` | Sanctum + active | View appointment |
| PATCH | `/api/v1/appointments/{appointment}` | Sanctum + active | Reschedule or update notes |
| PATCH | `/api/v1/appointments/{appointment}/status` | Sanctum + active | Change status |
| GET | `/api/v1/pipeline` | Sanctum + active | Default pipeline and stages |
| PATCH | `/api/v1/pipeline/stages/{stage}` | Sanctum + active, manager/admin | Update stage name/position |
| POST | `/api/v1/pipeline/stages/{stage}/activate` | Sanctum + active, manager/admin | Activate stage |
| POST | `/api/v1/pipeline/stages/{stage}/deactivate` | Sanctum + active, manager/admin | Deactivate stage |

## Versioning

V1 is additive. Breaking response changes require a new API version prefix.

## Identifiers

V1 uses integer primary keys with backend authorization. The API must not leak records the caller cannot access.

## Rate limiting

- Login: 5 attempts per minute per email + IP (`throttle:auth`)
- Authenticated API: 60 requests per minute (`throttle:api`)
- Public lead submission: 10 requests per minute per IP (`throttle:public-leads`)
