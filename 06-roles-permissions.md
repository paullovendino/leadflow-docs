# Roles and permissions

Backend authorization is authoritative. Frontend role checks only change what the UI shows.

## Roles

Stored on `users.role` as a string, cast to `App\Enums\UserRole`.

| Role | Intended access |
| --- | --- |
| administrator | Full system access |
| manager | Leads, customers, appointments, services, staff, pipeline, reports |
| staff | Assigned leads and related customers, appointments, notes, own availability |

## Current enforcement

Phase 5A enforces catalog, lead, customer, and appointment policies on the backend.

- `users.role` with a PostgreSQL check constraint
- `users.is_active` for deactivation instead of user soft deletes
- `EnsureUserIsActive` middleware (`active`)
- `UserPolicy`, `ServicePolicy`, `StaffAvailabilityPolicy`, `LeadPolicy`, `PipelinePolicy`, `PipelineStagePolicy`, `CustomerPolicy`, `AppointmentPolicy`
- `User::hasRole()`, `isAdministrator()`, `isManager()`, `isStaff()`
- `Gate::before` allows all abilities for administrators

## Policy map

| Area | administrator | manager | staff |
| --- | --- | --- | --- |
| Users / staff admin | yes | staff and managers only | self view only |
| Services | yes | yes | read active services |
| Own availability | yes | yes | yes |
| Other staff availability | yes | yes | no |
| Leads | yes | all | assigned |
| Customers | yes | all | related assigned leads |
| Notes | yes | yes | related records |
| Pipeline stages | yes | yes | read / move assigned leads |
| Appointments | yes | yes | related |
| Reports | yes | yes | no |

Manager limits:

- Cannot assign the `administrator` role
- Cannot update or deactivate administrator accounts
- Cannot deactivate themselves

Administrators cannot deactivate or demote the last active administrator. That rule lives in `UserManagementService` because `Gate::before` would otherwise allow it.

Staff customer visibility is derived from assigned related leads. Staff cannot create customers directly; they convert assigned leads.

Staff see an appointment when they are the assigned `staff_user_id` or when they can view the customer. Staff may create an appointment only for a customer they can view. Administrators are never bookable staff.

## Inactive users

Inactive users cannot log in. If a session already exists, `active` middleware logs them out and returns `403`.
