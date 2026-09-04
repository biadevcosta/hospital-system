# Hospital Appointment System

A backend of **four independent microservices** for hospital appointment scheduling,
patient history, and automatic reminders, with role-based access and asynchronous
communication. This README is also the **context and rules for AI assistants** working
on the codebase — follow every rule here.

---

## 1. Language & conventions (strict — for AI and humans)

- **ALL code, identifiers, comments, commit messages, logs, test names, and docs are in English.** Portuguese may appear only in chat, never in artifacts.
- Each service follows **Clean Architecture**. The dependency rule is absolute:
  - `domain`: pure entities and business rules. No framework imports, no annotations.
  - `application`: use cases as **plain POJOs** (no `@Component`/`@Service`), dependencies via constructor. **Ports (interfaces) are defined here.**
  - `infrastructure`: all framework detail — GraphQL/REST adapters, persistence, messaging, security, and `@Configuration` wiring.
- Use cases are wired with `@Bean` in an infrastructure `@Configuration` (never annotated in `application`).
- **Golden check:** open any `domain`/`application` class and inspect imports. If you see `org.springframework`, `RestClient`, `RabbitTemplate`, `KafkaTemplate`, or any concrete infra class → it's a leak. Every external collaborator of a use case must be a **port** you defined in the core.
- Persistence entities (with `@Table`/`@Id`) live in `infrastructure` and are mapped to/from domain entities — persistence annotations never touch the domain.
- Stack: **Java 21, Spring Boot 4.1, Spring Data JDBC (no JPA/Hibernate), MySQL, Flyway, Spring for GraphQL, Spring Security (JWT RS256), RabbitMQ, Kafka.** Tests: JUnit 5 + Mockito + AssertJ (unit); Testcontainers (integration). Coverage target: **80% (JaCoCo)**.

---

## 2. Services overview

| Service | Port | Responsibility | DB |
|---|---|---|---|
| `identity-service` | 8080 | Authentication, JWT issuance, user management (source of truth for users) | `identity_db` |
| `scheduling-service` | 8081 | Writes appointments (source of truth), publishes events | `scheduling_db` |
| `notification-service` | 8082 | Consumes reminders and sends them to patients | none (optional) |
| `history-service` | 8083 | Read side (CQRS): consumes events, serves history queries | `history_db` |

Key properties:
- **CQRS:** `scheduling` writes; `history` serves reads from its own read model.
- **Two brokers:** RabbitMQ = task (reminder); Kafka = event log (history feed).
- **Stateless JWT (RS256):** `identity` signs with the private key; every service validates with the public key. No shared session, no gateway.
- **Database per service:** no service reads another service's database.
- **User data on demand:** services needing a user's name/email call `identity-service` over HTTP (behind a port, with cache). No data replication.

---

## 3. BUSINESS RULES (authoritative)

These rules are the core of the system. Implement them exactly.

### 3.1 Roles and permissions

Three roles: `DOCTOR`, `NURSE`, `PATIENT`.

| Capability | DOCTOR | NURSE | PATIENT |
|---|:---:|:---:|:---:|
| Create (register) an appointment | ✅ | ✅ | ❌ |
| Edit an appointment | ✅ (owner only) | ❌ | ❌ |
| Read appointment history | ✅ (any patient) | ✅ (any patient) | ✅ (own only) |
| Read future appointments | ✅ (any patient) | ✅ (any patient) | ✅ (own only) |

Operation → allowed roles:
- `scheduleAppointment` (mutation) → `DOCTOR`, `NURSE`
- `editAppointment` (mutation) → `DOCTOR` **and** must be the owner (see 3.3)
- `history` / `futureAppointments` (queries) → `DOCTOR`, `NURSE`, `PATIENT`

### 3.2 Appointment lifecycle

- Status is an enum: `SCHEDULED`, `COMPLETED`, `CANCELLED`.
- A newly created appointment starts as `SCHEDULED`.
- **"Future appointment"** = status `SCHEDULED` **and** `scheduledAt` in the future.
- **"History"** = all appointments of a patient, regardless of status.

### 3.3 Ownership rules (business logic — inside the use case)

