# Como testar o sistema (passo a passo)

Guia prático pra subir e exercitar os serviços via Docker + Insomnia + as UIs de banco e de fila.
Cobre os **4 serviços** (identity, scheduling, notification, history) subindo juntos com **um único
`docker compose up`** a partir da raiz do repo.

> O `README.md` da raiz é o documento de arquitetura e regras. Este arquivo é só o roteiro de teste.

---

## Mapa rápido

| Serviço | App | Doc da API |
|---|---|---|
| **identity-service** | http://localhost:8080 | Swagger: http://localhost:8080/swagger-ui.html · JSON: `/v3/api-docs` |
| **scheduling-service** | http://localhost:8085 (container 8081) | GraphiQL: http://localhost:8085/graphiql · SDL: `/graphql/schema` |
| **notification-service** | http://localhost:8082 | sem API própria — só consome RabbitMQ |
| **history-service** | http://localhost:8083 | GraphiQL: http://localhost:8083/graphiql · SDL: `/graphql/schema` |

| Infra compartilhada | URL |
|---|---|
| Adminer (um só painel pros 4 bancos) | http://localhost:8090 (servidor: `mysql-identity` / `mysql-scheduling` / `mysql-notification` / `mysql-history`, `root`/`root`) |
| RabbitMQ management | http://localhost:15672 (`guest`/`guest`) |
| Kafka UI | http://localhost:8084 |

**Credenciais**
- Admin semeado (identity): `admin@hospital.local` / `admin12345`
- Doctor/Nurse/Patient: você cria no passo 4 (ex.: `doc@hospital.local` / `secret12345`).

**Collections Insomnia**
- `identity-service/identity.insomnia.json`
- `scheduling-service/scheduling.insomnia.json`

---

## 0. Pré-requisitos

- **Docker Desktop** aberto e rodando (`docker ps` responde).
- **Insomnia** instalado.
- **Chaves RSA**: `identity-service/src/main/resources/private.pem` tem que existir, e o `public.pem`
  precisa ser o mesmo par nos 4 serviços (identity assina o JWT, os outros só validam). Já está
  configurado. Se clonar do zero, regere seguindo `identity-service/README.md` → *RSA key pair*.
- Nenhuma conta/API key de provedor de e-mail é necessária — o envio é **simulado** (log). Ver §9.

---

## 1. Subir os containers

Um comando só, na **raiz do repo** — sobe os 4 apps + 4 MySQL + RabbitMQ + Kafka (compartilhados,
um container só de cada) + Kafka UI + Adminer, tudo na mesma rede Docker:

```bash
docker compose up --build -d
```

A **primeira** build baixa as dependências Maven dos 4 serviços (alguns minutos). Confira:

```bash
curl localhost:8080/actuator/health      # identity  — {"status":"UP"}
curl localhost:8085/actuator/health      # scheduling
curl localhost:8082/actuator/health      # notification
curl localhost:8083/actuator/health      # history
```

> **Por que scheduling é :8085 e não :8081?** A 8081 estava ocupada por outro projeto nesta
> máquina; o mapeamento host→container é `8085:8081`. Rodando fora do Docker (`./mvnw
> spring-boot:run`) volta a ser 8081. Os outros três (identity 8080, notification 8082,
> history 8083) usam a porta "de verdade" tanto no host quanto dentro do container.

> **Testar um serviço isolado?** Cada submódulo ainda tem seu próprio `docker-compose.yml`
> (`cd scheduling-service && docker compose up --build -d`, etc.) — útil pra debugar um serviço
> sozinho, mas **não** suba mais de um desses ao mesmo tempo: cada um declara seu próprio
> RabbitMQ/Kafka/porta e colide com os outros. Pra ponta a ponta, use sempre o compose da raiz.

Parar depois: `docker compose down` (ou `down -v` pra apagar também os volumes dos 4 MySQL).

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

Confirme no Adminer (http://localhost:8090, servidor `mysql-identity`, base `identity_db` →
tabela `users`) ou em **`GET /users/{id}`**.

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

