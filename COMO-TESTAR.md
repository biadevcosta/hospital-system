# Como testar o sistema (passo a passo)

Guia prático pra subir e exercitar os serviços via Docker + Insomnia + as UIs de banco e de fila.
Cobre hoje **identity-service** e **scheduling-service**; `notification-service` e `history-service`
entram nas seções 9–10 conforme forem testados.

> O `README.md` da raiz é o documento de arquitetura e regras. Este arquivo é só o roteiro de teste.

---

## Mapa rápido

| Serviço | App | Doc da API | phpMyAdmin (root/root) | UI de fila |
|---|---|---|---|---|
| **identity-service** | http://localhost:8080 | Swagger: http://localhost:8080/swagger-ui.html · JSON: `/v3/api-docs` | http://localhost:8082 → `identity_db` | — |
| **scheduling-service** | http://localhost:8085 (container 8081) | GraphiQL: http://localhost:8085/graphiql · SDL: `/graphql/schema` | http://localhost:8083 → `scheduling_db` | Kafka UI: http://localhost:8084 · RabbitMQ: http://localhost:15672 (guest/guest) |

**Credenciais**
- Admin semeado (identity): `admin@hospital.local` / `admin12345`
- Doctor/Nurse: você cria no passo 4 (ex.: `doc@hospital.local` / `secret12345`).

**Collections Insomnia**
- `identity-service/identity.insomnia.json`
- `scheduling-service/scheduling.insomnia.json`

---

## 0. Pré-requisitos

- **Docker Desktop** aberto e rodando (`docker ps` responde).
- **Insomnia** instalado.
- **Chaves RSA**: `identity-service/src/main/resources/private.pem` tem que existir, e o `public.pem`
  precisa ser o mesmo par nos 3 serviços (identity assina o JWT, scheduling/history validam). Já está
  configurado. Se clonar do zero, regere seguindo `identity-service/README.md` → *RSA key pair*.

---

## 1. Subir os containers

Suba o **identity primeiro** — o scheduling valida os tokens que o identity emite.

```bash
cd identity-service
docker compose up --build -d
```

Sobe: `identity-service` (:8080), `mysql-identity` (:3309), `phpmyadmin-identity` (:8082).
A **primeira** build baixa as dependências Maven (~4 min). Confira:

```bash
curl localhost:8080/actuator/health      # {"status":"UP"}
```

Depois o **scheduling**:

```bash
cd ../scheduling-service
docker compose up --build -d
```

Sobe: `scheduling-service` (host **:8085** → container 8081), `mysql-scheduling` (:3306),
`rabbitmq` (:5672 / UI :15672), `kafka` (:9092), `kafka-ui` (:8084), `phpmyadmin-scheduling` (:8083).

```bash
curl localhost:8085/actuator/health      # {"status":"UP"}
```

> **Por que :8085 e não :8081?** A 8081 estava ocupada por outro projeto na máquina. Todos os
> endpoints e o GraphiQL do scheduling usam **:8085** no Docker. Rodando no host (`./mvnw
> spring-boot:run`) volta a ser 8081.

Parar depois: `docker compose down` (ou `down -v` pra apagar também o volume do MySQL).

---

## 2. Importar as collections no Insomnia

Insomnia → menu → **Import** → *From File* → selecione:

1. `identity-service/identity.insomnia.json`
2. `scheduling-service/scheduling.insomnia.json`

Cada uma vem com um *Base Environment* já preenchido (URLs, credenciais). Os tokens **encadeiam
sozinhos**: você roda o login uma vez e os outros requests puxam o `accessToken` da resposta.

---

## 3. Ver a doc e o banco do identity

- **Swagger UI**: http://localhost:8080/swagger-ui.html — lista `/auth/login`, `/auth/refresh`,
  `/users`, `/users/{id}`. Depois de ter um token (passo 4), clique **Authorize**, cole o
  `accessToken` e teste o `POST /users` direto pela página.
- **OpenAPI JSON**: http://localhost:8080/v3/api-docs
- **phpMyAdmin**: http://localhost:8082 (login automático `root`/`root`) → base **`identity_db`** →
  tabela **`users`**. Aqui você vê os usuários criados e seus papéis.

---

## 4. Criar usuários e pegar um token (collection *identity-service*)

Na collection **identity-service**, rode em ordem:

1. **`1 · auth → POST /auth/login (admin)`** → `200`. Devolve `accessToken` (ADMIN) — os requests de
   `2 · users` já o reaproveitam automaticamente.
2. **`2 · users (admin) → POST /users (register DOCTOR)`** → `201`. Cria `doc@hospital.local` /
   `secret12345`, role `DOCTOR`. (Rodar de novo dá `409` — e-mail já usado; tudo bem.)
3. *(opcional)* **`POST /users (register NURSE)`** → `201`.
4. **`1 · auth → POST /auth/login (registered doctor)`** → `200`. **Esse** token (role `DOCTOR`) é o
   que o scheduling aceita para agendar.

Confirme no phpMyAdmin (`identity_db.users`) ou em **`GET /users/{id}`**.

Testes de erro que valem ver: `POST /users` sem token → `401`; com token não-admin → `403`;
`POST /auth/refresh` com o mesmo token 2× → a 2ª dá `401` (rotação).

---

## 5. Agendar uma consulta (collection *scheduling-service*)

Na collection **scheduling-service**:

1. **`0 · tokens (identity-service) → POST /auth/login (doctor)`** → `200`.
   (Bate no identity em `identity_url` = :8080; o token encadeia nos requests seguintes.)
2. **`2 · queries & mutations → mutation scheduleAppointment (DOCTOR)`** → `200`,
   `status: SCHEDULED`, retorna um `id`.
   Nesse único request o serviço: **(1)** grava a linha no MySQL → **(2)** publica um *reminder* no
   RabbitMQ → **(3)** publica um evento `AppointmentCreated` no Kafka.
