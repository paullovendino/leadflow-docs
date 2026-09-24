# Future integrations

These are explicitly out of scope until a later phase requests them.

The service layer should remain the place where side effects attach, so these can be added without rewriting controllers.

## Email

Possible later workflow:

- Lead created → confirmation
- Appointment created → confirmation
- Appointment approaching → reminder

V1 can log mail with the `log` mailer. That is not an external provider integration.

## n8n

Possible later workflow:

Laravel domain event or HTTP webhook → n8n → email / notification / external system

Do not add an n8n dependency now.

## Google Calendar

Possible later workflow:

LeadFlow appointment ↔ Google Calendar event

V1 scheduling uses internal availability and appointments only.

## Webhooks

Possible later events:

- `lead.created`
- `lead.updated`
- `appointment.created`
- `appointment.updated`
- `appointment.cancelled`
- `lead.converted`

Do not add a webhook dispatcher now.

## Constraint for current work

Avoid schema or API choices that hard-wire LeadFlow to a single calendar vendor, mail vendor, or automation tool.
