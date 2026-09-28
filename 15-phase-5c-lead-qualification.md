# Phase 5C — Lead qualification and customer linking

Phase 5C makes `Contacted → Qualified` an explicit human decision. Qualification finds or creates a customer, links the lead, and records activity. It does not book an appointment.

Convert (`POST /api/v1/leads/{lead}/convert`) remains unchanged: it always creates a new customer and moves the lead to `converted`.

## Confirmation

The UI must open the existing `AppModal` confirmation before calling the API. Browser `confirm()` is not used.

Canceling the modal:

- Does not call the API
- Leaves the lead `Contacted`
- Creates or modifies no customer
- Records no activity
- Creates no appointment

## Qualification

`POST /api/v1/leads/{lead}/qualify`

Runs in a single transaction:

1. Authorize with `LeadPolicy::qualify` (same as view: manager/admin any lead, staff assigned only)
2. If the lead is already `qualified` and has a `customer_id`, return the current lead without writes
3. Reject converted leads and leads that already have a customer
4. Reject leads that are not `contacted` (except a qualified lead missing a customer, which is healed)
5. Require an email or phone
6. Match existing customers by case-insensitive email and digit-normalized phone
7. Link the single match, or create a customer from the lead contact fields
8. Set `leads.customer_id` and move the lead to `qualified`
9. Write `lead_qualified` on the lead
10. Write `customer_created` on a newly created customer

Conflicting matches (email points to customer A, phone to customer B, or two customers share the same email/phone) return `422`. Nothing is written.

A generic `PATCH /api/v1/leads/{lead}/stage` to `qualified` is rejected so the confirmation + qualify action cannot be bypassed.

## Customer matching

Customers have no unique email/phone constraints. Matching is application-level:

| Lead contact | Result |
| --- | --- |
| No matching customer | Create customer, then link |
| One email match | Link that customer |
| One phone match | Link that customer |
| Same customer matches both | Link that customer |
| Two different customers | `422`, no mutation |

Email comparison uses `LOWER(trim(email))`. Phone comparison strips non-digits and, when there are more than 10 digits, uses the last 10 so `09171234567` and `+63 917 123 4567` match. Stored phone values are not rewritten.

## Activity

| Situation | Lead activity | Customer activity |
| --- | --- | --- |
| New customer | `Lead qualified and customer created` (`lead_qualified`) | `Customer created from lead qualification` (`customer_created`) |
| Existing customer | `Lead qualified and linked to customer` (`lead_qualified`) | none |
| Already qualified | none | none |

No appointment is created.

## Frontend

- `/admin/leads/:id` — Contacted leads show **Move to Qualified**. Confirming updates the lead, customer card, and activity in place.
- After qualification, **View customer** and **Book appointment** use the existing customer and appointments screens. Booking does not happen during qualify.
- Phase 6 opens booking as `/admin/appointments?customer_id={id}&create=1` with that Customer already selected. The user still completes the form.
- Pipeline stage selects to Qualified open the same confirmation modal.
- `/admin/customers` still supports **New customer** for walk-ins.

## Authorization

| Actor | Qualify |
| --- | --- |
| Administrator | Any lead (`Gate::before`) |
| Manager | Any lead |
| Staff | Assigned lead only |

Staff cannot qualify an unassigned or someone else's lead by calling the endpoint directly.

## Intentionally unchanged

- Convert endpoint and tests
- Customer CRUD and manual create
- Appointment booking (still a separate action against a Customer; Phase 6 opens the existing form with that Customer selected — see [16-phase-6-appointment-calendar-workflow.md](16-phase-6-appointment-calendar-workflow.md))
- Pipeline stage slugs
- Authentication and roles
