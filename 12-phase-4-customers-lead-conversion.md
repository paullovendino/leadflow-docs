# Phase 4 — Customers and lead conversion

Phase 4 adds `Customer` as a first-class CRM entity and an explicit Lead → Customer conversion action. Appointments, calendar, public capture, and integrations remain out of scope.

## Entities

| Entity | Table | Purpose |
| --- | --- | --- |
| Customer | `customers` | Established business contact |
| Lead | `leads` | Qualification record; may point at a customer after qualification or conversion |

## Relationship

```
Customer 1—* Lead
Lead belongsTo Customer (nullable customer_id)
```

A lead converts into one newly created customer. Phase 5C qualification can instead link a contacted lead to an existing customer when email or phone matches. Convert still always creates a new customer and moves the lead to `converted`.

## Conversion

`POST /api/v1/leads/{lead}/convert`

Runs in a single transaction:

1. Reject if the lead is already converted
2. Create a customer from the lead name, email, phone, and source
3. Set `leads.customer_id`
4. Move the lead to the active `converted` stage
5. Write `lead_converted` on the lead (`metadata.customer_id`)
6. Write `customer_created` on the customer (`metadata.lead_id`)

The user does not have to pre-move the lead to Converted. Conversion is the authoritative business action.

A second convert request returns `422` and does not create another customer.

## Direct creation

Administrators and managers may `POST /api/v1/customers` for walk-ins or historical records. Staff cannot create unrelated customers; they create or link customers by converting or qualifying assigned leads. A walk-in Customer does not require an originating Lead in order to book an appointment (Phase 6).

## Contact rules

Customers require `name` and at least one of `email` or `phone`. Source uses the existing `LeadSource` enum.

## Authorization

| Action | administrator | manager | staff |
| --- | --- | --- | --- |
| List / view customers | all | all | customers linked to assigned leads |
| Create customer directly | yes | yes | no |
| Update customer | yes | yes | if they can view it |
| Convert lead | any lead | any lead | assigned lead only |
| Notes / activities | yes | yes | if they can view the customer |

Staff visibility is derived from related leads. There is no `customer_owner_id`.

## API

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/api/v1/customers` | Search, filter by source, paginate |
| POST | `/api/v1/customers` | Direct create |
| GET | `/api/v1/customers/{customer}` | Detail, related leads, notes, activities, appointments |
| PATCH | `/api/v1/customers/{customer}` | Update contact fields |
| GET / POST | `/api/v1/customers/{customer}/notes` | Polymorphic notes |
| GET | `/api/v1/customers/{customer}/activities` | Timeline, newest first |
| POST | `/api/v1/leads/{lead}/convert` | Convert lead to customer |
| POST | `/api/v1/leads/{lead}/qualify` | Qualify contacted lead (Phase 5C) |

Customer list does not eager-load notes, activities, or leads.

## Frontend

- `/admin/customers` — table, search, source filter, create (managers/admins). The create action is **New customer**.
- `/admin/customers/:id` — edit, related leads, notes, activity. Phase 6 adds **Book appointment** plus **Upcoming** / **Past** lists. See [16-phase-6-appointment-calendar-workflow.md](16-phase-6-appointment-calendar-workflow.md).
- `/admin/leads/:id` — Convert to Customer confirmation, or View customer when already converted. Contacted leads use **Move to Qualified** (Phase 5C).

## Deferred

- Customer merge UI
- Automation

Appointments, public capture, and qualification matching are documented in later phase files.
