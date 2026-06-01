
## Architecture Decisions

### Why microservices over a monolith?

The healthcare domain has naturally distinct bounded contexts — patient demographics, appointment state, notification delivery, and post-care data each change at different rates, have different scaling needs, and are owned by different teams in a real organisation. A monolith would couple these tightly, making independent deployment and scaling impossible. Microservices allow each domain to evolve, scale, and fail independently.

The cost is real: more infrastructure, distributed debugging, and no cross-service joins. For a single developer this is overhead. For a team operating a production healthcare platform it is the correct trade-off.

---

### Why event-driven over synchronous REST between services?

Synchronous service-to-service calls create temporal coupling — if the Notification Service is down, booking an appointment fails. That is the wrong failure mode for a healthcare system. Asynchronous events decouple the services: the Scheduling Service publishes `AppointmentCreated` and its job is done. The Notification Service processes it when it is ready.

This also makes it trivial to add new subscribers — a Follow-Up Service can react to `AppointmentCreated` without the Scheduling Service knowing it exists.

---

### Why the outbox pattern?

Naive event publishing — publish to the bus after the database commit — has a race condition. If the application crashes between the commit and the publish, the event is silently lost. The database was updated but no downstream service was notified.

The outbox pattern solves this by writing the event to an `OutboxEvents` table in the **same database transaction** as the domain write. A background worker then polls this table and publishes to the bus, marking events as published on success. The event is now guaranteed to eventually reach the bus, or the domain write is rolled back with it. This is the correct solution for at-least-once delivery in an event-driven system and is one of the least-understood patterns in distributed systems.

---

### Why database-per-service?

Shared databases are the most common way microservices architectures fail in practice. A shared schema creates invisible coupling: changing a table structure for one service breaks another. Schema locks held by one service slow down others. A single database becomes a single point of failure and a scaling bottleneck.

Database-per-service enforces the boundary in code and infrastructure. The cost — no cross-service joins — is a deliberate constraint that forces explicit API contracts between domains.

---

### Why JWT at the gateway with Managed Identity for service-to-service?

Validating tokens in every service duplicates logic, creates inconsistency, and increases attack surface. Centralising JWT validation at the gateway means a single policy enforcement point. Services trust the identity context forwarded by the gateway.

For internal service-to-service calls (Follow-Up calling AI Assistance), Azure Managed Identity provides short-lived credentials with no secrets in code or config. This is the Azure-native zero-secret model.

---

### Why the AI layer is restricted to non-clinical tasks only?

LLMs are probabilistic. A hallucination in a patient summary is acceptable context for a clinician who will verify it. A hallucination in a triage decision or drug recommendation is a patient safety event. The AI layer in this system is explicitly constrained to text structuring, summarisation, and communication simplification — tasks where an error is inconvenient, not dangerous. All clinical and operational decisions are handled by deterministic service logic or human workflows.