**Adminer**: http://localhost:8090 (servidor `mysql-scheduling`, `root`/`root`) → base
**`scheduling_db`** → tabela **`appointments`**. Cada `scheduleAppointment` = 1 linha (`id`,
`patient_id`, `doctor_id`, `scheduled_at`, `status`, `created_at`). `editAppointment` altera a
linha existente.

CLI equivalente:
```bash
docker exec mysql-scheduling mysql -uroot -proot scheduling_db \
  -e "SELECT id, patient_id, doctor_id, scheduled_at, status FROM appointments ORDER BY created_at DESC;"
```

---

## 7. Ver as mensagens nas filas

O scheduling publica em **dois** brokers a cada agendamento. Como `notification-service` e
`history-service` já estão rodando (subiram juntos no passo 1), o consumo é **quase imediato** —
se você for rápido no Kafka UI / RabbitMQ UI ainda dá pra ver a mensagem passando; senão, ela já
vai ter sido processada e vai aparecer direto nos passos 9/10 (histórico e e-mail).

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

## 9. notification-service — o lembrete por e-mail (simulado)

O `notification-service` consome `reminder.queue` (RabbitMQ), resolve nome/e-mail do paciente via
`GET /users/{id}` no identity, e "envia" o lembrete — hoje isso é **simulado**: `LoggingEmailSender`
só loga o e-mail em vez de chamar um provedor real (a integração com um provedor de verdade, tipo
Brevo, fica pra uma segunda etapa — a porta `EmailSender` já existe pronta pra essa troca).

1. **RabbitMQ UI** (http://localhost:15672, `guest`/`guest`) → **Queues and Streams** →
   `reminder.queue` → o contador de mensagens deve zerar logo depois do `scheduleAppointment` do
   passo 5 (consumo é automático).
2. Log do container: `docker logs notification-service` → procure por
   `Simulated e-mail sent to <nome> <<e-mail>> — subject: "..."`. Isso confirma que a mensagem
   chegou certinho na fila, foi resolvida no identity e passou pelo caminho inteiro do use case.
3. Duplicar o mesmo `appointmentId` (reenviando a mensagem manualmente pelo RabbitMQ UI) não gera
   uma segunda linha de log — dedup via tabela `processed_reminders` (veja no Adminer, servidor
   `mysql-notification`).
4. Publicar com um `patientId` que **não existe** no identity → sem provider pra falhar, a única
   forma de cair na `reminder.dlq` agora é a busca do paciente falhar (404 no identity) — depois do
   retry (3 tentativas, 2s de intervalo), a mensagem aparece em **Queues and Streams → reminder.dlq**.

## 10. history-service — o histórico via GraphQL

O `history-service` consome `appointment-events` (Kafka), monta o read model
(`appointment_history`) e serve duas queries GraphQL.

1. **Kafka UI** (http://localhost:8084) → **Consumers** → deve aparecer o grupo **`history`**
   com lag baixo/zero — sinal de que ele está consumindo o tópico `appointment-events`.
2. **GraphiQL**: http://localhost:8083/graphiql → aba **Headers** → cole
   `{ "Authorization": "Bearer <accessToken>" }` (o mesmo token do passo 5) → **F5**.
   ```graphql
   query {
     history(patientId: "<patientId>") {
       id patientName doctorName scheduledAt status
     }
   }
   ```
   Depois de um `scheduleAppointment` (passo 5), a linha deve aparecer aqui em poucos segundos,
   com `patientName`/`doctorName` já resolvidos pelo identity. `editAppointment` (mudança de
   status) atualiza a **mesma** linha, sem duplicar.
3. `futureAppointments(patientId: ...)` só devolve o que está `SCHEDULED` com data no futuro.
4. Regra "paciente só vê o próprio": logue como o **paciente** (token com `role: PATIENT`, veja
   `identity_db.users`/passo 4) e chame `history(patientId: "outro-id-qualquer")` — o resultado
   volta com o histórico do **próprio** paciente do token, ignorando o argumento.
5. Banco: Adminer (http://localhost:8090, servidor `mysql-history`) → tabelas
   `appointment_history` (read model) e `processed_events` (idempotência do consumer).

---

## Reset rápido

```bash
docker compose down -v   # para tudo e apaga os 4 volumes de MySQL — volta só o admin semeado no identity
```