- **Only the owner edits:** the appointment's doctor (`doctorId`) is the owner. Only that doctor may edit it. Enforce this **in the domain/use case**, not only via the role gate.
- **Patient sees only their own:** for a `PATIENT` caller, the `patientId` used to query MUST come from the **authenticated token**, never from a client-supplied argument. Do not trust an incoming `patientId` for a patient caller. For `DOCTOR`/`NURSE`, the requested `patientId` argument is honored.

### 3.4 Invariants (reject with a domain exception)

- `scheduledAt` must be in the **future** at creation and at edit.
- `patientId` and `doctorId` are required.
- Status transitions use only the valid enum values.
- (Optional feature) reject double-booking: same doctor, same time slot.

### 3.5 Authorization placement (Clean Architecture)

- **Coarse role gate** (`hasRole` / `hasAnyRole`) → on the adapter (GraphQL resolver / controller) via `@PreAuthorize`.
- **Fine-grained rules** (ownership, "patient only own") → inside the **use case**, receiving the caller identity (role, userId/patientId) as a **parameter**. **Never** read `SecurityContextHolder` inside the core.

### 3.6 Messaging behavior

- In `scheduling`, on **create** and on **edit**, the order is: (1) validate, (2) **persist** to the database, (3) **then** publish. Never publish before persisting.
  - Publish to **RabbitMQ**: `AppointmentReminder` (carries IDs, e.g. `appointmentId`, `patientId`, `scheduledAt`).
  - Publish to **Kafka** topic `appointment-events`: `AppointmentCreated` / `AppointmentUpdated` (carries IDs; partition key = `patientId` for per-patient ordering).
- `notification-service` consumes the reminder, resolves the patient's name/email from `identity-service` (via a port, cached), and sends the reminder (log channel is acceptable; real email/SMS is pluggable).
- `history-service` consumes `appointment-events` **idempotently** (a `processed_events` table; ignore already-seen `eventId` — Kafka may redeliver), builds/updates its read model, and resolves user names from `identity-service` at query time (shows the **current** name).

### 3.7 Identity & tokens

- `identity-service` authenticates (login), hashes passwords with **BCrypt**, and issues a **JWT signed with RS256** (private key).
- JWT claims: `sub` (userId), `role`, `patientId` (only for patient users), plus `iss`, `aud`, `exp`, `iat`.
- Other services validate the token with the **public key** (stateless). They never call `identity` to validate a token — only to fetch user profile data when needed.

---

## 4. Communications

Rule: **client → service is synchronous (HTTP); service → service is asynchronous (broker)** — except the on-demand user lookups to `identity`, which are synchronous HTTP (cached, on the least-critical paths).

| From → to | Type | Mechanism | Payload |
|---|---|---|---|
| Client → Identity | sync | HTTP (login) | credentials → token; profile |
| Client → Scheduling | sync | GraphQL + JWT | schedule/edit mutation |
| Client → History | sync | GraphQL + JWT | history / future queries |
| Scheduling → History | async | Kafka `appointment-events` | appointment created/updated (IDs) |
| Scheduling → Notification | async | RabbitMQ `reminder.queue` | reminder (IDs) |
| Notification → Identity | sync | HTTP (cached) | patient name/email |
| History → Identity | sync | HTTP (cached) | resolve user names |

---

## 5. Data & messaging

Databases (one per service):

| DB | Service | Tables |
|---|---|---|
| `identity_db` | identity | `users`, `doctor_profile`, `patient_profile` |
| `scheduling_db` | scheduling | `appointments` |
| `history_db` | history | `appointment_history`, `processed_events` |

Brokers:

| Broker | Name | Flow |
|---|---|---|
| RabbitMQ | `reminder.queue` (+ `reminder.dlq`) | Scheduling → Notification |
| Kafka | `appointment-events` (key = `patientId`) | Scheduling → History |

Ports defined in the core (implemented by infrastructure adapters):
`AppointmentRepository`, `ReminderPublisher`, `AppointmentEventPublisher`,
`UserDirectory`, `PasswordHasher`, `TokenIssuer`, `HistoryRepository`, `ProcessedEventStore`.

