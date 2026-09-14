# Hospital Appointment System

Backend of **four independent microservices** for hospital appointment scheduling, patient
history, and automatic reminders, with role-based access control and asynchronous
service-to-service communication.

Built as a graduate (pós-graduação) project to demonstrate Clean Architecture, CQRS, event-driven
integration between services, and stateless JWT security in a realistic multi-service backend.

| Service | Repo (submodule) | Port | Responsibility |
|---|---|---|---|
| [`identity-service`](identity-service/README.md) | `biadevcosta/identity-service` | 8080 | Authentication, JWT (RS256) issuance, user management |
| [`scheduling-service`](scheduling-service/README.md) | `biadevcosta/scheduling-service` | 8081 | Writes appointments (source of truth), publishes events |
| [`notification-service`](notification-service/README.md) | `biadevcosta/notification-service` | 8082 | Consumes reminders, resolves the patient's contact, sends them |
| [`history-service`](history-service/README.md) | `biadevcosta/history-service` | 8083 | Read side (CQRS): consumes events, serves history queries |

Each service has its own README with endpoints, request/response examples, and how to run and test
it in isolation. This document covers the system as a whole: architecture, business rules,
communication between services, and how to run everything together.

See [`arquitetura-sistemas.drawio`](arquitetura-sistemas.drawio) for the visual diagrams
(component/data-flow view and an end-to-end walkthrough) — open it at [app.diagrams.net](https://app.diagrams.net).
For a hands-on manual test session (Swagger + GraphiQL, copy-paste ready, PT-BR), see
[`ROTEIRO-TESTES.md`](ROTEIRO-TESTES.md).

---

## 1. Architecture

Every service follows **Clean Architecture** with the same three layers and the same dependency
rule: dependencies point inward, never outward.

```
<service>/
├── domain/            pure entities + business rules — no framework imports
├── application/       use cases as plain classes (no framework annotations)
│                       + ports (interfaces) that the domain/use cases depend on
└── infrastructure/    everything concrete: GraphQL/REST adapters, persistence,
                        messaging, security, @Configuration wiring
```

- `domain`: entities and invariants only. No Spring, no JPA/JDBC annotations, no HTTP/messaging types.
- `application`: use cases as plain objects, wired with dependencies through the constructor. Ports
  (interfaces) that a use case needs — a repository, a publisher, a token issuer — are **defined
  here**, never in `infrastructure`.
- `infrastructure`: implements every port with a concrete adapter (a JDBC repository, a Rabbit/Kafka
  publisher, an HTTP client), exposes the API (GraphQL resolver or REST controller), and wires it
  all together with `@Configuration` classes.

**Why this shape:** a use case's test never needs Spring, a real database, or a real broker — every
collaborator is an interface a test can fake. Swapping an adapter (e.g. a different e-mail provider,
a different message broker) never touches `domain` or `application`.

Stack: **Java 21, Spring Boot 4.1, Spring Data JDBC (no JPA/Hibernate), MySQL, Flyway, Spring for
GraphQL, Spring Security (JWT RS256), RabbitMQ, Kafka.** Tests: JUnit 5 + Mockito + AssertJ (unit),
Testcontainers (integration). Coverage gate: **80% line coverage (JaCoCo)** per service.

Key system-wide properties:

- **CQRS:** `scheduling-service` writes; `history-service` serves reads from its own read model,
  built asynchronously from events.
- **Two message brokers, two different jobs:** RabbitMQ carries a one-off task (send this reminder);
  Kafka carries an ordered, replayable event log (the history feed).
- **Stateless JWT (RS256):** `identity-service` signs tokens with a private key; every other service
  validates them locally with the matching public key. No shared session store, no API gateway,
  no network call needed just to check if a token is valid.
- **Database per service:** each service owns its schema; no service reads another service's
  database directly.
- **User data resolved on demand:** a service that needs a user's name/e-mail calls
  `identity-service` over HTTP (through a port, with a local cache) instead of copying user data
  into its own database.

---

## 2. Business rules

### 2.1 Roles and permissions

Three roles: `DOCTOR`, `NURSE`, `PATIENT` (plus `ADMIN`, internal to `identity-service`, used only
to register users).

| Capability | DOCTOR | NURSE | PATIENT |
|---|:---:|:---:|:---:|
| Create (register) an appointment | ✅ | ✅ | ❌ |
| Edit an appointment | ✅ (owner only) | ❌ | ❌ |
| Read appointment history | ✅ (any patient) | ✅ (any patient) | ✅ (own only) |
| Read future appointments | ✅ (any patient) | ✅ (any patient) | ✅ (own only) |

| Operation | Allowed roles |
|---|---|
| `scheduleAppointment` (mutation) | `DOCTOR`, `NURSE` |
| `editAppointment` (mutation) | `DOCTOR`, **and** must be the appointment's owner |
| `history` / `futureAppointments` (queries) | `DOCTOR`, `NURSE`, `PATIENT` |

### 2.2 Appointment lifecycle

- Status is an enum: `SCHEDULED`, `COMPLETED`, `CANCELLED`.
- A newly created appointment starts as `SCHEDULED`.
- **"Future appointment"** = status `SCHEDULED` **and** `scheduledAt` in the future.
- **"History"** = all appointments of a patient, regardless of status.

### 2.3 Ownership rules

- **Only the owner edits:** the appointment's doctor (`doctorId`) is its owner. Only that doctor may
  edit it — enforced inside the domain/use case, not only by the role gate.
- **A patient only sees their own data:** for a `PATIENT` caller, the `patientId` used to query
  history/future appointments comes from the **authenticated token**, never from a client-supplied
  argument. For `DOCTOR`/`NURSE`, the requested `patientId` argument is honored.

### 2.4 Invariants (rejected with a domain exception)

- `scheduledAt` must be in the **future**, both at creation and at edit.
- `patientId` and `doctorId` are required.
- Status transitions only ever use a valid enum value.

### 2.5 Where authorization lives

- **Coarse role gate** (`hasRole` / `hasAnyRole`) → on the adapter, via `@PreAuthorize` on the
  GraphQL resolver / REST controller.
- **Fine-grained rules** (ownership, "patient sees only their own") → inside the **use case**,
  which receives the caller's identity (role, userId/patientId) as a plain constructor/method
  parameter. The core never reads `SecurityContextHolder` directly.

### 2.6 Messaging behavior

- In `scheduling-service`, on both **create** and **edit**, the order is always: (1) validate,
  (2) **persist**, (3) **then** publish. Nothing is ever published before the appointment is saved.
  - RabbitMQ — `AppointmentReminder` (carries only ids: `appointmentId`, `patientId`, `scheduledAt`).
  - Kafka topic `appointment-events` — `AppointmentCreated` / `AppointmentUpdated` (ids + status;
    partition key = `patientId`, so events for the same patient are strictly ordered).
- `notification-service` consumes the reminder, resolves the patient's name/e-mail from
  `identity-service` (cached), and sends it (delivery is currently a log line — a real provider is
  a pluggable adapter, see its README).
- `history-service` consumes `appointment-events` **idempotently** (a `processed_events` table
  ignores an already-seen `eventId`, since Kafka may redeliver), builds/updates its read model, and
  resolves user names from `identity-service` at query time (so it always shows the **current**
  name, not a stale copy).

---

## 3. Communication between services

Rule: **client → service is synchronous (HTTP/GraphQL)**; **service → service is asynchronous
(broker)** — the only exception is the on-demand user lookup to `identity-service`, which is a
synchronous, cached HTTP call on a non-critical path.

| From → To | Type | Mechanism | Payload |
|---|---|---|---|
| Client → Identity | sync | HTTP REST | credentials → JWT; user registration |
| Client → Scheduling | sync | GraphQL + JWT | schedule / edit mutation |
| Client → History | sync | GraphQL + JWT | history / future-appointments query |
| Scheduling → Notification | async | RabbitMQ `reminder.queue` | reminder (ids only) |
| Scheduling → History | async | Kafka `appointment-events` | appointment created/updated (ids + status) |
| Notification → Identity | sync (cached) | HTTP | patient name/e-mail |
| History → Identity | sync (cached) | HTTP | user name resolution |

### Data per service

| Database | Owner | Tables |
|---|---|---|
| `identity_db` | identity-service | `users`, `refresh_tokens` |
| `scheduling_db` | scheduling-service | `appointments` |
| `notification_db` | notification-service | `processed_reminders` (idempotency only) |
| `history_db` | history-service | `appointment_history`, `processed_events` |

### Brokers

| Broker | Name | Flow | Semantics |
|---|---|---|---|
| RabbitMQ | `reminder.queue` (+ `reminder.dlq`) | scheduling → notification | one-off task, retried 3× then dead-lettered |
| Kafka | `appointment-events` (key = `patientId`) | scheduling → history | ordered per patient, replayable, consumer is idempotent |

---

## 4. Security — JWT signed with RS256

`identity-service` is the only service that can **issue** a token; every other service can only
**verify** one. This is possible because RS256 is an asymmetric signature: a **private key**
produces a signature that only the matching **public key** can verify, and the public key alone is
useless for forging a new signature.

- `identity-service` holds `private.pem` and signs every access token with it after a successful
  login.
- `scheduling-service` and `history-service` hold a copy of `public.pem` and validate the token's
  signature **locally**, with no network call back to `identity-service` — keeping the write and
  read paths fast and decoupled. `identity-service` is only ever called synchronously to fetch a
  user's profile (name/e-mail), never to ask "is this token valid?".

Token claims: `sub` (userId), `role`, `patientId` (only for patient users), plus `iss`, `aud`,
`iat`, `exp`.

Generating the key pair (done once, `private.pem` is git-ignored everywhere):

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out keys/private.pem
openssl rsa -in keys/private.pem -pubout -out keys/public.pem
```

`public.pem` is copied into every service's `src/main/resources/`; `private.pem` lives only in
`identity-service`.

---

## 5. Repository layout

Each of the four services is its own Git repository, wired into this one as a **Git submodule**
(`github.com/biadevcosta/<service>`), with its own `Dockerfile`, `pom.xml`, tests, and README.

```
hospital-system/
├── docker-compose.yml          # runs all 4 services + their infra together
├── arquitetura-sistemas.drawio # architecture + end-to-end flow diagrams
├── identity-service/           # submodule
├── scheduling-service/         # submodule
├── notification-service/       # submodule
└── history-service/            # submodule
```

---

## 6. How to run the whole system

The root `docker-compose.yml` starts all four services, one MySQL per service, a shared RabbitMQ,
and a shared Kafka, on a single Docker network:

```bash
git clone --recurse-submodules <this-repo-url>
cd hospital-system

# generate the RSA key pair once, then copy the public key into every service
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out keys/private.pem
openssl rsa -in keys/private.pem -pubout -out keys/public.pem
cp keys/private.pem identity-service/src/main/resources/private.pem
cp keys/public.pem  identity-service/src/main/resources/public.pem
cp keys/public.pem  scheduling-service/src/main/resources/public.pem
cp keys/public.pem  history-service/src/main/resources/public.pem

docker compose up --build -d
docker compose logs -f identity scheduling notification history   # follow startup
```

| Endpoint | URL |
|---|---|
| identity-service (REST + Swagger) | http://localhost:8080/swagger-ui.html |
| scheduling-service (GraphiQL) | http://localhost:8085/graphiql |
| history-service (GraphiQL) | http://localhost:8083/graphiql |
| notification-service | no API — consumer only, check `docker compose logs notification` |
| RabbitMQ management UI | http://localhost:15672 (guest/guest) |
| Kafka UI | http://localhost:8084 |
| Adminer (all 4 databases) | http://localhost:8090 (root/root) |

A seeded admin (`admin@hospital.local` / `admin12345`) is created on `identity-service` startup, so
`POST /users` can be used immediately to register a doctor and a patient.

**Exercising the full flow** (see the "end-to-end flow" page in the `.drawio` diagram for the
illustrated version):

1. `identity` → log in as admin → register a `DOCTOR` and a `PATIENT` → log in as the doctor and
   keep the access token.
2. `scheduling` → `scheduleAppointment` with that token → persists to `scheduling_db`, publishes to
   RabbitMQ and Kafka.
3. `notification` → consumes the reminder, resolves the patient's name/e-mail from `identity`, logs
   the simulated e-mail.
4. `history` → consumes the Kafka event, then `history(patientId)` / `futureAppointments(patientId)`
   return the appointment.

To stop everything: `docker compose down` (add `-v` to also wipe the MySQL volumes).

## 7. How to test

Each service is independently testable — see its README for details. In every service:

```bash
cd <service>
./mvnw test       # unit tests only — no Docker required
./mvnw verify      # + integration test (Testcontainers) + JaCoCo 80% line-coverage gate
```

Integration tests self-skip (instead of failing) when no Docker daemon is reachable, so `./mvnw
verify` always completes. Coverage report: `target/site/jacoco/index.html` in each service.

Each service also ships an OpenAPI/GraphiQL explorer (see its README) and, for `identity-service`
and `scheduling-service`, an importable Insomnia collection.

For a full manual walkthrough — creating users, scheduling an appointment, checking the
reminder and the history — with exact requests/variables and expected responses, see
[`ROTEIRO-TESTES.md`](ROTEIRO-TESTES.md).
