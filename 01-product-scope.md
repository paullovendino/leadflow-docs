# Product scope

## Vision

LeadFlow helps appointment-based businesses capture leads, manage a CRM pipeline, book appointments, and convert leads into customers.

Core workflow:

Landing Page → Lead Form → Lead → CRM → Pipeline → Appointment → Customer → Appointment History

## In scope for V1

- Public lead capture
- Authenticated CRM for leads, notes, and activity history
- Database-backed pipeline stages
- Staff assignment
- Lead-to-customer conversion
- Services, staff availability, and internal calendar
- Appointment scheduling with conflict detection
- Role-based authorization
- Operational dashboard metrics

## Out of scope for V1

Do not implement unless a later phase explicitly requests it:

- n8n
- Google Calendar synchronization
- External email providers
- SMS
- Webhooks
- Payment processing
- Multi-tenant billing

## Users

LeadFlow is operated by staff of a single appointment-based business in V1. There is no tenant model yet.

Roles:

- administrator
- manager
- staff

## Success criteria for the portfolio

The project should demonstrate:

- Clear business-domain modeling
- Thin controllers and explicit services
- REST API design
- Cookie-based SPA authentication
- Backend-authoritative authorization
- Database integrity
- Feature tests around business workflows
