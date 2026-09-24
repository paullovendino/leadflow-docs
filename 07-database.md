# Database

## Engine

Production and local development use PostgreSQL.

The current environment has PostgreSQL 18 listening on port `5432` with superuser `postgres`.

Create a dedicated database:

```sql
CREATE DATABASE leadflow;
```

Set `DB_PASSWORD` in `leadflow-api/.env`. The installer password is not stored in this repository.

## Tests

PHPUnit uses SQLite in-memory (`phpunit.xml`). This keeps tests isolated and avoids requiring PostgreSQL credentials in CI.

PostgreSQL-only constraints must be guarded by driver checks so SQLite tests still run. The `users.role` check constraint is an example.

## Current schema

### users

- `id` bigint PK
- `name`
- `email` unique
- `email_verified_at` nullable
- `password`
- `role` default `staff`, indexed
- `is_active` default true
- `remember_token`
- timestamps
- PostgreSQL check: `role IN ('administrator', 'manager', 'staff')`

### sessions, cache, jobs, password_reset_tokens

Laravel defaults. Sessions are stored in the database for the SPA cookie flow.

### personal_access_tokens

Published by Sanctum. Unused by the SPA.

### services

- `id` bigint PK
- `name`
- `description` nullable
- `duration_minutes` unsigned integer
- `is_active` default true, indexed
- timestamps
- PostgreSQL check: `duration_minutes > 0`

### staff_availabilities

- `id` bigint PK
- `user_id` FK → users, cascade on delete
- `day_of_week`
- `start_time`, `end_time` as `time`
- `is_active` default true
- timestamps
- composite index `(user_id, day_of_week, is_active)`
- PostgreSQL checks: valid day names and `start_time < end_time`

## Integrity rules for later phases

- Foreign keys for all domain relationships
- Unique constraint preventing a second customer from the same lead
- Indexes that match list/filter query patterns (lead stage, assigned user, appointment staff + time range)
- Soft deletes only when a business record must remain queryable after removal
- No Google Calendar IDs or webhook tables until those phases exist

## Current Phase 3 schema

### pipelines

- `id` bigint PK
- `name`
- `is_default` default false, indexed
- `is_active` default true
- timestamps

### pipeline_stages

- `id` bigint PK
- `pipeline_id` FK → pipelines, cascade on delete
- `name`
- `slug`
- `position` unsigned integer
- `is_active` default true
- timestamps
- unique `(pipeline_id, slug)`
- index `(pipeline_id, position)`

### leads

- `id` bigint PK
- `name`
- `email` nullable, indexed
- `phone` nullable, indexed
- `service_id` nullable FK → services, null on delete
- `source` nullable, indexed; PostgreSQL check against `LeadSource`
- `message` nullable
- `assigned_user_id` nullable FK → users, null on delete
- `pipeline_stage_id` FK → pipeline_stages, restrict on delete
- `customer_id` nullable FK → customers, restrict on delete
- timestamps
- `created_at` indexed

Leads are not hard-deleted in Phase 3.

### customers

- `id` bigint PK
- `name`
- `email` nullable, indexed
- `phone` nullable, indexed
- `source` nullable, indexed; PostgreSQL check against `LeadSource`
- timestamps
- `created_at` indexed

Customers are not hard-deleted in Phase 4.

### appointments

- `id` bigint PK
- `customer_id` FK → customers, restrict on delete
- `service_id` FK → services, restrict on delete
- `staff_user_id` FK → users, restrict on delete
- `scheduled_date` date, indexed
- `start_time`, `end_time` as `time`
- `status` default `scheduled`, indexed
- `notes` nullable
- timestamps
- composite indexes `(staff_user_id, scheduled_date)` and `(customer_id, scheduled_date)`
- PostgreSQL check: `status IN ('scheduled', 'confirmed', 'completed', 'cancelled', 'no_show')`

Appointments are not hard-deleted in Phase 5A. Cancelled rows remain for history and no longer occupy a slot.

### activities

- `id` bigint PK
- `user_id` nullable FK → users, null on delete
- `activityable_type` / `activityable_id` (morphs, composite index)
- `type`; PostgreSQL check against `ActivityType`
- `description`
- `metadata` JSON nullable
- timestamps
- `created_at` indexed

### notes

- `id` bigint PK
- `user_id` FK → users, restrict on delete
- `noteable_type` / `noteable_id` (morphs, composite index)
- `body`
- timestamps
- `created_at` indexed

## Soft deletes

Users are deactivated with `is_active`, not soft deleted.

Leads are preserved and moved to terminal stages (`lost`, `not_interested`, `no_response`) instead of being deleted. Customers are retained because later appointments will reference them.
