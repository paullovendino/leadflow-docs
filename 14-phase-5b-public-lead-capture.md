# Phase 5B — Public landing page and lead capture

Phase 5B adds the public entry point that was missing from the CRM flow:

```
Visitor → Public landing page → Lead capture form → LeadFlow API → Lead → CRM pipeline
```

Public visitors can request a service. Staff review the resulting lead inside the authenticated CRM. This phase does **not** book appointments, expose a calendar, or send email/SMS.

## Routing

| Audience | Route | Auth |
| --- | --- | --- |
| Unauthenticated visitor | `/` | Public |
| Staff sign-in | `/admin` | Guest only |
| Authenticated operator | `/admin/dashboard` and other `/admin/*` CRM routes | Sanctum session |

`/` is the public landing page. The public navigation does not expose a staff login link. Authenticated users can still open `/` to preview the marketing site. Guest login success lands on `/admin/dashboard`. Logout returns to `/admin`.

The authenticated layout lives under `/admin` (`/admin/leads`, `/admin/customers`, `/admin/appointments`, …). The public route is a distinct page — no sidebar, topbar, or dashboard chrome.

## Public API

Unauthenticated endpoints under `/api/v1/public`:

| Method | Path | Auth | Purpose |
| --- | --- | --- | --- |
| GET | `/api/v1/public/services` | None | Active services, public-safe fields only |
| POST | `/api/v1/public/leads` | None, `throttle:public-leads` | Create a normal `Lead` |

CSRF still applies because Sanctum marks the API as stateful. The SPA calls `GET /sanctum/csrf-cookie` before submitting.

Existing CRM endpoints are unchanged. `POST /api/v1/leads` remains Sanctum-protected.

## Lead creation flow

`PublicLeadController` validates with `StorePublicLeadRequest`, then calls `LeadService::createPublicLead()`.

That method is separate from `createLead()` so public submissions cannot inherit staff auto-assignment or accept client-chosen source, stage, or assignee.

Server-controlled values:

| Field | Public behavior |
| --- | --- |
| `source` | Always `website` (`LeadSource::Website`) |
| `pipeline_stage_id` | Default pipeline stage with slug `new` |
| `assigned_user_id` | Always `null` |
| `customer_id` | Not set |
| Activity | `lead_created` — “Lead submitted from the website” |

The lead is a normal `leads` row. It appears in `/admin/leads` for administrators and managers. Staff see it after assignment.

## Accepted payload

```json
{
  "name": "John Customer",
  "email": "john@example.com",
  "phone": "09171234567",
  "service_id": 1,
  "message": "I'd like to know more about this service."
}
```

Allowed fields: `name`, `email`, `phone`, `service_id`, `message`.

The honeypot field `company` is accepted only so it can be rejected. An empty honeypot is stripped before validation.

Rejected or ignored internal fields include `assigned_user_id`, `pipeline_id`, `pipeline_stage_id`, `customer_id`, `status`, `source`, notes, and activity payloads. Extra keys are not passed into `createPublicLead()`.

## Validation

| Field | Rules |
| --- | --- |
| `name` | Required, trimmed, max 255 |
| `email` | Required without phone, valid email, max 255 |
| `phone` | Required without email, max 50, digits and `+() .-` |
| `service_id` | Nullable; must exist; must be an active service |
| `message` | Nullable, max 2000 |
| `company` | Prohibited when present |

At least one of email or phone is required.

Inactive and missing services return `422` on `service_id`.

## Public response

`PublicLeadResource` returns only:

- `id`, `name`, `email`, `phone`, `source`
- `service` (`id`, `name`, `duration_minutes`) when loaded
- `created_at`

It does not return assignee, pipeline stage, notes, activities, or authorization data. The SPA uses this payload for the success state and does not re-fetch the lead.

## Public services

`GET /api/v1/public/services` exists because `GET /api/v1/services` requires an authenticated CRM session.

`PublicServiceResource` returns `id`, `name`, `description`, and `duration_minutes` for active services, ordered by name. It does not return `is_active`, audit timestamps, or user relationships.

The landing page loads only this catalog. It does not load leads, customers, appointments, activities, or staff.

## Rate limiting and abuse protection

| Control | Implementation |
| --- | --- |
| Rate limit | `public-leads`: 10 submissions per minute per IP |
| Field lengths | Name 255, phone 50, message 2000 |
| Service integrity | Active catalog only |
| Internal fields | Not in the Form Request; not copied from the payload |
| Honeypot | Hidden `company` input; backend `prohibited` |
| CSRF | Sanctum cookie required for the SPA POST |

There is no CAPTCHA in this phase.

`429` is returned when the limiter is exceeded. The landing form shows a visitor-safe message and does not expose Laravel exception text.

## Activity

`ActivityService::record()` accepts a nullable actor. Website submissions store `user_id = null` with metadata `{ "source": "website" }`. No IP address or request headers are stored.

## Frontend UX

`LandingView` is a marketing page, not a CRM screen:

1. Sticky public navigation
2. Hero with product copy and a CSS inquiry card
3. Trust strip (no fabricated metrics)
4. Real active services (skeletons + retry)
5. Three-step “How it works”
6. Lead capture form
7. Final CTA
8. Footer with staff sign-in

The form (`PublicLeadForm`) provides:

- Visible labels, `type="email"` / `type="tel"`, autocomplete
- Client-side checks plus server field errors
- `Sending...` + disabled submit while in flight
- Preserved values on failure
- Success state with first name and “Send another request”
- `422` / `429` / network-or-500 visitor copy
- Invisible honeypot (`company`)

Copy is product-focused. No fake testimonials, ratings, logos, or statistics.

## Responsive behavior

The page is designed for:

- 390px — stacked hero, compact header + menu, stacked form fields
- 768px — two-column form fields, service grid
- 1024px+ — hero split, spacious max-width (`max-w-6xl`)

Motion stays in the 150–250ms range. `prefers-reduced-motion` removes movement.

## Security decisions

- Public clients cannot assign leads, choose stages, choose source, create customers, or create appointments.
- Public clients cannot read CRM collections.
- PHP cannot use `Public` as a namespace segment; public HTTP types live under `App\Http\...\PublicApi`.
- Tests live under `Tests\Feature\PublicCapture`.
- Authenticated CRM authorization is unchanged.

## Schema

No migrations. Public capture reuses `leads` and `activities`.

## Known limitations

- No public appointment booking or slot selection
- No email, SMS, or webhook acknowledgement
- No CAPTCHA
- The public site does not expose a staff login link; operators open `/admin` directly
- Rate limiting is IP-based only
- Honeypot is a simple hidden field, not a bot platform

## Future booking flow

A later phase can add public booking after the lead (or a converted customer) exists:

```
Public request → Lead → Staff follow-up → Customer → Appointment
```

Do not attach appointments to unconverted public leads.