---

## 6. Repository layout (per service)

```
<service>/
├── domain/            # pure entities + business rules (no framework)
├── application/       # use cases (POJOs) + ports (interfaces) + commands
└── infrastructure/    # web (GraphQL/REST), persistence, messaging, security, config
```

Each service is its own Git repository with its own `Dockerfile`.

---

## 7. How to run (local)

Start infrastructure once (MySQL, RabbitMQ, Kafka) via Docker, create the three databases,
and generate the RSA key pair for RS256. Then run each service (`./mvnw spring-boot:run`) or
use `docker compose up` to start everything together.

Order to exercise the full flow:
1. Start `identity` → register a doctor and a patient → log in and copy the token.
2. Start `scheduling` → schedule an appointment with the token.
3. Start `notification` → see the reminder logged (name/email fetched from identity).
4. Start `history` → run the query and see the appointment appear.

Suggested build order: **scheduling → notification → history → identity** (each phase ends with something running end to end).

---

## 8. Deliverables

- Working endpoints for all required capabilities, with correct role-based access.
- Clean, layered code; no service imports another; core free of framework leaks.
- Tests (unit + Testcontainers integration), **JaCoCo ≥ 80%**, Allure report.
- Insomnia/Postman collection covering login, schedule/edit, and the queries.
- README per service + this architecture/rules document.

---

## 9. Current implementation status

| Service | Port | Responsibility | Status |
|---|---|---|---|
| `scheduling-service` | 8081 | Writes appointments (source of truth), publishes reminder (Rabbit) and event (Kafka) | ✅ **implemented + tested** |
| `identity-service` | 8080 | Authentication, JWT RS256 issuance, user registration | ⏳ skeleton |
| `notification-service` | 8082 | Consumes the Rabbit reminder, resolves contact via identity, "sends" | ⏳ skeleton |
| `history-service` | 8083 | Read side (CQRS): consumes Kafka events, serves GraphQL queries | ⏳ skeleton |

Each service is its own **Git submodule** (`github.com/biadevcosta/<service>`).

> ⚠️ The skeletons' `pom.xml` use **Spring Boot 4.1** module names
> (`spring-boot-starter-webmvc`, `spring-boot-starter-flyway`, per-slice test starters).
> The implementation guide was written for Boot 3.x — follow what's in `pom.xml`.

---

## 10. Messaging configuration (what has been defined)

Messaging was configured **on the producer side**, inside `scheduling-service`. The two brokers
have different roles:

| Broker | Role | Flow | Semantics |
|---|---|---|---|
| **RabbitMQ** | **Task** queue (reminder) | scheduling → notification | 1 logical consumer, with retry + DLQ |
| **Kafka** | **Event log** (history feed) | scheduling → history | ordered per patient, replayable, idempotent on the consumer |

Golden rule applied in the use cases: **validate → persist → publish**. Nothing is published before
the appointment is saved to the database (`ScheduleAppointmentUseCase` / `EditAppointmentUseCase`).

### 10.1 RabbitMQ — appointment reminder

**Connection** (`application.yaml`)

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
app:
  rabbit:
    exchange: appointment.reminders   # DirectExchange
    queue: reminder.queue             # main queue (durable)
    routing-key: appointment.reminder
```

**Topology declared in code** (`infrastructure/messaging/rabbit/RabbitConfig.java`)

| Bean | What it is | Detail |
|---|---|---|
| `reminderExchange` | `DirectExchange` `appointment.reminders` | — |
| `reminderQueue` | durable `Queue` `reminder.queue` | args: `x-dead-letter-exchange = ""` (default exchange) and `x-dead-letter-routing-key = reminder.dlq` |
| `reminderDlq` | durable `Queue` `reminder.dlq` | "dead" (rejected/expired) messages land here |
| `reminderBinding` | binding | binds `reminder.queue` → `appointment.reminders` with routing key `appointment.reminder` |
| `rabbitJsonConverter` | `Jackson2JsonMessageConverter` | payload travels as JSON |
| `rabbitTemplate` | `RabbitTemplate` | uses the JSON converter |

**Publisher** — `RabbitReminderPublisher implements ReminderPublisher` (core port).
Calls `convertAndSend(exchange, routingKey, payload)`.

**Payload** — `AppointmentReminderMessage(appointmentId, patientId, scheduledAt)`.
IDs only; `notification-service` resolves name/email from `identity-service`.

**Retry + DLQ:** the main queue already dead-letters to `reminder.dlq`. **Retry** (retrying
N times before sending to the DLQ) is the **consumer's** responsibility (`notification-service`), via
`application.yaml` — not implemented yet. Planned config:

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        retry: { enabled: true, max-attempts: 3, initial-interval: 2000 }
        default-requeue-rejected: false   # rejected goes to the DLQ, not back to the queue
```

