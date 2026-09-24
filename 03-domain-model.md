# Domain model

This is the intended V1 domain. Phase 5A implements `User`, `Service`, `StaffAvailability`, `Pipeline`, `PipelineStage`, `Lead`, `Customer`, `Appointment`, `Activity`, and `Note`.

## Entities

| Entity | Purpose |
| --- | --- |
| User | Authenticated operator with a role |
| Lead | Potential customer |
| Customer | Converted customer |
| Service | Bookable offering |
| StaffAvailability | Weekly working hours for a staff user |
| Appointment | Scheduled service occurrence |
| Pipeline | Named sales process |
| PipelineStage | Ordered stage inside a pipeline |
| Activity | Timeline event for CRM history |
| Note | Staff-authored comment on a record |

## Relationships

```
User 1—* Lead (created)
User 1—* Lead (assigned)
User 1—* Appointment (staff)
User 1—* Activity
User 1—* Note
User 1—* StaffAvailability

Pipeline 1—* PipelineStage
PipelineStage 1—* Lead

Customer 1—* Lead
Lead belongsTo Customer (nullable)
Lead 1—* Activity
Lead 1—* Note

Customer 1—* Appointment
Customer 1—* Note
Customer 1—* Activity

Service 1—* Appointment
Appointment 1—* Activity

Appointment belongs to Customer, Service, User (staff)
Appointments do not belong to unconverted leads
```

## Lead

Expected fields:

- name, email, phone
- service (relationship to `Service`, not a hardcoded label)
- source
- message
- assigned staff
- pipeline stage
- timestamps

Lead sources:

`website`, `facebook`, `instagram`, `referral`, `google`, `walk_in`, `other`

## Default pipeline stages

Conversion path:

1. New
2. Contacted
3. Qualified
4. Appointment Booked
5. Appointment Completed
6. Converted

Non-conversion stages:

- Not Interested
- Lost
- No Response

Pipeline stages are database records. The SPA must not hard-code stage behavior.

## Customer

Expected fields:

- name
- email nullable
- phone nullable
- source (same controlled `LeadSource` values as leads)
- timestamps

Rules:

- `name` is required
- at least one of `email` or `phone` is required
- customers are not hard-deleted
- staff visibility comes from related assigned leads

## Conversion

Converting a lead is an explicit business operation, not a row copy.

- Create a new customer from the lead contact fields
- Set `leads.customer_id`
- Move the lead to the `converted` stage
- Write `lead_converted` and `customer_created` activities
- Reject a second conversion of the same lead

## Appointments

Statuses:

`scheduled`, `confirmed`, `completed`, `cancelled`, `no_show`

Rescheduling updates date, time, staff, or service. It is not a separate status.

Rules:

- Appointment belongs to an existing customer, not an unconverted lead
- Service must be active for new bookings
- Assigned staff must be an active staff or manager user
- Same staff member cannot have overlapping occupying appointments
- Adjacent intervals are allowed
- End time is derived from start time plus `Service.duration_minutes`
- The interval must fit an active availability window for that weekday
- New bookings and reschedules cannot start in the past

## Notes and activities

Notes are authored comments.

Activities are system-traceable events such as:

- Lead created
- Lead assigned
- Pipeline stage changed
- Note added
- Appointment created / rescheduled / cancelled / completed
- Lead converted

## Open modeling questions

Resolved in Phase 3:

1. V1 uses one default pipeline. Additional pipelines can be added later without rewriting stages.
2. Notes and activities are polymorphic (`noteable`, `activityable`) with morph map aliases of `lead`, `customer`, and `appointment`.
3. A lead can exist without a selected service.

Resolved in Phase 4:

4. Customer is a first-class entity, not a lead status.
5. Conversion is transactional and idempotent. A lead cannot be converted twice.
6. Staff customer access is derived from assigned related leads.

Resolved in Phase 5A:

7. Appointments attach only to customers (`Lead → Customer → Appointments`).
8. Overlap uses interval comparison; cancelled appointments do not occupy a slot.
9. Status transitions are centralized on `AppointmentStatus`.

Still deferred:

- Duplicate customer matching and merge
- Public booking and a full calendar UI

## Services

Fields:

- name
- description nullable
- duration_minutes, positive integer
- is_active
- timestamps

Services are deactivated rather than deleted so later appointment history can keep a stable service reference.

## Staff availability

Fields:

- user_id
- day_of_week: `monday` … `sunday`
- start_time / end_time as `time`
- is_active
- timestamps

Rules:

- Only active `staff` and `manager` users can have availability
- `start_time` must be earlier than `end_time`
- Active records for the same user and day cannot overlap
- Adjacent ranges such as 09:00–12:00 and 12:00–17:00 are allowed
- Inactive records are ignored by overlap checks and by `?is_active=1` working-hour queries
