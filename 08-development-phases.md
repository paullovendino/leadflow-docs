# Development phases

Implement one phase at a time. Do not expand scope silently.

Before each phase:

1. Inspect the repository
2. Follow existing conventions
3. Identify dependencies
4. Reuse existing code
5. Implement only the requested phase
6. Run tests
7. Fix failures
8. Review
9. Update documentation

## Phase 1 — Foundation

- Documentation structure
- Laravel API skeleton
- Vue SPA skeleton
- PostgreSQL configuration
- Sanctum cookie authentication
- User roles and active flag
- Consistent API conventions
- Authentication and authorization tests

## Phase 2 — Identity and catalog

- Staff user management
- Services
- Staff availability

Depends on Phase 1 users and roles.

## Phase 3 — Pipeline and leads

- Pipelines and pipeline stages
- Lead records
- Assignment and stage movement
- Notes and activity history
- Authenticated CRM screens (`/admin/leads`, `/admin/leads/:id`, `/admin/pipeline`)

Depends on users, services, and the role model.

## Phase 4 — Customers and lead conversion

- Customer records
- Explicit Lead → Customer conversion
- Customer notes and activities
- Authenticated screens (`/admin/customers`, `/admin/customers/:id`)

Depends on Phase 3 leads and pipeline stages.

## Phase 5A — Appointments and scheduling

- Appointment CRUD against customers
- Service-duration end times
- Staff availability and overlap validation
- Status lifecycle
- Authenticated screens (`/admin/appointments`, `/admin/appointments/:id`)
- Customer detail appointment history

Depends on customers, services, and staff availability.

A full calendar UI landed in [Phase 8](18-phase-8-calendar-scheduling-ux.md). Reminders, recurrence, and public booking remain later work.

## Phase 5B — Public landing page and lead capture

- Unauthenticated landing page at `/`
- `GET /api/v1/public/services` and `POST /api/v1/public/leads`
- Server-controlled `website` source, New stage, unassigned lead
- Validation, honeypot, and `public-leads` rate limiting

Depends on Phase 3. Originally numbered Phase 5; appointments were implemented first as Phase 5A. Public booking remains later work.

## Phase 5C — Lead qualification and customer linking

- Confirmation modal before Contacted → Qualified
- `POST /api/v1/leads/{lead}/qualify` finds or creates a customer in one transaction
- Email/phone matching with conflict rejection
- Lead detail shows the linked customer and Book appointment
- Convert and appointment booking remain separate actions

Depends on Phase 3 leads and Phase 4 customers.

## Phase 6 — Appointment and calendar workflow

- Customer preselected when booking from a qualified lead or customer detail
- Upcoming / past appointments on customer detail
- Transaction lock around same-staff same-day overlap checks
- Clearer booking validation messages
- Manual customers can be booked without a lead
- No new appointment endpoints or migrations
- `AppointmentWorkflowTest.php` (7 cases); suite reported 166 passed

Depends on Phase 5A appointments and Phase 5C qualification. Qualification still does not create an appointment. A day/week calendar view landed in [Phase 8](18-phase-8-calendar-scheduling-ux.md). Google Calendar, reminders, email/SMS, n8n, and public booking remain later work.

## Phase 7 — Dashboard and operational metrics

- Dedicated `GET /api/v1/dashboard` aggregate
- Overview: total / currently qualified / currently converted leads, customers
- Appointment today, upcoming, and status counts
- Dynamic pipeline and lead-source bars
- Recent leads and recent activity (server-scoped)
- Application timezone from `APP_TIMEZONE` (`Asia/Manila`)

Depends on Phases 3–6 data. Staff metrics stay scoped.

## Phase 8 — Calendar and scheduling UX (current)

- List | Calendar toggle on `/admin/appointments`
- Day and week grids over `GET /api/v1/appointments` (no calendar endpoint)
- Desktop week / mobile day
- Booking from an empty time uses the existing drawer and slot API
- Past slot starts for today are omitted in `availableSlots()`
- No drag/drop, month view, or calendar library

Depends on Phase 6 appointment workflow. Google Calendar, n8n, email/SMS, public booking, and API performance work remain later.

## Phase 9 — n8n automation (explicit request only)

Out of scope until requested.

## Phase 10 — Email and notifications (explicit request only)

Out of scope until requested.

## Phase 11 — Google Calendar (explicit request only)

Out of scope until requested.

## Phase 12 — Public booking (explicit request only)

Out of scope until requested.

## Phase 13 — Performance and API optimization

Out of scope until requested. Includes Pipeline TTFB and other serving-layer work.

## Phase 14 — Final QA / production hardening

Out of scope until requested.
