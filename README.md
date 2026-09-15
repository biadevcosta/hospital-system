# Hospital Appointment System

Backend de **4 microserviços independentes** para agendamento de consultas hospitalares,
histórico de pacientes e lembretes automáticos, com controle de acesso por papel e comunicação
assíncrona entre serviços.

Projeto de pós-graduação: Clean Architecture, CQRS, integração orientada a eventos e segurança
JWT stateless num backend multi-serviço real.

---

## 📎 Comece por aqui

> Antes de entrar em qualquer serviço, estes dois arquivos dão o panorama completo do sistema.

<table>
<tr>
<td width="50%" valign="top">

### 🗺️ [`arquitetura-sistemas.drawio`](arquitetura-sistemas.drawio)

[![Abrir diagrama](https://img.shields.io/badge/📐_ABRIR-arquitetura--sistemas.drawio-2b6cb0?style=for-the-badge)](arquitetura-sistemas.drawio)

Diagrama de **componentes e fluxo de dados** — os 4 serviços, os 2 brokers (RabbitMQ/Kafka),
os 4 bancos e o sentido de cada seta. O fluxo end-to-end (login → agendamento → notificação →
histórico) na prática está no [`guia-de-testes.md`](guia-de-testes.md), com requests e
prints reais.

Abra em **[app.diagrams.net](https://app.diagrams.net)** (File → Open from → Device).

</td>
<td width="50%" valign="top">

### 🧪 [`guia-de-testes.md`](guia-de-testes.md)

[![Abrir guia](https://img.shields.io/badge/▶️_ABRIR-guia--de--testes.md-2f855a?style=for-the-badge)](guia-de-testes.md)

Como subir o sistema e testar na mão, passo a passo, do `docker compose up` até a consulta
aparecer no histórico:
- Requests prontos pra copiar e colar (Swagger + GraphiQL).
- Resposta esperada em cada passo.
- Cobre setup, login, cadastro de usuários, agendamento, e-mail simulado, histórico, edição/
  cancelamento, consulta direta no banco e o relatório de testes (Allure).

</td>
</tr>
</table>

> O restante deste README explica **por que** o sistema é assim: regras de negócio, comunicação
> entre serviços e as decisões por trás do diagrama.

---

| Serviço | Repositório (submódulo) | Porta | Responsabilidade |
|---|---|---|---|
| [`identity-service`](identity-service/README.md) | `biadevcosta/identity-service` | 8080 | Autenticação, emissão de JWT (RS256), gestão de usuários |
| [`scheduling-service`](scheduling-service/README.md) | `biadevcosta/scheduling-service` | 8081 | Escreve consultas (fonte da verdade), publica eventos |
| [`notification-service`](notification-service/README.md) | `biadevcosta/notification-service` | 8082 | Consome lembretes, resolve o contato do paciente, envia |
| [`history-service`](history-service/README.md) | `biadevcosta/history-service` | 8083 | Lado de leitura (CQRS): consome eventos, serve consultas de histórico |

Cada serviço tem seu próprio README, com endpoints, exemplos de request/response e como
rodar/testar isoladamente. Este documento cobre o sistema como um todo.

---

## 1. Arquitetura

Todo serviço segue **Clean Architecture**, três camadas, mesma regra de dependência: sempre de
fora pra dentro.

```
<service>/
├── domain/            entidades e regras de negócio — sem imports de framework
├── application/       casos de uso (classes simples) + ports (interfaces)
└── infrastructure/    adapters concretos: GraphQL/REST, persistência, mensageria,
                        segurança, configuração (@Configuration)
```

- `domain`: entidades e invariantes. Sem Spring, sem anotações JPA/JDBC, sem tipos de
  HTTP/mensageria.
- `application`: casos de uso simples, dependências via construtor. Ports (interfaces) —
  repositório, publisher, emissor de token — são definidas aqui, nunca em `infrastructure`.
- `infrastructure`: implementa cada port com um adapter concreto (repositório JDBC, publisher
  Rabbit/Kafka, cliente HTTP), expõe a API (resolver GraphQL / controller REST) e faz a fiação
  com `@Configuration`.

Testar um caso de uso não exige Spring, banco real nem broker real — todo colaborador é uma
interface substituível por fake. Trocar um adapter (outro provedor de e-mail, outro broker) não
toca `domain` nem `application`.

Stack: **Java 21, Spring Boot 4.1, Spring Data JDBC (sem JPA/Hibernate), MySQL, Flyway, Spring
for GraphQL, Spring Security (JWT RS256), RabbitMQ, Kafka.** Testes: JUnit 5 + Mockito + AssertJ
(unitários), Testcontainers (integração). Gate de cobertura: **80% de linha (JaCoCo)** por
serviço.

Decisões do sistema:

- **CQRS:** `scheduling-service` escreve; `history-service` lê do próprio read model, montado de
  forma assíncrona a partir dos eventos.
- **Dois brokers, dois papéis:** RabbitMQ carrega uma tarefa pontual (enviar este lembrete);
  Kafka carrega um log de eventos ordenado e replayable (o feed de histórico).
- **JWT stateless (RS256):** `identity-service` assina com chave privada; os demais validam
  localmente com a chave pública correspondente — sem sessão compartilhada, sem gateway, sem
  chamada de rede pra checar validade do token.
- **Banco por serviço:** cada serviço é dono do próprio schema; nenhum lê o banco de outro
  diretamente.
- **Dados de usuário resolvidos sob demanda:** quem precisa de nome/e-mail chama
  `identity-service` via HTTP (com cache local) em vez de copiar esse dado pro próprio banco.

---

## 2. Regras de negócio

### 2.1 Papéis e permissões

Três papéis: `DOCTOR`, `NURSE`, `PATIENT` (mais `ADMIN`, interno ao `identity-service`, só pra
cadastrar usuários).

| Capacidade | DOCTOR | NURSE | PATIENT |
|---|:---:|:---:|:---:|
| Criar consulta | ✅ | ✅ | ❌ |
| Editar consulta | ✅ (só o dono) | ❌ | ❌ |
| Ler histórico | ✅ (qualquer paciente) | ✅ (qualquer paciente) | ✅ (só o próprio) |
| Ler consultas futuras | ✅ (qualquer paciente) | ✅ (qualquer paciente) | ✅ (só o próprio) |

| Operação | Papéis permitidos |
|---|---|
| `scheduleAppointment` (mutation) | `DOCTOR`, `NURSE` |
| `editAppointment` (mutation) | `DOCTOR`, e precisa ser o dono da consulta |
| `history` / `futureAppointments` (queries) | `DOCTOR`, `NURSE`, `PATIENT` |

### 2.2 Ciclo de vida da consulta

- Status: `SCHEDULED`, `COMPLETED`, `CANCELLED`.
- Toda consulta nasce `SCHEDULED`.
- **Consulta futura** = status `SCHEDULED` e `scheduledAt` no futuro.
- **Histórico** = todas as consultas do paciente, qualquer status.

### 2.3 Regras de posse

- **Só o dono edita:** o médico da consulta (`doctorId`) é o dono. Só ele edita — validado no
  domínio/caso de uso, não só pela role.
- **Paciente só vê o próprio dado:** para um `PATIENT`, o `patientId` usado na consulta vem do
  **token autenticado**, nunca de um argumento enviado pelo cliente. Para `DOCTOR`/`NURSE`, o
  `patientId` do argumento é respeitado.

### 2.4 Invariantes (rejeitadas com exceção de domínio)

- `scheduledAt` precisa estar no **futuro**, na criação e na edição.
- `patientId` e `doctorId` são obrigatórios.
- Transição de status só aceita valor válido do enum.

### 2.5 Onde vive a autorização

- **Checagem de papel** (`hasRole`/`hasAnyRole`) → no adapter, via `@PreAuthorize` no resolver
  GraphQL / controller REST.
- **Regra fina** (posse, "paciente só vê o próprio") → dentro do **caso de uso**, que recebe a
  identidade do chamador (papel, userId/patientId) como parâmetro simples. O núcleo nunca lê
  `SecurityContextHolder` diretamente.

### 2.6 Mensageria

- No `scheduling-service`, em criação e edição, a ordem é sempre: validar → **persistir** →
  **então** publicar. Nada é publicado antes de salvar.
  - RabbitMQ — `AppointmentReminder` (só ids: `appointmentId`, `patientId`, `scheduledAt`).
  - Kafka, tópico `appointment-events` — `AppointmentCreated`/`AppointmentUpdated` (ids + status;
    chave de partição = `patientId`, garantindo ordem por paciente).
- `notification-service` consome o lembrete, resolve nome/e-mail do paciente no
  `identity-service` (cache) e envia (hoje só loga; um provedor real é um adapter plugável).
- `history-service` consome `appointment-events` de forma **idempotente** (tabela
  `processed_events` ignora `eventId` repetido, já que o Kafka pode reentregar), atualiza o read
  model, e resolve nomes no `identity-service` no momento da consulta (sempre o nome atual, nunca
  uma cópia velha).

---

## 3. Comunicação entre serviços

Regra: **cliente → serviço é síncrono** (HTTP/GraphQL); **serviço → serviço é assíncrono**
(broker) — única exceção é a busca de dados de usuário no `identity-service`, síncrona e
cacheada, fora do caminho crítico.

| De → Para | Tipo | Mecanismo | Payload |
|---|---|---|---|
| Cliente → Identity | síncrono | HTTP REST | credenciais → JWT; cadastro de usuário |
| Cliente → Scheduling | síncrono | GraphQL + JWT | mutation de agendar/editar |
| Cliente → History | síncrono | GraphQL + JWT | query de histórico/futuras |
| Scheduling → Notification | assíncrono | RabbitMQ `reminder.queue` | lembrete (só ids) |
| Scheduling → History | assíncrono | Kafka `appointment-events` | consulta criada/atualizada (ids + status) |
| Notification → Identity | síncrono (cache) | HTTP | nome/e-mail do paciente |
| History → Identity | síncrono (cache) | HTTP | resolução de nome de usuário |

### Banco por serviço

| Banco | Dono | Tabelas |
|---|---|---|
| `identity_db` | identity-service | `users`, `refresh_tokens` |
| `scheduling_db` | scheduling-service | `appointments` |
| `notification_db` | notification-service | `processed_reminders` (idempotência) |
| `history_db` | history-service | `appointment_history`, `processed_events` |

### Brokers

| Broker | Nome | Fluxo | Semântica |
|---|---|---|---|
| RabbitMQ | `reminder.queue` (+ `reminder.dlq`) | scheduling → notification | tarefa pontual, 3 tentativas e depois dead-letter |
| Kafka | `appointment-events` (chave = `patientId`) | scheduling → history | ordenado por paciente, replayable, consumidor idempotente |

---

## 4. Segurança — JWT assinado com RS256

Só o `identity-service` **emite** token; os demais só **verificam**. Funciona porque RS256 é
assinatura assimétrica: a **chave privada** assina, e só a **chave pública** correspondente
verifica — a pública sozinha não serve pra forjar assinatura.

- `identity-service` guarda `private.pem` e assina cada access token após login.
- `scheduling-service` e `history-service` guardam uma cópia de `public.pem` e validam a
  assinatura **localmente**, sem chamada de rede de volta ao `identity-service` — escrita e
  leitura ficam rápidas e desacopladas. `identity-service` só é chamado de forma síncrona pra
  buscar perfil (nome/e-mail), nunca pra perguntar "esse token é válido?".

Claims do token: `sub` (userId), `role`, `patientId` (só para pacientes), mais `iss`, `aud`,
`iat`, `exp`.

Gerar o par de chaves (uma vez só; `private.pem` é git-ignored em todo lugar):

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out keys/private.pem
openssl rsa -in keys/private.pem -pubout -out keys/public.pem
```

`public.pem` é copiada pro `src/main/resources/` de cada serviço; `private.pem` fica só no
`identity-service`.

---

## 5. Layout do repositório

Cada um dos 4 serviços é um repositório Git próprio, ligado a este como **submódulo**
(`github.com/biadevcosta/<service>`), com `Dockerfile`, `pom.xml`, testes e README próprios.

```
hospital-system/
├── docker-compose.yml          # sobe os 4 serviços + infra juntos
├── arquitetura-sistemas.drawio # diagrama de componentes e fluxo de dados
├── identity-service/           # submódulo
├── scheduling-service/         # submódulo
├── notification-service/       # submódulo
└── history-service/            # submódulo
```

---

## 6. Como testar

Cada serviço é testável isoladamente — detalhes no README de cada um:

```bash
cd <service>
./mvnw test       # só testes unitários — sem Docker
./mvnw verify     # + integração (Testcontainers) + gate JaCoCo 80%
```

Sem Docker disponível, os testes de integração se auto-pulam em vez de falhar — `./mvnw verify`
sempre completa. Relatório de cobertura: `target/site/jacoco/index.html` em cada serviço.

Cada serviço também expõe um explorador OpenAPI/GraphiQL (ver seu README); `identity-service` e
`scheduling-service` incluem ainda uma coleção Insomnia importável.

Guia manual completo, com requests e respostas esperadas: [`guia-de-testes.md`](guia-de-testes.md).
