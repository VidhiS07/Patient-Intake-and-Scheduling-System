# Patient-Intake-and-Scheduling-System

**Cloud-Native Event-Driven Microservices — .NET 8 · Angular 21 · Azure**

A distributed healthcare workflow platform that automates patient intake, appointment scheduling, reminders, and post-care follow-up, designed as a production-grade event-driven microservices system to demonstrate real-world backend and distributed systems engineering.

---

## Overview

Healthcare operations suffer from fragmented, manual workflows — intake forms processed by hand, reminders sent ad hoc, and no-shows with no automated follow-up. This system replaces those gaps with an event-driven microservices platform where each domain owns its data, communicates asynchronously, and fails gracefully without cascading.

The architecture is intentionally scoped to a vertical slice that demonstrates the hardest engineering problems, not maximum breadth. The Patient, Scheduling, and Notification services are fully implemented end-to-end. The Follow-up and AI Assistance services are included as working consumers to complete the event chain.

---

## Key Features

**Patient Management**

- Patient registration with demographic capture and PHI-safe storage
- Role-based access control — `patient`, `clinician`, and `admin` claims enforced at the gateway
- All PHI access audit-logged with user identity, action, and timestamp
- Encryption at rest (Azure SQL TDE) and in transit (TLS 1.2+) on all patient data

**Appointment Scheduling**

- Appointment booking with real-time slot availability and conflict detection
- Full appointment lifecycle — Created → Confirmed → Completed / Cancelled / No-Show
- Rescheduling and cancellation with automatic downstream notification triggers
- No-show flagging with follow-up workflow initiation

**Notifications**

- Automatic confirmation SMS/email on appointment creation
- Reminder dispatch at configurable intervals before appointment time
- No-show and cancellation alerts routed to the relevant clinician
- Full delivery log per notification — status, channel, timestamp, and retry history

**Post-Care Follow-Up**

- Post-visit questionnaire automatically scheduled on appointment completion
- Patient response capture with structured storage per visit
- AI-assisted summarisation of free-text responses for clinician review (non-clinical, assistive only)
- Follow-up status tracking — Scheduled → Sent → Responded → Reviewed

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Angular 21 |
| API Gateway | .NET 8 — custom middleware, JWT validation, correlation ID injection |
| Microservices | .NET 8 Web API |
| Event Bus (local) | RabbitMQ |
| Event Bus (cloud) | Azure Service Bus |
| Databases | SQL Server (local) · Azure SQL Database (cloud) |
| Auth | Azure AD B2C — OAuth2 / OIDC / JWT RS256 |
| Secrets | Azure Key Vault |
| Observability | OpenTelemetry · Azure Application Insights · Serilog |
| Resilience | Polly (.NET) |
| Containerisation | Docker · Docker Compose |
| CI/CD | GitHub Actions · Azure Container Registry |
| Cloud | Azure App Service · Azure API Management |

---

## System Architecture

```
                          ┌─────────────────┐
                          │ Angular Frontend │
                          └────────┬────────┘
                                   │ HTTPS
                    ┌──────────────▼──────────────┐
                    │         API Gateway          │
                    │  JWT validation · Routing    │
                    │  Rate limiting · TraceId     │
                    │  Azure AD B2C (OAuth2/OIDC)  │
                    └──────┬──────────────┬────────┘
                           │              │
            ┌──────────────▼──┐      ┌───▼──────────────┐
            │ Patient Service │      │Scheduling Service │
            │  (PatientsDB)   │      │  (SchedulingDB)   │
            └──────────┬──────┘      └────────┬──────────┘
                       │                      │
                       │    ┌─────────────────▼──────────────────┐
                       │    │  Notification Service               │
                       │    │  (NotificationDB)                   │
                       │    └─────────────────┬──────────────────┘
                       │                      │
                       └──────────┬───────────┘
                                  │ Domain Events
                    ┌─────────────▼─────────────────┐
                    │         Event Bus              │
                    │  RabbitMQ (local)              │
                    │  Azure Service Bus (cloud)     │
                    └──────┬─────────────┬───────────┘
                           │             │
            ┌──────────────▼──┐    ┌─────▼──────────────┐
            │ Follow-Up       │    │  AI Assistance      │
            │ Service         │    │  Service            │
            │ (FollowUpDB)    │    │  (AIServiceDB)      │
            └─────────────────┘    └────────────────────┘

```

---

## Running Locally

```bash
git clone https://github.com/your-username/healthcare-system
cd healthcare-system
docker-compose up --build
```

**Prerequisites:** Docker Desktop · .NET 8 SDK · Node.js 18+

---

## Core Workflow — End to End

```
1.  Patient registers via Angular frontend
    → POST /patients through API Gateway
    → Patient Service stores record in PatientsDB
    → PatientRegistered event written to OutboxEvents (same transaction)
    → OutboxPublisher sends PatientRegistered to event bus

2.  Patient books appointment
    → POST /appointments through API Gateway
    → Scheduling Service validates slot, stores in SchedulingDB
    → AppointmentCreated event written to OutboxEvents (same transaction)
    → OutboxPublisher sends AppointmentCreated to event bus

3.  Notification Service consumes AppointmentCreated
    → Sends confirmation SMS/email
    → Logs delivery to NotificationDB
    → Idempotency key prevents duplicate sends on replay

4.  Follow-Up Service consumes AppointmentCreated
    → Schedules post-visit questionnaire workflow

5.  Patient submits follow-up form
    → Follow-Up Service stores response in FollowUpDB
    → FollowUpSubmitted event published
    → AI Assistance Service generates structured summary (non-clinical)
    → Summary available for clinician review via API
```

---

## Author

Vidhi Sheth
Full Stack Engineer — .NET · Angular · Microservices · Azure · System Design

## 

This is a private repository. If you'd like a live demo or code walkthrough, feel free to reach out.

