
---

## 💡 Why This Architecture?

**Microservices**
Each service owns its own business logic and database, enabling independent deployment, scalability, and maintainability.

**Database per Service**
Services do not directly access each other's databases, reducing coupling and preserving service boundaries.

**Dual Event Bus (RabbitMQ + Azure Service Bus)**
Local development uses RabbitMQ for a fast, dependency-free feedback loop. Production deployments swap in Azure Service Bus for managed scaling, dead-lettering, and geo-redundancy — the application code targets a shared messaging abstraction, so the switch is configuration-driven, not code-driven.

**No separate Identity service**
Rather than building and securing a custom Identity microservice, the API Gateway issues and validates its own signed JWTs (see Authentication & Authorization above). The Gateway's JWT config also supports Microsoft Entra External ID as an alternate, config-driven branch — which would offload credential storage, MFA, and social login to a managed identity provider — but that branch isn't connected in this environment.

**Custom API Gateway Middleware**
The gateway is implemented as custom .NET 10 middleware rather than an off-the-shelf reverse proxy, giving full control over JWT validation, correlation ID propagation, and rate limiting — all tuned specifically to this platform's needs.

**Hangfire**
Hangfire handles background processing and scheduled tasks such as reminders and retries. It complements the event bus rather than replacing it — events drive _reactions_, Hangfire drives _time-based_ execution.

**Polly**
Resilience policies (retry, circuit breaker, timeout) are applied at service-to-service and service-to-Azure-resource boundaries, so transient failures in Azure Service Bus, Azure SQL, or downstream services don't cascade.

**API Gateway**
Provides a single entry point for clients and centralizes routing, authentication, rate limiting, and tracing.

**AI Assistant**
AI is integrated as its own service to keep machine learning capabilities isolated from core business logic and independently scalable/replaceable.

---

## 📂 Project Structure

```
IntakeIQ/
├── src/
│   ├── ApiGateway/                  # .NET 10 — routing, JWT issuing/validation, rate limiting
│   │   └── Auth/                    # JwtIssuer, JwtOptions, AuthEndpoints (register/sign-in)
│   ├── Services/
│   │   ├── Patient/
│   │   │   ├── Patient.Api/
│   │   │   ├── Patient.Application/
│   │   │   ├── Patient.Domain/
│   │   │   └── Patient.Infrastructure/
│   │   ├── Scheduling/
│   │   │   ├── Scheduling.Api/
│   │   │   ├── Scheduling.Application/
│   │   │   ├── Scheduling.Domain/
│   │   │   └── Scheduling.Infrastructure/
│   │   ├── Notification/            # same 4-layer shape as Patient/Scheduling, plus Hangfire
│   │   │   ├── Notification.Api/
│   │   │   ├── Notification.Application/
│   │   │   ├── Notification.Domain/
│   │   │   └── Notification.Infrastructure/
│   │   └── AIAssistance/            # same 4-layer shape, plus the Claude API integration
│   │       ├── AIAssistance.Api/
│   │       ├── AIAssistance.Application/
│   │       ├── AIAssistance.Domain/
│   │       └── AIAssistance.Infrastructure/
│   └── Shared/
│       ├── Shared.Contracts/        # Domain event definitions
│       ├── Shared.Messaging/        # RabbitMQ / Azure Service Bus abstraction
│       └── Shared.Observability/    # Serilog, OpenTelemetry, correlation ID, Polly resilience
├── frontend/                        # Angular 21 app — standalone/zoneless, Signals, no NgRx
│   ├── src/app/
│   │   ├── core/                    # auth (stub/Entra seam), API clients, shared models, date utils
│   │   ├── features/
│   │   │   ├── auth/                # login / sign-up
│   │   │   ├── dashboard/
│   │   │   ├── doctors/             # search + booking
│   │   │   ├── appointments/        # list, cancel, reschedule
│   │   │   ├── chat/                # AI assistant
│   │   │   ├── profile/
│   │   │   ├── notifications/
│   │   │   └── staff/               # Staff-only reminder worklist
│   │   ├── shared/                  # icon component, availability calendar
│   │   └── shell/                   # app shell (sidebar nav)
│   ├── Dockerfile                   # two-stage: node build -> nginx serve
│   └── nginx.conf
├── tests/
│   ├── Patient.Tests/
│   ├── Scheduling.Tests/
│   ├── Notification.Tests/
│   ├── AIAssistance.Tests/
│   ├── ApiGateway.Tests/
│   └── Shared.Messaging.Tests/      # real RabbitMQ integration test, needs a live broker
├── docker-compose.yml
├── IntakeIQ.sln
└── .github/workflows/               # CI/CD pipelines
```

