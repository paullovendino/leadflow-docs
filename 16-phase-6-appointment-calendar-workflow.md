# Phase 6 — Appointment and calendar workflow

Phase 6 refined the existing appointment system from [Phase 5A](13-phase-5-appointments.md). It did **not** rebuild appointments, add a day or week calendar view, or create appointments during Lead qualification.

The primary improvement was making **Customer → Book appointment** reliable. The appointment form preserves the selected Customer when opened from a qualified Lead or Customer Detail.

Qualification still never creates an appointment. Booking remains an explicit user action.

## Workflows

Lead-originated:

```
Lead → Contacted → Confirm qualification → Qualified → Customer
        ↓
Book appointment
        ↓
/admin/appointments?customer_id={id}&create=1
        ↓
Customer already selected in the form
        ↓
Service → Staff → Date → Available time → Confirm
        ↓
Appointment created (`scheduled`)
```

Manual Customer (no originating Lead required):

```
Customers → New customer → Customer Detail → Book appointment → Appointment
```

Appointments belong to Customers. There is no `appointments.lead_id`.

## Qualification vs conversion vs booking

These are three separate actions.

| Action | Endpoint | Result |
| --- | --- | --- |
| Qualify | `POST /api/v1/leads/{lead}/qualify` | Lead → `qualified`, Customer created or linked. **No appointment.** |
| Convert | `POST /api/v1/leads/{lead}/convert` | Lead → `converted`, new Customer. **No appointment.** |
| Book | `POST /api/v1/appointments` | Appointment on an existing Customer |

## Booking from the CRM

The create drawer opens with:

```
/admin/appointments?customer_id={id}&create=1
```

| Origin | Form customer field |
| --- | --- |
| Qualified lead **Book appointment** | Locked to that Customer (shown by name) |
| Customer detail **Book appointment** | Same |
| Appointments **New appointment** | Customer dropdown |

The lock is **form context only**. It does not make the Customer immutable in the database. Copy:

```
This customer is already selected.
```

If the Customer is not already in the first page of the customer list, the drawer loads it with `getCustomer`.

## Frontend behavior

`AppointmentFormDrawer.vue`:

- Submit label: **Book appointment** / **Booking...** (reschedule still uses Reschedule / Rescheduling...)
- Slots: **Loading available times...**
- Stale slots are cleared when service, staff, or date changes
- Field-level validation errors are shown (including overlap, missing Customer, inactive Service)

`AppointmentsView.vue`:

- Watches `customer_id` and `create` query params
- Success toast: **Appointment booked successfully.**
- The appointment list updates without leaving the page

`CustomerDetailView.vue`:

- **Book appointment** opens the existing create flow
- Empty: **No appointments yet.** plus **Book appointment**
- Otherwise: **Upcoming** and **Past**

`/admin/customers` still supports **New customer**.

## API

No new endpoints. No API version change.

```
POST /api/v1/appointments
GET  /api/v1/appointments/slots
```

Existing list, detail, reschedule, and status routes from Phase 5A are unchanged.

`StoreAppointmentRequest` still requires `customer_id`, `service_id`, `staff_user_id`, `scheduled_date`, and `start_time`. Phase 6 only added clearer `exists` messages:

```
The selected customer could not be found.
This service is no longer available.
```

Inactive services are still rejected in `AppointmentService` with the same inactive-service message.

## Availability

Unchanged from Phase 5A. `GET /api/v1/appointments/slots` remains the only source of selectable times. Slots are 30-minute starts inside active weekday availability that fit the service duration and do not overlap occupying appointments. The backend still validates create and reschedule if the UI is bypassed.

## Conflict protection

Application-level overlap. There is **no** unique database constraint on appointment time slots (cancelled rows must be able to free a slot).

All statuses except `cancelled` occupy the slot.

Create and reschedule, inside a transaction:

```
Booking request
      ↓
Lock occupying rows for that staff member and date (`lockForUpdate`)
      ↓
Check existing occupying appointments
      ↓
Conflict?
   ├── Yes → 422
   └── No  → create / update appointment
```

Conflict message:

```
This time slot is no longer available. Please select another time.
```

Adjacent intervals remain allowed (`10:00–11:00` then `11:00–12:00`).

## Statuses

Unchanged: `scheduled`, `confirmed`, `completed`, `cancelled`, `no_show`.

## Authorization

Unchanged from Phase 5A.

| Actor | Book |
| --- | --- |
| Administrator | Any customer (`Gate::before`) |
| Manager | Any customer |
| Staff | Only a customer they can already view through assigned related Leads |

Staff may also view appointments assigned to them as `staff_user_id`. Changing `customer_id`, `staff_user_id`, or `appointment_id` in the request cannot bypass policy. Staff cannot book an unrelated walk-in Customer.

## Activities

Unchanged. Create writes `appointment_created` on the appointment and the Customer.

## Migrations

None. The Phase 5A appointment schema, statuses, and Customer / Service / staff relationships were sufficient. `migrate:fresh` was not used.

## Tests

`tests/Feature/Appointments/AppointmentWorkflowTest.php` — 7 cases:

- Manual Customer → Appointment (no Lead)
- Qualified Lead → Customer → Appointment
- Qualification does not create an appointment
- Duplicate slot rejected with the conflict message
- Missing Customer
- Inactive Service
- Staff cannot book an unrelated walk-in Customer

After Phase 6, `php artisan test` reported **166 passed** (159 existing + 7 workflow). Existing appointment, Lead, Customer, and auth tests were not rewritten.

There is **no** frontend test suite. No browser automation tests were added.

## Manual browser verification (manager)

Verified:

```
Qualified Lead
 ↓
Book appointment
 ↓
/admin/appointments?customer_id=7&create=1
 ↓
Customer already selected
 ↓
Select Initial Assessment
 ↓
LeadFlow Staff
 ↓
2026-09-28
 ↓
09:00
 ↓
Book (button showed Booking...)
 ↓
List updated in place (1 appointment)
```

Customer Detail then showed:

```
Upcoming
Initial Assessment
9:00 AM
Staff: LeadFlow Staff
Scheduled
```

Qualification itself did not create that appointment; booking was a later explicit action.

### What was not verified in the browser

- Duplicate 09:00 was omitted from available times in the UI. Backend rejection of a conflicting POST was verified in PHPUnit, not by submitting a hidden slot in the drawer.
- Staff unauthorized booking was verified in PHPUnit, not by logging in as staff in the browser.

## Intentionally not implemented

- Day calendar view
- Week calendar view
- Google Calendar / Google OAuth
- n8n
- Email or SMS notifications
- Appointment reminders
- Public booking
- Automated follow-ups
- New appointment statuses
- Appointment creation during Lead qualification
- New appointment endpoints or migrations

A richer day/week schedule visualization remains later work. Phase 6 uses the existing appointments table plus the create drawer.
