# Phase 4 — Customers and lead conversion

Phase 4 adds `Customer` as a first-class CRM entity and an explicit Lead → Customer conversion action. Appointments, calendar, public capture, and integrations remain out of scope.

## Entities

| Entity | Table | Purpose |
| --- | --- | --- |
| Customer | `customers` | Established business contact |
| Lead | `leads` | Qualification record; may point at a customer after conversion |

## Relationship

```
Customer 1—* Lead
Lead belongsTo Customer (nullable customer_id)
```

A lead converts into one newly created customer. Later inquiries may attach additional leads to the same customer, but Phase 4 does not auto-merge duplicates.

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

Administrators and managers may `POST /api/v1/customers` for walk-ins or historical records. Staff cannot create unrelated customers; they create customers by converting assigned leads.

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

Customer list does not eager-load notes, activities, or leads.

## Frontend

- `/admin/customers` — table, search, source filter, create (managers/admins)
- `/admin/customers/:id` — edit, related leads, notes, activity
- `/admin/leads/:id` — Convert to Customer confirmation, or View customer when already converted

## Deferred

- Duplicate matching / customer merge
- Appointments and calendar
- Public lead capture
- Automation
