# Phase 8 — Calendar and scheduling UX

Phase 8 adds a visual day/week schedule on `/admin/appointments`. It does **not** replace the Phase 5A/6 appointment domain.

```
Existing Appointment API
        ↓
AppointmentsView (list or calendar)
        ↓
AppointmentFormDrawer / AppointmentDetailView
```

There is no calendar table, no `GET /appointments/calendar`, and no second conflict engine.

## Scope

Must:

- List | Calendar toggle (list remains)
- Desktop default: Calendar → Week
- Mobile (`< lg`): Calendar → Day (not seven columns)
- Day and week grids from `scheduled_date` / `start_time` / `end_time`
- Previous / Today / Next (Manila calendar dates)
- Appointment blocks → `/admin/appointments/{id}`
- Empty grid click → existing create drawer with date/time (and staff if filtered)
- Preserve `?customer_id=&create=1`
- Same list filters and staff visibility as the appointment list
- Filter past slot starts for **today** in `availableSlots()`
- No FullCalendar or other date libraries

Must not:

- Drag/drop or resize
- Month view, recurrence, resource (per-staff) columns
- Google Calendar, n8n, email/SMS, reminders, public booking
- New appointment statuses or migrations

## Data loading

Calendar uses the existing list endpoint:

- Day: `GET /api/v1/appointments?date={YYYY-MM-DD}&per_page=100`
- Week: `date_from` + `date_to` (Monday–Sunday) + `per_page=100`
- If `meta.last_page > 1`, additional pages are concatenated

Cancelled appointments are omitted from the grid (they do not occupy). Completed and no-show remain as historical blocks.

`GET /api/v1/appointments/slots` is used only inside the booking/reschedule drawer, never to paint the week.

## Booking from the calendar

Clicking a time snaps to the nearest 30-minute start and opens `AppointmentFormDrawer`. The drawer still loads slots. If the preset start is missing (past, overlap, or duration), it is cleared. `POST /api/v1/appointments` is unchanged.

## Timezone

Today, the now line, and date navigation use `Asia/Manila` via `Intl.DateTimeFormat` (`src/lib/format.ts`). Calendar dates are `YYYY-MM-DD` strings, not `new Date("YYYY-MM-DD")` UTC parsing.

## Authorization

Unchanged. Staff see the same rows as `GET /appointments` (assigned as `staff_user_id` or a viewable customer). Staff do not get an All Staff selector. Admin/Manager keep the existing staff filter; that is not a multi-column resource calendar.

## Tests

`AppointmentManagementTest::test_available_slots_for_today_exclude_past_starts` covers Manila “today” slot filtering. Existing appointment, workflow, dashboard, and qualification tests remain the suite.

There is no frontend test framework. Browser verification is the SPA check.

## Intentionally deferred

- Drag/drop reschedule
- Availability-window shading
- Multi-staff resource week
- Month view
- Integrations (email, n8n, Google Calendar)
- Public booking
- Performance / Pipeline TTFB work