### 10.2 Kafka — appointment events (history feed)

**Connection + serialization** (`application.yaml`)

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      properties:
        spring.json.add.type.headers: true
app:
  kafka:
    topic: appointment-events
```

**Topic declared in code** (`infrastructure/messaging/kafka/KafkaConfig.java`)

| Bean | Detail |
|---|---|
| `appointmentEventsTopic` | `NewTopic` `appointment-events`, **3 partitions**, 1 replica (dev) |

**Publisher** — `KafkaAppointmentEventPublisher implements AppointmentEventPublisher` (core port).
`publishCreated` / `publishUpdated` call `kafkaTemplate.send(topic, patientId, event)`.

- **Message key = `patientId`** → all events for a patient go to the same partition
  ⇒ **per-patient ordering** guaranteed.
- **`eventId` (random UUID)** on each event → `history-service` uses it for **idempotency**
  (`processed_events` table, ignores an already-seen `eventId`, since Kafka may redeliver).

**Payload** — `AppointmentEventMessage`:

```
type          "AppointmentCreated" | "AppointmentUpdated"   (constants CREATED / UPDATED)
eventId       unique UUID for the publication (consumer idempotency)
appointmentId, patientId, doctorId
scheduledAt, status
```

**Consumer (`history-service`)** — not implemented yet. Planned config:

```yaml
spring:
  kafka:
    consumer:
      group-id: history
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "*"
```

### 10.3 Summary of what runs today

```
Client ──GraphQL+JWT──▶ scheduling-service
                              │  1) validates (domain)
                              │  2) persists to scheduling_db  (Flyway: V1__create_appointments.sql)
                              │  3) publishes:
                              ├─▶ RabbitMQ  exchange "appointment.reminders"
                              │        routing-key "appointment.reminder"
                              │        → queue "reminder.queue"  (DLQ: "reminder.dlq")
                              │        payload: AppointmentReminderMessage (IDs)
                              └─▶ Kafka  topic "appointment-events" (key = patientId)
                                       payload: AppointmentEventMessage (type, eventId, IDs, status)
```

Consumers (`notification-service` on Rabbit, `history-service` on Kafka) are the **next step**.

---

## 11. What has been implemented in `scheduling-service`

Structured with **Clean Architecture** (the dependency rule is absolute: `domain` and `application`
import nothing from any framework).

```
domain/
  Appointment                 entity + rules: required IDs, scheduledAt in the future,
                              "only the owning doctor edits" (edit(callerDoctorId, ...))
  AppointmentStatus           SCHEDULED | COMPLETED | CANCELLED
  exception/                  AppointmentException, InvalidAppointmentException,
                              NotAppointmentOwnerException, AppointmentNotFoundException

application/
  usecase/ScheduleAppointmentUseCase   validate → repo.save → reminder.publish → event.publishCreated
  usecase/EditAppointmentUseCase       repo.findById → appointment.edit(...) → repo.save → event.publishUpdated
  port/AppointmentRepository            ports (interfaces) defined in the core
  port/ReminderPublisher
  port/AppointmentEventPublisher
  command/ScheduleAppointmentCommand, EditAppointmentCommand