3. **`scheduleAppointment (NURSE)`** também funciona — rode `login (nurse)` antes.
4. **`editAppointment`** — regra de posse: só o **doctor dono** edita (o `doctorId` do agendamento
   tem que ser igual ao `sub` do JWT). Use o par **`scheduleAppointment (owner-linked)`** +
   **`editAppointment (as owner)`** com a variável de ambiente `caller_doctor_id` = id do doctor no
   identity (o `sub` do token; dá pra ver em jwt.io ou no `identity_db.users`).

Erros que valem ver (pasta `3 · error cases`): sem token → `401`; token ADMIN → `FORBIDDEN`;
data no passado → `BAD_REQUEST`; id inexistente → `NOT_FOUND`; editar sem ser dono → `FORBIDDEN`.

### Alternativa: GraphiQL no navegador

http://localhost:8085/graphiql → aba **Headers** (painel de baixo) → cole
`{ "Authorization": "Bearer <accessToken>" }` → aperte **F5** (ele reintrospecta o schema com o
header). Sem o header, a página abre com erro "No Schema Available" porque a introspection exige JWT.
O schema em texto puro (sem token) está em http://localhost:8085/graphql/schema.

---

## 6. Ver a consulta no banco

**phpMyAdmin do scheduling**: http://localhost:8083 (`root`/`root`) → base **`scheduling_db`** →
tabela **`appointments`**. Cada `scheduleAppointment` = 1 linha (`id`, `patient_id`, `doctor_id`,
`scheduled_at`, `status`, `created_at`). `editAppointment` altera a linha existente.

CLI equivalente:
```bash
docker exec mysql-scheduling mysql -uroot -proot scheduling_db \
  -e "SELECT id, patient_id, doctor_id, scheduled_at, status FROM appointments ORDER BY created_at DESC;"
```

---

## 7. Ver as mensagens nas filas

O scheduling publica em **dois** brokers a cada agendamento. Como `notification-service` e
`history-service` ainda não estão rodando, nada consome — as mensagens ficam acumuladas, ótimo pra
inspecionar.

### Kafka — evento para o history-service

- **Kafka UI**: http://localhost:8084 → **Topics** → **`appointment-events`** → aba **Messages**.
  - **Key** = `patientId` (mesma key → mesma partição → ordem por paciente)
  - **Value** = JSON do `AppointmentEventMessage` (`type`, `eventId`, `appointmentId`, `patientId`,
    `doctorId`, `scheduledAt`, `status`)
  - **Header** `__TypeId__` = FQN do record (o consumer usa pra desserializar)
  - `type` = `AppointmentCreated` no agendar, `AppointmentUpdated` no editar
- CLI:
  ```bash
  docker exec kafka /opt/kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server localhost:9092 --topic appointment-events \
    --from-beginning --timeout-ms 5000 --property print.key=true
  ```

### RabbitMQ — reminder para o notification-service

- **Management UI**: http://localhost:15672 (`guest`/`guest`) → **Queues and Streams** →
  **`reminder.queue`** → seção **Get messages** → **Requeue: Yes** → *Get Message(s)* (assim você
  espia sem consumir).
  - Payload = `{ "appointmentId", "patientId", "scheduledAt" }` (só ids)
  - Exchange `appointment.reminders` (direct), routing key `appointment.reminder`, com DLQ
    `reminder.dlq`
- CLI:
  ```bash
  docker exec rabbitmq rabbitmqctl list_queues name messages
  ```

---

## 8. O que o scheduling faz de mensageria (resumo da config)

| Onde | Config |
|---|---|
| `application.yaml` | Kafka: `value-serializer: JacksonJsonSerializer`, `spring.json.add.type.headers: true`, `key-serializer: StringSerializer` · `app.kafka.topic: appointment-events` · `app.rabbit.exchange/queue/routing-key: appointment.reminders / reminder.queue / appointment.reminder` |
| `docker-compose.yml` | `SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:19092` · `SPRING_RABBITMQ_HOST: rabbitmq` (nomes da rede interna) |
| `KafkaConfig` | cria o tópico `appointment-events` no startup: **3 partições, replicação 1** |
| `KafkaAppointmentEventPublisher` | `send(topic, patientId, evento)` — key = `patientId` |
| `RabbitConfig` | declara exchange + `reminder.queue` (durável, com dead-letter → `reminder.dlq`) + binding; `RabbitTemplate` com `JacksonJsonMessageConverter` |
| `RabbitReminderPublisher` | `convertAndSend(exchange, routingKey, msg)` — só ids |
| `ScheduleAppointmentUseCase` | ordem fixa: **persiste → reminder (Rabbit) → evento (Kafka)**. Nada sai antes do commit |

---

## 9. notification-service — *a documentar*

- [ ] Subir junto (consome `reminder.queue`)
- [ ] Como ver o lembrete sendo processado / enviado
- [ ] Efeito no contador da `reminder.queue` (deve zerar conforme consome)

## 10. history-service — *a documentar*

- [ ] Subir junto (consome `appointment-events` no Kafka)
- [ ] Consumer group aparecendo no Kafka UI
- [ ] Queries de histórico (`history`, `futureAppointments`) + regra "paciente só vê o próprio"
- [ ] Banco `history_db` (read model) no phpMyAdmin

> Quando for testar esses dois, a gente preenche as seções acima.

---

## Reset rápido

```bash
cd identity-service   && docker compose down -v   # -v apaga o MySQL → volta só o admin semeado
cd ../scheduling-service && docker compose down -v # apaga MySQL + volume; Kafka/Rabbit recriam limpos
```
