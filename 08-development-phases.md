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
- Authenticated CRM screens (`/leads`, `/leads/:id`, `/pipeline`)

Depends on users, services, and the role model.

## Phase 4 — Customers and lead conversion

- Customer records
- Explicit Lead → Customer conversion
- Customer notes and activities
- Authenticated screens (`/customers`, `/customers/:id`)

Depends on Phase 3 leads and pipeline stages.

## Phase 5A — Appointments and scheduling (current)

- Appointment CRUD against customers
- Service-duration end times
- Staff availability and overlap validation
- Status lifecycle
- Authenticated screens (`/appointments`, `/appointments/:id`)
- Customer detail appointment history

Depends on customers, services, and staff availability.

A full calendar UI, reminders, recurrence, and public booking remain later work.

## Phase 5 — Public lead form (still deferred)

- Unauthenticated lead capture
- Default stage assignment
- Validation and abuse protection

Depends on Phase 3. Originally numbered Phase 5; appointments were implemented first as Phase 5A.

## Phase 6 — Calendar visualization

- Internal day/week schedule view
- Richer slot presentation

Depends on Phase 5A appointments.

## Phase 7 — Dashboard

- Lead and appointment operational metrics

Depends on a stable data model from Phases 3–6.

## Phase 8 — Integrations (explicit request only)

- Email
- n8n
- Google Calendar
- Webhooks

Out of scope until requested.