infrastructure/
  persistence/   AppointmentEntity (@Table, @Id, @Version), AppointmentJdbcRepository (CrudRepository),
                 AppointmentMapper, AppointmentRepositoryImpl (implements the port;
                 uses @Version to decide INSERT vs UPDATE, id generated in the domain)
  messaging/     AppointmentReminderMessage, AppointmentEventMessage
                 rabbit/RabbitConfig + RabbitReminderPublisher
                 kafka/KafkaConfig + KafkaAppointmentEventPublisher
  security/      SecurityConfig — stateless resource server, validates JWT RS256 with public.pem,
                 converts the "role" claim → authority ROLE_<role>, @EnableMethodSecurity
  web/graphql/   AppointmentMutationController (@PreAuthorize does the coarse role gate)
                 DomainExceptionResolver (domain exception → GraphQL error FORBIDDEN/NOT_FOUND/BAD_REQUEST)
  config/        UseCaseConfig — wires the use cases (POJOs) via @Bean
```

**Where each rule is applied**

| Rule | Location |
|---|---|
| `scheduleAppointment` → `DOCTOR`/`NURSE`; `editAppointment` → `DOCTOR` | `@PreAuthorize` on the GraphQL resolver |
| Only the owning doctor (`doctorId`) edits | `Appointment.edit(callerDoctorId, …)` in the domain |
| `scheduledAt` must be in the future (create and edit) | `Appointment.schedule` / `Appointment.edit` |
| Persist before publishing | use cases |
| Caller identity passed as a **parameter** (never `SecurityContextHolder` in the core) | `EditAppointmentCommand.callerId` = `jwt.getSubject()` |

**GraphQL** (`src/main/resources/graphql/schema.graphqls`)

```graphql
type Mutation {
  scheduleAppointment(input: ScheduleInput!): Appointment!
  editAppointment(input: EditInput!): Appointment!
}
type Query { _ping: String! }
```

**Migration** (`src/main/resources/db/migration/V1__create_appointments.sql`) — `appointments`
table (`id`, `version`, `patient_id`, `doctor_id`, `scheduled_at`, `status`, `created_at`)
plus indexes by patient and by doctor.

---

## 12. Configuration

### 12.1 `application.yaml` (scheduling-service) — main keys

| Key | Default value |
|---|---|
| `server.port` | `8081` |
| `spring.datasource.url` | `jdbc:mysql://localhost:3306/scheduling_db` (root/root) |
| `spring.rabbitmq.*` | `localhost:5672`, guest/guest |
| `spring.kafka.bootstrap-servers` | `localhost:9092` |
| `security.jwt.public-key` | `classpath:public.pem` |
| `app.rabbit.exchange / queue / routing-key` | `appointment.reminders` / `reminder.queue` / `appointment.reminder` |
| `app.kafka.topic` | `appointment-events` |

### 12.2 RSA key pair (JWT RS256)

Generated once in `keys/` at the repository root (not committed — see `.gitignore`):

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out keys/private.pem
openssl rsa -in keys/private.pem -pubout -out keys/public.pem
```

- `public.pem` was **copied to `src/main/resources/`** of all four services (token validation).
- `private.pem` lives **only in `identity-service`** (signing). In production it would come from a
  secret/environment variable; versioning the **public** key is acceptable here, the **private** one is not
  — both are kept out of git via `.gitignore`.

---

## 13. How to run `scheduling-service`

### 13.1 Infrastructure (MySQL + RabbitMQ + Kafka)

`scheduling-service/docker-compose.yml` starts everything (Kafka in KRaft mode, no Zookeeper):

```bash
cd scheduling-service
docker compose up -d
```

- RabbitMQ UI: <http://localhost:15672> (guest/guest)
- MySQL: `localhost:3306`, database `scheduling_db`

### 13.2 The service

```bash
cd scheduling-service
./mvnw spring-boot:run     # starts on 8081
```

> ⚠️ Port **8081** may be taken by a `phpmyadmin` container on your machine —
> stop it or change `server.port` before running.

---

## 14. How to test `scheduling-service`

### 14.1 Automated tests

```bash
cd scheduling-service

