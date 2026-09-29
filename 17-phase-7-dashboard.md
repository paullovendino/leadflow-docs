# Phase 7 — Dashboard and operational metrics

Phase 7 replaces the client-side dashboard (paginated CRM totals such as `per_page=1` + `meta.total`) with a dedicated aggregate API.

Booking still does **not** move a Lead to `appointment_booked`. Dashboard appointment counts come from the `appointments` table, independent of pipeline stage occupancy.

## Architecture

```
GET /api/v1/dashboard
        ↓
DashboardController
        ↓
DashboardService (scoped aggregate queries)
        ↓
DashboardResource
        ↓
HomeView.vue
```

The SPA route remains `/admin/dashboard` (`HomeView.vue`).

## Endpoint

```
GET /api/v1/dashboard
```

Middleware: `auth:sanctum`, `active`.

Authorization: `viewAny` on `Lead` (all authenticated CRM roles). **Visibility is applied in `DashboardService`**, matching existing list scoping:

| Role | Leads | Customers | Appointments | Activity |
| --- | --- | --- | --- | --- |
| Administrator | All | All | All | All |
| Manager | All | All | All | All |
| Staff | Assigned | Related assigned leads | `staff_user_id` or viewable customer | Same entity set |

There is no `customer_owner_id` and no Dashboard-only permission.

## Response (`data`)

- `overview.total_leads` — visible leads
- `overview.qualified_leads` — current `qualified` **stage** occupancy (not lifetime qualify history)
- `overview.converted_leads` — current `converted` **stage** occupancy (not `customer_id`)
- `overview.total_customers` — visible customers
- `appointments.today` — `scheduled_date` = today (all statuses)
- `appointments.upcoming` — start >= now, status not `completed` / `cancelled` / `no_show`
- `appointments.scheduled|confirmed|completed|cancelled|no_show` — status counts
- `pipeline[]` — active default-pipeline stages in position order, with counts
- `lead_sources[]` — `{ source, count }`; `source` may be `null`
- `appointment_breakdown.by_status|by_service|by_staff` — from visible appointments only
- `recent_leads[]` — 8 newest visible leads (compact)
- `recent_activity[]` — 10 newest visible activities (lead, customer, or appointment subject)

## Timezone

`config('app.timezone')` reads `APP_TIMEZONE`. Local `.env` and PHPUnit use `Asia/Manila`. Today / upcoming / appointment past checks share that clock. The dashboard does not use a separate timezone.

## Tests

`tests/Feature/Dashboard/DashboardMetricsTest.php` covers unauthenticated access, empty workspace, admin/manager totals, staff scoping, current-stage qualified/converted, inactive stages, null source, today/upcoming/status, Manila “today”, recent leads, activity visibility, and inactive users.

## Intentionally not implemented

- Google Calendar, n8n, email/SMS, reminders
- Public booking, day/week calendar (Phase 8 delivered the internal day/week calendar)
- Advanced reporting
- Performance / index / Pipeline TTFB work
- Automatic Lead stage change on appointment create
- Chart libraries
- Frontend test suite
