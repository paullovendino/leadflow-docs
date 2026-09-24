# Architectural decisions

Decisions below are in effect unless a later phase changes them. If a choice would reshape the domain model, stop and confirm it instead of inventing a requirement.

## ADR-001: Split API and SPA repositories in one workspace

`leadflow-api`, `leadflow-web`, and `leadflow-docs` are sibling directories. The SPA talks to Laravel only over HTTP. This matches the freelance-style delivery of a separate frontend and backend.

## ADR-002: Laravel 13 and PHPUnit

`composer create-project laravel/laravel` installed Laravel 13 with PHPUnit 12. Tests follow PHPUnit, not Pest.

## ADR-003: PostgreSQL in development, SQLite in tests

The product database is PostgreSQL. PHPUnit uses SQLite in-memory so tests do not require local PostgreSQL credentials. Driver-specific SQL is isolated behind driver checks.

## ADR-004: Cookie authentication only for the SPA

Sanctum stateful API authentication is the SPA login mechanism. No login endpoint issues bearer tokens.

## ADR-005: Integer primary keys

Laravel's default bigint keys are used. Authorization, not obfuscated IDs, prevents unauthorized record access.

## ADR-006: Roles as a PHP enum plus a string column

`App\Enums\UserRole` is the application contract. PostgreSQL also checks allowed values. Native PostgreSQL enums are avoided because they are awkward to migrate.

## ADR-007: Deactivate users instead of soft-deleting them

`is_active` is sufficient for V1 operator accounts. Soft deletes can be introduced later if audit requirements need restored user rows.

## ADR-008: Administrator gate bypass

`Gate::before` allows administrators every policy ability. Middleware still requires a logged-in active user. This matches "full system access" without spreading administrator conditionals through every policy.

## ADR-009: Keep Sanctum's personal access token table

The table is a Sanctum default and may support a future non-SPA client. Phase 1 does not expose token APIs.

## ADR-010: JSON wrapping via API Resources

Success payloads use Laravel's `data` wrap. Validation and authentication errors keep Laravel's standard `message` / `errors` shape.

## ADR-011: Forget auth guards on logout

After invalidating the session, logout calls `Auth::forgetGuards()`. This prevents a cached Sanctum/web user from surviving in long-lived application processes (HTTP tests and Laravel Octane). Each real PHP-FPM request is a new process, but forgetting guards is still the safer session teardown.

## ADR-012: Deactivate services instead of deleting them

Appointments do not exist yet, but they will reference services. Phase 2 uses `is_active` so a paused offering can be restored without breaking later historical rows.

## ADR-013: String weekday names

`day_of_week` is stored as `monday` … `sunday`. This is easy to query (`where day_of_week = 'monday'`) and readable in the API. A PHP enum is the application contract; PostgreSQL checks the allowed values.

## ADR-014: No overlapping active availability

Two active records for the same user and weekday cannot overlap. Adjacent ranges are allowed. Inactive records are excluded from the conflict check so a replaced shift can be deactivated and then recreated.

Availability is assigned only to active `staff` and `manager` users. Administrators are operators, not bookable practitioners, unless they also have a staff/manager account.

## ADR-015: Last administrator protection in the service layer

`Gate::before` grants administrators every policy ability, including self-deactivation. Preventing removal of the last active administrator is therefore enforced in `UserManagementService`, not only in `UserPolicy`.

## ADR-016: One default pipeline in V1

LeadFlow seeds a single `Default Lead Pipeline`. The `pipelines` table exists so later phases can add named pipelines without rewriting stages or leads.

## ADR-017: Database-backed pipeline stages

Stage names, slugs, and positions live in `pipeline_stages`. The SPA must load them from `GET /api/v1/pipeline` rather than hard-coding conversion paths.

## ADR-018: Leads are not hard-deleted

CRM history matters. Phase 3 has no `DELETE /leads/{lead}` endpoint. Terminal stages (`lost`, `not_interested`, `no_response`) replace deletion. Archiving can be added later.

## ADR-019: Polymorphic notes and activities

`notes` and `activities` use morph maps (`lead` today; `customer` and `appointment` later) so timeline history can attach to future records without separate comment tables.

## ADR-020: Meaningful CRM actions write activity rows

Lead create, assignment, stage movement, notes, and meaningful field updates generate activities in the same transaction as the domain change. Metadata preserves old/new stage identities independently of the description string.

## ADR-021: Optional service and assignee at capture

A lead can be captured before the caller knows the service or before staff is assigned. Staff create requests auto-assign the creator so staff can still see the lead. Administrators are never the default assignee.

## ADR-022: Staff cannot manage global pipeline configuration

Staff can move assigned leads between existing stages. Only managers and administrators can rename, reorder, activate, or deactivate stages.

## ADR-023: Appointments were deferred through Phase 4

Phase 3 and Phase 4 stopped before booking. Phase 5A implements appointments. A full calendar UI remains later work.

## ADR-024: Customer is a first-class entity

A customer is not a lead status. `customers` holds established contacts. Leads remain the qualification record.

## ADR-025: Lead references the customer it converted into

`leads.customer_id` is nullable. One customer may later have multiple leads. Phase 4 conversion always creates one new customer for that lead.

## ADR-026: Customers are not hard-deleted

There is no `DELETE /customers/{customer}`. Future appointments will need a stable customer row.

## ADR-027: Conversion is an explicit transactional action

`POST /api/v1/leads/{lead}/convert` creates the customer, links the lead, moves the lead to `converted`, and writes `lead_converted` plus `customer_created` activities in one transaction. Repeat conversion is rejected with `422`.

## ADR-028: Staff customer visibility comes from related leads

There is no `customer_owner_id`. Staff see a customer only when they are assigned to at least one of that customer's leads. Direct customer creation is manager/admin only.

## ADR-029: Duplicate customer matching is deferred

Phase 4 only prevents converting the same lead twice. Fuzzy matching and merges are out of scope.

## ADR-030: Appointments attach only to customers

Booking uses `Lead → Customer → Appointments`. Unconverted leads cannot be scheduled. This avoids a second appointment identity and keeps history on the customer record.

## ADR-031: Backend calculates appointment end time

`end_time` is `start_time + Service.duration_minutes`. Clients may display duration and suggest slots. They cannot submit an authoritative end time.

## ADR-032: Interval overlap with adjacent bookings allowed

Two occupying appointments overlap when `start < other.end AND end > other.start`. Cancelled appointments do not occupy a slot. Completed and no-show rows still occupy the original interval.

## ADR-033: Administrators are not bookable staff

Bookable users are active `staff` or `manager` accounts, matching availability and lead assignment. An administrator is an operator, not a practitioner, unless they also have a staff/manager account.

## ADR-034: Simple slot suggestions, not a calendar engine

`GET /api/v1/appointments/slots` returns 30-minute starts that fit availability and duration. Phase 5A does not add a week/month calendar or recurring series.

## ADR-035: Application timezone for past-date checks

Past-date validation uses `config('app.timezone')` and `now()`. LeadFlow currently keeps the existing app timezone rather than introducing a Philippines-specific scheduling clock.

## Environment notes

- PHP 8.3.32 is available via XAMPP (`C:\xampp\php\php.exe`)
- `pdo_pgsql` was already enabled
- `pdo_sqlite` and `sqlite3` were enabled in `C:\xampp\php\php.ini` so Laravel's default test suite can run
- PostgreSQL 18 is installed and the `postgresql-x64-18` service is running
- Docker is not available on this machine
- The Laravel installer CLI is not installed globally; Composer was used instead