./mvnw test                 # 29 unit tests (domain + use cases + adapters + resolver)
./mvnw verify               # + integration test (Testcontainers) + JaCoCo 80% gate
./mvnw allure:serve         # Allure report in the browser
```

Coverage: `target/site/jacoco/index.html`
(`*Application`, `infrastructure/config/**`, `infrastructure/security/**`, and the broker `*Config`
classes are excluded from the gate — framework wiring, only exercised by the integration test).

**What the tests cover**

| Test | Verifies |
|---|---|
| `AppointmentTest` | creation invariants; "only the owner edits"; date/status edit; rehydration doesn't revalidate |
| `ScheduleAppointmentUseCaseTest` | `save → publishReminder → publishCreated` order (InOrder); invalid command doesn't touch repo/brokers |
| `EditAppointmentUseCaseTest` | owner edits and publishes; non-owner is blocked (nothing saved/published); missing id → NotFound; invalid status → Invalid |
| `AppointmentPersistenceTest` | round-trip mapper; `@Version` null → INSERT, present → UPDATE |
| `PublisherAdaptersTest` | Rabbit sends only IDs to the right exchange/routing-key; Kafka uses `patientId` as key and `type` CREATED/UPDATED |
| `AppointmentMutationControllerTest` | resolver delegates to the use case, builds the command with the JWT's `callerId`, maps the view |
| `DomainExceptionResolverTest` | domain exception → `FORBIDDEN` / `NOT_FOUND` / `BAD_REQUEST` |
| `SchedulingIntegrationTest` | end to end: real MySQL/RabbitMQ/Kafka + real security + GraphQL over HTTP |

> **Integration test (`SchedulingIntegrationTest`)** needs a Docker daemon the docker-java client
> can reach. On some machines the Docker Desktop pipe returns HTTP 400 to Testcontainers (the
> `docker` CLI works fine; it's a Docker Desktop incompatibility, not a code issue).
> The test **self-skips** (`assumeTrue`) when Docker isn't reachable, so `./mvnw verify` still
> passes. On a "normal" Docker setup it runs unchanged. It signs its own RS256 tokens with an
> in-memory key pair (`support/SecurityTestConfig`).

### 14.2 Manual test (GraphiQL)

1. Start the infrastructure and the service (section 13).
2. Open <http://localhost:8081/graphiql>.
3. Since the route requires a JWT and `identity-service` doesn't exist yet, generate a test RS256
   token signed with `keys/private.pem` (minimal claims: `sub`, `role`, `exp`). Header in GraphiQL:

   ```json
   { "Authorization": "Bearer YOUR_TOKEN" }
   ```

4. Schedule:

   ```graphql
   mutation {
     scheduleAppointment(input: {
       patientId: "pat-1", doctorId: "doc-1", scheduledAt: "2030-12-01T10:00:00"
     }) { id status }
   }
   ```

5. Edit (owning doctor only, role `DOCTOR`):

   ```graphql
   mutation {
     editAppointment(input: { appointmentId: "<id>", status: "CANCELLED" }) { id status }
   }
   ```

6. **Confirm messaging:**
   - RabbitMQ UI (<http://localhost:15672>) → *Queues* tab → `reminder.queue` should have received
     a message (`AppointmentReminderMessage`).
   - Kafka → topic `appointment-events` should have an `AppointmentCreated` / `AppointmentUpdated`
     with key = `patientId`:

     ```bash
     docker exec -it kafka /opt/kafka/bin/kafka-console-consumer.sh \
       --bootstrap-server localhost:9092 --topic appointment-events --from-beginning \
       --property print.key=true
     ```
   - Database: `SELECT * FROM scheduling_db.appointments;`

---

## 15. Next steps

1. **`notification-service`** — `@RabbitListener` on `reminder.queue`, retry + DLQ in `application.yaml`,
   `UserDirectory` port (HTTP adapter + Caffeine cache for identity), send channel (log).
2. **`history-service`** — `@KafkaListener` on `appointment-events` with idempotency
   (`processed_events`), read model, GraphQL queries `history` / `futureAppointments`
   ("patient sees only their own" rule inside the use case).
3. **`identity-service`** — login (BCrypt) + JWT RS256 issuance with `private.pem`,
   `GET /users/{id}` for the other services to query.
4. Versioned Insomnia/Postman collection; per-service `README.md`.
