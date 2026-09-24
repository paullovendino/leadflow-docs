# Phase 5A — Appointments and scheduling

Phase 5A adds the first complete appointments domain. Public booking, Google Calendar, reminders, recurring series, and a full calendar UI remain future work.

The original phase plan listed a public lead form as Phase 5. That work is still deferred. This document records the appointments implementation that landed next after Phase 4.

## Entities

| Entity | Table | Purpose |
| --- | --- | --- |
| Appointment | `appointments` | Scheduled service interaction between a customer and bookable staff |
| Customer | `customers` | Required parent of every appointment |
| Service | `services` | Duration source of truth |
| User | `users` | Bookable staff/manager assigned to the appointment |

## Relationship

```
Lead → Customer → Appointments
```

Appointments attach only to customers. An unconverted lead cannot be booked. After conversion, schedule against the resulting customer.

This is intentional. A second appointment identity on leads would duplicate customer history.

## Fields

- `customer_id` FK → customers, restrict on delete
- `service_id` FK → services, restrict on delete
- `staff_user_id` FK → users, restrict on delete
- `scheduled_date` date
- `start_time` / `end_time` time
- `status` string enum
- `notes` nullable
- timestamps

Indexes: `scheduled_date`, `status`, `(staff_user_id, scheduled_date)`, `(customer_id, scheduled_date)`. Foreign keys already index the relationship columns.

## Status lifecycle

Statuses: `scheduled`, `confirmed`, `completed`, `cancelled`, `no_show`.

There is no `rescheduled` status. Rescheduling is an update of date, time, staff, or service.

Allowed transitions, centralized on `AppointmentStatus`:

```
scheduled → confirmed | cancelled | completed
confirmed → completed | cancelled | no_show
completed / cancelled / no_show → (none)
```

`cancelled → confirmed` is rejected.

## Domain rules

- Customer must exist. Staff may create only when they can view that customer.
- Service must exist and be active for new bookings.
- Assigned staff must be an active `staff` or `manager`. Administrators are not bookable.
- End time is calculated from `Service.duration_minutes`. The frontend cannot set duration.
- New appointments and reschedules cannot start in the past (application timezone).
- The interval must fit inside an active availability window for that weekday.
- Occupying appointments (all statuses except `cancelled`) cannot overlap. Adjacent intervals are allowed (`10:00–11:00` then `11:00–12:00`).
- Only `scheduled` and `confirmed` appointments can be updated or rescheduled.

## Availability and slots

`GET /api/v1/appointments/slots` returns 30-minute start times that:

1. Sit inside an active weekday window
2. Fit the selected service duration
3. Do not overlap occupying appointments

This is a simple selector, not a calendar engine. The backend still validates create/reschedule even if the UI is bypassed.

## Authorization

| Action | administrator | manager | staff |
| --- | --- | --- | --- |
| List / view | all | all | assigned as staff **or** customer they can view |
| Create | yes | yes | if they can view the customer |
| Update / reschedule | yes | yes | if they can view the appointment |
| Change status | yes | yes | if they can view the appointment |

Staff visibility matches the existing customer rule: related assigned leads. Being the appointment's `staff_user_id` also grants access.

Frontend hiding of buttons is not authorization. Policies and `AppointmentService` enforce every rule.

Staff cannot list `/api/v1/users`, so the create form only offers the current staff user in the staff dropdown. Managers and administrators load bookable colleagues from the users index.

## API

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/api/v1/appointments` | Filter and paginate |
| POST | `/api/v1/appointments` | Create (`scheduled`) |
| GET | `/api/v1/appointments/slots` | Suggested start times |
| GET | `/api/v1/appointments/{appointment}` | Detail and activities |
| PATCH | `/api/v1/appointments/{appointment}` | Reschedule / notes |
| PATCH | `/api/v1/appointments/{appointment}/status` | Status transition |

List filters: `search`, `status`, `staff_user_id`, `customer_id`, `service_id`, `date`, `date_from`, `date_to`, `per_page`.

Pagination matches leads and customers (`data` + `meta`).

Nested `/customers/{customer}/appointments` was not added. Customer detail already embeds appointments. The list endpoint covers staff/customer scoping.

## Activities

Morph map alias: `appointment`.

Types:

- `appointment_created`
- `appointment_updated`
- `appointment_confirmed`
- `appointment_completed`
- `appointment_cancelled`
- `appointment_no_show`

Create and status changes also write a customer-facing activity so the customer timeline stays readable.

Metadata is limited to `appointment_id`, `service_id`, `staff_user_id`, and status `previous_status` / `new_status` when relevant.

## Frontend

- Workspace nav: Appointments
- `/appointments` — filters, table, create drawer, in-place insert on page 1
- `/appointments/:id` — details, allowed status actions, reschedule, activity
- `/customers/:id` — upcoming and recent appointments replace the Phase 4 placeholder

Loading labels: `Creating...`, `Rescheduling...`, `Confirming...`, `Completing...`, `Cancelling...`. Failed requests leave the form open.

## Tests

`tests/Feature/Appointments/AppointmentManagementTest.php` and `tests/Unit/AppointmentStatusTest.php` cover creation, availability, overlap, adjacent slots, past dates, reschedule, status transitions, staff authorization, filters, pagination, activities, and customer embedding.

## Known limitations

- App timezone remains the existing Laravel timezone (`UTC` unless changed). Past-date checks use that clock, not a Philippines-specific timezone.
- Slots are 30-minute increments inside weekday windows. There is no week/month calendar view.
- Service and user lookups still use the existing 15-row catalog pages.
- Staff create-form staff picker is self-only because `/users` is manager/admin.
- Appointments cannot be booked against unconverted leads.
- No reminders, recurrence, public booking, or external calendar sync.

## Future work

- Public lead form (original Phase 5)
- Lightweight day/week schedule visualization
- Reminders and no-show follow-up
- Recurring appointments
- Public / anonymous booking
- Google Calendar, email, SMS, n8n
