# Phase 3 — Leads, CRM, and pipeline

Phase 3 adds the CRM foundation: one default pipeline, database-backed stages, leads, polymorphic notes, and polymorphic activities. Customers landed in Phase 4, appointments in Phase 5A, and public lead capture in Phase 5B. Integrations remain out of scope.

## Entities

| Entity | Table | Purpose |
| --- | --- | --- |
| Pipeline | `pipelines` | Named sales process. V1 seeds one default pipeline. |
| PipelineStage | `pipeline_stages` | Ordered stage inside a pipeline |
| Lead | `leads` | Inbound inquiry that can be assigned and moved |
| Activity | `activities` | Immutable timeline event |
| Note | `notes` | Staff-authored comment |

## Relationships

```
Pipeline 1—* PipelineStage
PipelineStage 1—* Lead
Lead belongsTo Service (optional)
Lead belongsTo User as assignedUser (optional)
Lead morphMany Activity (alias `lead`)
Lead morphMany Note (alias `lead`)
Activity belongsTo User (nullable if the author is later removed)
Note belongsTo User
```

## Default pipeline

Seeded name: `Default Lead Pipeline` (`is_default`, `is_active`).

Stages (position order):

1. New (`new`)
2. Contacted (`contacted`)
3. Qualified (`qualified`)
4. Appointment Booked (`appointment_booked`)
5. Appointment Completed (`appointment_completed`)
6. Converted (`converted`)
7. Not Interested (`not_interested`)
8. Lost (`lost`)
9. No Response (`no_response`)

The SPA reads stages from `GET /api/v1/pipeline`. Stage slugs are not hard-coded in Vue.

## Lead rules

- `name` is required.
- `email` or `phone` is required (both may be present).
- `service_id` is optional. Inactive services are rejected.
- `assigned_user_id` is optional. Assignees must be active `staff` or `manager` users.
- `pipeline_stage_id` is required. If omitted on create, the default pipeline's active `New` stage is used.
- Staff who create a lead are assigned to themselves so they can still see the record.
- Leads are not hard-deleted. Use Lost, Not Interested, or No Response instead.

Sources are the `LeadSource` enum: `website`, `facebook`, `instagram`, `referral`, `google`, `walk_in`, `other`.

## Activities

Types: `lead_created`, `lead_assigned`, `stage_changed`, `note_added`, `lead_updated`.

Stage-change metadata stores `old_stage_id`, `old_stage_name`, `new_stage_id`, and `new_stage_name`. Descriptions are human-readable, but metadata is the historical source of truth.

Activities are append-only. There is no activity CRUD besides listing.

## Notes

Notes are append-oriented. Phase 3 supports create and list/view. Editing and versioning are deferred.

Creating a note also writes a `note_added` activity in the same transaction.

## Authorization

| Action | administrator | manager | staff |
| --- | --- | --- | --- |
| Create leads | yes | yes | yes (auto-assigned to self) |
| View / update leads | all | all | assigned only |
| Assign leads | yes | yes | no |
| Move stages | all leads | all leads | assigned leads |
| Manage pipeline stages | yes | yes | no |
| Notes / activities | all leads | all leads | assigned leads |

`Gate::before` still grants administrators every policy ability. The API enforces these rules even if the SPA hides controls.

## API

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/api/v1/leads` | Search, filter, paginate leads |
| POST | `/api/v1/leads` | Create lead |
| GET | `/api/v1/leads/{lead}` | Lead detail, notes, activities |
| PATCH | `/api/v1/leads/{lead}` | Update lead (assignment/stage reuse service methods) |
| PATCH | `/api/v1/leads/{lead}/assignment` | Assign or unassign |
| PATCH | `/api/v1/leads/{lead}/stage` | Move stage |
| GET | `/api/v1/leads/{lead}/notes` | List notes |
| POST | `/api/v1/leads/{lead}/notes` | Add note |
| GET | `/api/v1/leads/{lead}/activities` | Timeline, newest first |
| GET | `/api/v1/pipeline` | Default pipeline and stages |
| PATCH | `/api/v1/pipeline/stages/{stage}` | Rename or reorder |
| POST | `/api/v1/pipeline/stages/{stage}/activate` | Activate stage |
| POST | `/api/v1/pipeline/stages/{stage}/deactivate` | Deactivate stage |

Lead list query parameters: `search`, `stage`, `assigned_user`, `service`, `source`, `page`, `per_page` (1–100).

List responses eager-load `service`, `assignedUser`, and `pipelineStage`. Notes and activities load only on detail/timeline endpoints.

## Frontend

Authenticated routes:

- `/admin/leads` — table, search, filters, pagination, create
- `/admin/leads/:id` — details, assignment, stage change, notes, activity
- `/admin/pipeline` — Kanban columns from the API, optional stage management for managers/admins

## Future extension points

Do not implement these in Phase 3:

- Public landing page / unauthenticated lead form
- Customer conversion
- Appointments and calendar
- n8n, email, SMS, Google Calendar, webhooks
