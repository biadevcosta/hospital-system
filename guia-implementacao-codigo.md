# Guia de implementação — passo a passo com código

Sistema hospitalar · 4 serviços · Java 21 · Spring Boot 4.1 · Clean Architecture

---

## Como usar este guia

- O **`scheduling-service` é a referência completa** — todas as camadas com código. Os outros três serviços **repetem a mesma estrutura** de Clean Architecture, então mostro deles apenas o **setup + o código distinto** (JWT RS256, listener Kafka, listener Rabbit + DLQ, cache, queries GraphQL). Onde disser "igual ao molde", copie o padrão do scheduling.
- Pacote base usado nos exemplos: `com.biadevcosta.<servico>`. Ajuste ao seu.
- Ordem sugerida de leitura/construção: **scheduling → notification → history → identity**.
- Portas: identity `8080`, scheduling `8081`, notification `8082`, history `8083`.

> **Regra de ouro da Clean Architecture (use para revisar você mesma):** abra qualquer classe de `domain` ou `application` e olhe os `import`. Se aparecer `org.springframework`, `RestClient`, `RabbitTemplate`, `KafkaTemplate` ou qualquer classe concreta de infra, há vazamento. Todo colaborador externo de um caso de uso deve ser uma **interface (porta)** que você definiu no core — nunca uma classe concreta de fora. Exemplos de portas neste guia: `AppointmentRepository`, `ReminderPublisher`, `AppointmentEventPublisher`, `UserDirectory`, `PasswordHasher`, `TokenIssuer`, `HistoryRepository`, `ProcessedEventStore`.

---

## 0. Infraestrutura local (uma vez)

Suba MySQL, RabbitMQ e Kafka com Docker. Você desenvolve os serviços na IDE apontando pra eles.

```bash
# MySQL (cria os 3 databases via script de init, veja abaixo)
docker run --name mysql-hospital -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=root \
  -d mysql:8

# RabbitMQ com UI de management (http://localhost:15672 — guest/guest)
docker run --name rabbit -p 5672:5672 -p 15672:15672 \
  -d rabbitmq:3-management

# Kafka em modo KRaft (sem Zookeeper)
docker run --name kafka -p 9092:9092 \
  -e KAFKA_CFG_NODE_ID=0 \
  -e KAFKA_CFG_PROCESS_ROLES=controller,broker \
  -e KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@localhost:9093 \
  -e KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093 \
  -e KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 \
  -e KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER \
  -e KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT \
  -d bitnami/kafka:latest
```

Crie os três bancos:

```bash
docker exec -i mysql-hospital mysql -uroot -proot -e \
"CREATE DATABASE IF NOT EXISTS identity_db;
 CREATE DATABASE IF NOT EXISTS scheduling_db;
 CREATE DATABASE IF NOT EXISTS history_db;"
```

### Par de chaves RSA (para o JWT RS256)

O `identity-service` assina com a **privada**; os outros validam com a **pública**.

```bash
# gera chave privada e pública em formato PEM
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

Guarde `private.pem` só no identity; distribua `public.pem` aos demais (em `src/main/resources/`). Em produção viriam de variável de ambiente/secret — no challenge, versionar a **pública** é aceitável; a **privada** fica fora do Git.

---

# 1. scheduling-service (referência completa)

Dono das consultas. Escreve via GraphQL, valida JWT, publica lembrete (Rabbit) e evento (Kafka).

## 1.1 Dependências (`pom.xml`)

```xml
<dependencies>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-web</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-graphql</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-security</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-oauth2-resource-server</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-data-jdbc</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-validation</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-actuator</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-amqp</artifactId></dependency>
  <dependency><groupId>org.springframework.kafka</groupId><artifactId>spring-kafka</artifactId></dependency>
  <dependency><groupId>com.mysql</groupId><artifactId>mysql-connector-j</artifactId><scope>runtime</scope></dependency>
  <dependency><groupId>org.flywaydb</groupId><artifactId>flyway-core</artifactId></dependency>
  <dependency><groupId>org.flywaydb</groupId><artifactId>flyway-mysql</artifactId></dependency>
  <dependency><groupId>org.projectlombok</groupId><artifactId>lombok</artifactId><optional>true</optional></dependency>

  <!-- Testes -->
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-test</artifactId><scope>test</scope></dependency>
  <dependency><groupId>org.springframework.graphql</groupId><artifactId>spring-graphql-test</artifactId><scope>test</scope></dependency>
  <dependency><groupId>org.springframework.security</groupId><artifactId>spring-security-test</artifactId><scope>test</scope></dependency>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-testcontainers</artifactId><scope>test</scope></dependency>
  <dependency><groupId>org.testcontainers</groupId><artifactId>junit-jupiter</artifactId><scope>test</scope></dependency>
  <dependency><groupId>org.testcontainers</groupId><artifactId>mysql</artifactId><scope>test</scope></dependency>
  <dependency><groupId>org.testcontainers</groupId><artifactId>rabbitmq</artifactId><scope>test</scope></dependency>
  <dependency><groupId>org.testcontainers</groupId><artifactId>kafka</artifactId><scope>test</scope></dependency>
</dependencies>
```

## 1.2 `application.yml`

```yaml
server:
  port: 8081
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/scheduling_db
    username: root
    password: root
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
  graphql:
    graphiql:
      enabled: true      # UI em /graphiql (desligue em prod)
security:
  jwt:
    public-key: classpath:public.pem   # chave pública para validar o token
app:
  rabbit:
    exchange: appointment.reminders
    queue: reminder.queue
    routing-key: appointment.reminder
  kafka:
    topic: appointment-events
```

## 1.3 Migration Flyway

`src/main/resources/db/migration/V1__create_appointments.sql`

```sql
CREATE TABLE appointments (
    id           VARCHAR(36)  NOT NULL PRIMARY KEY,
    patient_id   VARCHAR(36)  NOT NULL,
    doctor_id    VARCHAR(36)  NOT NULL,
    scheduled_at DATETIME     NOT NULL,
    status       VARCHAR(20)  NOT NULL,
    created_at   DATETIME     NOT NULL
);
CREATE INDEX idx_appointments_patient ON appointments (patient_id);
CREATE INDEX idx_appointments_doctor  ON appointments (doctor_id);
```

## 1.4 Camada `domain` (Java puro, sem Spring)

`domain/AppointmentStatus.java`
```java
package com.biadevcosta.scheduling.domain;

public enum AppointmentStatus { SCHEDULED, COMPLETED, CANCELLED }
```

`domain/exception/AppointmentException.java`
```java
package com.biadevcosta.scheduling.domain.exception;

public class AppointmentException extends RuntimeException {
    public AppointmentException(String message) { super(message); }
}

// arquivos separados na prática:
class AppointmentNotFoundException extends AppointmentException {
    public AppointmentNotFoundException(String id) { super("Appointment not found: " + id); }
}
class NotAppointmentOwnerException extends AppointmentException {
    public NotAppointmentOwnerException() { super("Only the owner doctor can edit this appointment"); }
}
class InvalidAppointmentException extends AppointmentException {
    public InvalidAppointmentException(String reason) { super(reason); }
}
```

`domain/Appointment.java` — a entidade e as regras de negócio
```java
package com.biadevcosta.scheduling.domain;

import com.biadevcosta.scheduling.domain.exception.InvalidAppointmentException;
import com.biadevcosta.scheduling.domain.exception.NotAppointmentOwnerException;

import java.time.LocalDateTime;
import java.util.UUID;

public class Appointment {

    private final String id;
    private final String patientId;
    private String doctorId;
    private LocalDateTime scheduledAt;
    private AppointmentStatus status;
    private final LocalDateTime createdAt;

    private Appointment(String id, String patientId, String doctorId,
                        LocalDateTime scheduledAt, AppointmentStatus status, LocalDateTime createdAt) {
        this.id = id;
        this.patientId = patientId;
        this.doctorId = doctorId;
        this.scheduledAt = scheduledAt;
        this.status = status;
        this.createdAt = createdAt;
    }

    /** Factory de criação: aplica invariantes. */
    public static Appointment schedule(String patientId, String doctorId, LocalDateTime scheduledAt) {
        if (patientId == null || doctorId == null)
            throw new InvalidAppointmentException("patient and doctor are required");
        if (scheduledAt == null || scheduledAt.isBefore(LocalDateTime.now()))
            throw new InvalidAppointmentException("scheduledAt must be in the future");
        return new Appointment(UUID.randomUUID().toString(), patientId, doctorId,
                scheduledAt, AppointmentStatus.SCHEDULED, LocalDateTime.now());
    }

    /** Reconstrução a partir do banco (sem validar invariantes de criação). */
    public static Appointment rehydrate(String id, String patientId, String doctorId,
                                        LocalDateTime scheduledAt, AppointmentStatus status, LocalDateTime createdAt) {
        return new Appointment(id, patientId, doctorId, scheduledAt, status, createdAt);
    }

    /** Regra "só o dono edita": o médico dono (doctorId) é quem pode alterar. */
    public void edit(String callerDoctorId, LocalDateTime newScheduledAt, AppointmentStatus newStatus) {
        if (!this.doctorId.equals(callerDoctorId))
            throw new NotAppointmentOwnerException();
        if (newScheduledAt != null) {
            if (newScheduledAt.isBefore(LocalDateTime.now()))
                throw new InvalidAppointmentException("scheduledAt must be in the future");
            this.scheduledAt = newScheduledAt;
        }
        if (newStatus != null) this.status = newStatus;
    }

    public String id() { return id; }
    public String patientId() { return patientId; }
    public String doctorId() { return doctorId; }
    public LocalDateTime scheduledAt() { return scheduledAt; }
    public AppointmentStatus status() { return status; }
    public LocalDateTime createdAt() { return createdAt; }
}
```

## 1.5 Camada `application` (POJOs + portas)

`application/port/AppointmentRepository.java`
```java
package com.biadevcosta.scheduling.application.port;

import com.biadevcosta.scheduling.domain.Appointment;
import java.util.Optional;

public interface AppointmentRepository {
    Appointment save(Appointment appointment);
    Optional<Appointment> findById(String id);
}
```

`application/port/ReminderPublisher.java` e `AppointmentEventPublisher.java`
```java
package com.biadevcosta.scheduling.application.port;
import com.biadevcosta.scheduling.domain.Appointment;

public interface ReminderPublisher {          // implementado com RabbitMQ
    void publishReminder(Appointment appointment);
}

public interface AppointmentEventPublisher {  // implementado com Kafka
    void publishCreated(Appointment appointment);
    void publishUpdated(Appointment appointment);
}
```

`application/command/*` — entradas dos casos de uso (carregam a identidade do chamador)
```java
package com.biadevcosta.scheduling.application.command;
import java.time.LocalDateTime;

public record ScheduleAppointmentCommand(
        String patientId, String doctorId, LocalDateTime scheduledAt) {}

public record EditAppointmentCommand(
        String appointmentId, LocalDateTime newScheduledAt, String newStatus,
        String callerId /* userId do token */) {}
```

`application/usecase/ScheduleAppointmentUseCase.java`
```java
package com.biadevcosta.scheduling.application.usecase;

import com.biadevcosta.scheduling.application.command.ScheduleAppointmentCommand;
import com.biadevcosta.scheduling.application.port.*;
import com.biadevcosta.scheduling.domain.Appointment;

public class ScheduleAppointmentUseCase {

    private final AppointmentRepository repository;
    private final ReminderPublisher reminderPublisher;
    private final AppointmentEventPublisher eventPublisher;

    public ScheduleAppointmentUseCase(AppointmentRepository repository,
                                      ReminderPublisher reminderPublisher,
                                      AppointmentEventPublisher eventPublisher) {
        this.repository = repository;
        this.reminderPublisher = reminderPublisher;
        this.eventPublisher = eventPublisher;
    }

    public Appointment execute(ScheduleAppointmentCommand cmd) {
        Appointment appointment = Appointment.schedule(cmd.patientId(), cmd.doctorId(), cmd.scheduledAt());
        Appointment saved = repository.save(appointment);   // 1) persiste (fonte da verdade)
        reminderPublisher.publishReminder(saved);           // 2) Rabbit (lembrete)
        eventPublisher.publishCreated(saved);               // 3) Kafka (histórico)
        return saved;
    }
}
```

`application/usecase/EditAppointmentUseCase.java`
```java
package com.biadevcosta.scheduling.application.usecase;

import com.biadevcosta.scheduling.application.command.EditAppointmentCommand;
import com.biadevcosta.scheduling.application.port.*;
import com.biadevcosta.scheduling.domain.*;
import com.biadevcosta.scheduling.domain.exception.AppointmentNotFoundException;

public class EditAppointmentUseCase {

    private final AppointmentRepository repository;
    private final AppointmentEventPublisher eventPublisher;

    public EditAppointmentUseCase(AppointmentRepository repository, AppointmentEventPublisher eventPublisher) {
        this.repository = repository;
        this.eventPublisher = eventPublisher;
    }

    public Appointment execute(EditAppointmentCommand cmd) {
        Appointment appointment = repository.findById(cmd.appointmentId())
                .orElseThrow(() -> new AppointmentNotFoundException(cmd.appointmentId()));
        AppointmentStatus status = cmd.newStatus() == null ? null : AppointmentStatus.valueOf(cmd.newStatus());
        appointment.edit(cmd.callerId(), cmd.newScheduledAt(), status);  // regra do dono no domínio
        Appointment saved = repository.save(appointment);
        eventPublisher.publishUpdated(saved);
        return saved;
    }
}
```

## 1.6 Camada `infrastructure`

### Persistência (Spring Data JDBC)

`infrastructure/persistence/AppointmentEntity.java`
```java
package com.biadevcosta.scheduling.infrastructure.persistence;

import org.springframework.data.annotation.Id;
import org.springframework.data.relational.core.mapping.Table;
import java.time.LocalDateTime;

@Table("appointments")
public class AppointmentEntity {
    @Id private String id;
    private String patientId;
    private String doctorId;
    private LocalDateTime scheduledAt;
    private String status;
    private LocalDateTime createdAt;
    // getters/setters (ou use @Data do Lombok)
}
```

`infrastructure/persistence/AppointmentJdbcRepository.java`
```java
package com.biadevcosta.scheduling.infrastructure.persistence;

import org.springframework.data.repository.CrudRepository;

public interface AppointmentJdbcRepository extends CrudRepository<AppointmentEntity, String> { }
```

`infrastructure/persistence/AppointmentMapper.java`
```java
package com.biadevcosta.scheduling.infrastructure.persistence;

import com.biadevcosta.scheduling.domain.*;

public class AppointmentMapper {
    public static AppointmentEntity toEntity(Appointment a) {
        AppointmentEntity e = new AppointmentEntity();
        e.setId(a.id()); e.setPatientId(a.patientId()); e.setDoctorId(a.doctorId());
        e.setScheduledAt(a.scheduledAt()); e.setStatus(a.status().name()); e.setCreatedAt(a.createdAt());
        return e;
    }
    public static Appointment toDomain(AppointmentEntity e) {
        return Appointment.rehydrate(e.getId(), e.getPatientId(), e.getDoctorId(),
                e.getScheduledAt(), AppointmentStatus.valueOf(e.getStatus()), e.getCreatedAt());
    }
}
```

`infrastructure/persistence/AppointmentRepositoryImpl.java`
```java
package com.biadevcosta.scheduling.infrastructure.persistence;

import com.biadevcosta.scheduling.application.port.AppointmentRepository;
import com.biadevcosta.scheduling.domain.Appointment;
import org.springframework.stereotype.Repository;
import java.util.Optional;

@Repository
public class AppointmentRepositoryImpl implements AppointmentRepository {
    private final AppointmentJdbcRepository jdbc;
    public AppointmentRepositoryImpl(AppointmentJdbcRepository jdbc) { this.jdbc = jdbc; }

    @Override public Appointment save(Appointment a) {
        return AppointmentMapper.toDomain(jdbc.save(AppointmentMapper.toEntity(a)));
    }
    @Override public Optional<Appointment> findById(String id) {
        return jdbc.findById(id).map(AppointmentMapper::toDomain);
    }
}
```

### Wiring dos casos de uso (`@Bean`)

`infrastructure/config/UseCaseConfig.java`
```java
package com.biadevcosta.scheduling.infrastructure.config;

import com.biadevcosta.scheduling.application.port.*;
import com.biadevcosta.scheduling.application.usecase.*;
import org.springframework.context.annotation.*;

@Configuration
public class UseCaseConfig {
    @Bean
    ScheduleAppointmentUseCase scheduleAppointmentUseCase(
            AppointmentRepository repo, ReminderPublisher reminder, AppointmentEventPublisher events) {
        return new ScheduleAppointmentUseCase(repo, reminder, events);
    }
    @Bean
    EditAppointmentUseCase editAppointmentUseCase(AppointmentRepository repo, AppointmentEventPublisher events) {
        return new EditAppointmentUseCase(repo, events);
    }
}
```

### Mensageria — RabbitMQ

`infrastructure/messaging/rabbit/RabbitConfig.java`
```java
package com.biadevcosta.scheduling.infrastructure.messaging.rabbit;

import org.springframework.amqp.core.*;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.*;

@Configuration
public class RabbitConfig {
    @Value("${app.rabbit.exchange}") String exchange;
    @Value("${app.rabbit.queue}") String queue;
    @Value("${app.rabbit.routing-key}") String routingKey;

    @Bean DirectExchange reminderExchange() { return new DirectExchange(exchange); }
    @Bean Queue reminderQueue() {
        // fila principal com dead-letter apontando para a DLQ (o consumidor configura a DLQ)
        return QueueBuilder.durable(queue)
                .withArgument("x-dead-letter-exchange", "")
                .withArgument("x-dead-letter-routing-key", "reminder.dlq")
                .build();
    }
    @Bean Binding reminderBinding() {
        return BindingBuilder.bind(reminderQueue()).to(reminderExchange()).with(routingKey);
    }
    @Bean Jackson2JsonMessageConverter jsonConverter() { return new Jackson2JsonMessageConverter(); }
    @Bean RabbitTemplate rabbitTemplate(ConnectionFactory cf, Jackson2JsonMessageConverter conv) {
        RabbitTemplate t = new RabbitTemplate(cf); t.setMessageConverter(conv); return t;
    }
}
```

`infrastructure/messaging/AppointmentReminder.java` (payload do Rabbit — só IDs; o Notification resolve o contato)
```java
package com.biadevcosta.scheduling.infrastructure.messaging;
import java.time.LocalDateTime;
public record AppointmentReminder(String appointmentId, String patientId, LocalDateTime scheduledAt) {}
```

`infrastructure/messaging/rabbit/RabbitReminderPublisher.java`
```java
package com.biadevcosta.scheduling.infrastructure.messaging.rabbit;

import com.biadevcosta.scheduling.application.port.ReminderPublisher;
import com.biadevcosta.scheduling.domain.Appointment;
import com.biadevcosta.scheduling.infrastructure.messaging.AppointmentReminder;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class RabbitReminderPublisher implements ReminderPublisher {
    private final RabbitTemplate template;
    @Value("${app.rabbit.exchange}") String exchange;
    @Value("${app.rabbit.routing-key}") String routingKey;
    public RabbitReminderPublisher(RabbitTemplate template) { this.template = template; }

    @Override public void publishReminder(Appointment a) {
        template.convertAndSend(exchange, routingKey,
            new AppointmentReminder(a.id(), a.patientId(), a.scheduledAt()));
    }
}
```

### Mensageria — Kafka

`infrastructure/messaging/AppointmentEvent.java`
```java
package com.biadevcosta.scheduling.infrastructure.messaging;
import java.time.LocalDateTime;
public record AppointmentEvent(
    String type,           // "AppointmentCreated" | "AppointmentUpdated"
    String eventId,        // id único do evento (idempotência no consumidor)
    String appointmentId, String patientId, String doctorId,
    LocalDateTime scheduledAt, String status) {}
```

`infrastructure/messaging/kafka/KafkaConfig.java`
```java
package com.biadevcosta.scheduling.infrastructure.messaging.kafka;

import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.*;
import org.springframework.kafka.config.TopicBuilder;

@Configuration
public class KafkaConfig {
    @Bean NewTopic appointmentEvents() {
        return TopicBuilder.name("appointment-events").partitions(3).replicas(1).build();
    }
}
```

`infrastructure/messaging/kafka/KafkaAppointmentEventPublisher.java`
```java
package com.biadevcosta.scheduling.infrastructure.messaging.kafka;

import com.biadevcosta.scheduling.application.port.AppointmentEventPublisher;
import com.biadevcosta.scheduling.domain.Appointment;
import com.biadevcosta.scheduling.infrastructure.messaging.AppointmentEvent;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;
import java.util.UUID;

@Component
public class KafkaAppointmentEventPublisher implements AppointmentEventPublisher {
    private final KafkaTemplate<String, Object> template;
    @Value("${app.kafka.topic}") String topic;
    public KafkaAppointmentEventPublisher(KafkaTemplate<String, Object> template) { this.template = template; }

    @Override public void publishCreated(Appointment a) { send("AppointmentCreated", a); }
    @Override public void publishUpdated(Appointment a) { send("AppointmentUpdated", a); }

    private void send(String type, Appointment a) {
        AppointmentEvent event = new AppointmentEvent(type, UUID.randomUUID().toString(),
            a.id(), a.patientId(), a.doctorId(), a.scheduledAt(), a.status().name());
        // chave = patientId garante ordem por paciente
        template.send(topic, a.patientId(), event);
    }
}
```

### Segurança — validação do JWT (RS256, resource server)

`infrastructure/security/SecurityConfig.java`
```java
package com.biadevcosta.scheduling.infrastructure.security;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.*;
import org.springframework.core.io.Resource;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.oauth2.jwt.*;
import org.springframework.security.oauth2.server.resource.authentication.*;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.core.convert.converter.Converter;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;

import java.security.KeyFactory;
import java.security.interfaces.RSAPublicKey;
import java.security.spec.X509EncodedKeySpec;
import java.util.*;

@Configuration
@EnableMethodSecurity   // habilita @PreAuthorize
public class SecurityConfig {

    @Value("${security.jwt.public-key}") Resource publicKey;

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http, JwtDecoder decoder) throws Exception {
        http
          .csrf(c -> c.disable())
          .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
          .authorizeHttpRequests(a -> a
              .requestMatchers("/actuator/**", "/graphiql/**").permitAll()
              .anyRequest().authenticated())
          .oauth2ResourceServer(o -> o.jwt(j -> j.decoder(decoder)
              .jwtAuthenticationConverter(rolesConverter())));
        return http.build();
    }

    @Bean
    JwtDecoder jwtDecoder() throws Exception {
        byte[] bytes = publicKey.getContentAsByteArray();
        String pem = new String(bytes)
            .replaceAll("-----\\w+ PUBLIC KEY-----", "").replaceAll("\\s", "");
        var spec = new X509EncodedKeySpec(Base64.getDecoder().decode(pem));
        RSAPublicKey key = (RSAPublicKey) KeyFactory.getInstance("RSA").generatePublic(spec);
        return NimbusJwtDecoder.withPublicKey(key).build();
    }

    /** Converte o claim "role" (ex: "DOCTOR") em ROLE_DOCTOR para o hasRole funcionar. */
    private JwtAuthenticationConverter rolesConverter() {
        Converter<Jwt, Collection<GrantedAuthority>> authorities = jwt -> {
            String role = jwt.getClaimAsString("role");
            return role == null ? List.of()
                : List.of(new SimpleGrantedAuthority("ROLE_" + role));
        };
        JwtAuthenticationConverter conv = new JwtAuthenticationConverter();
        conv.setJwtGrantedAuthoritiesConverter(authorities);
        return conv;
    }
}
```

### GraphQL — schema + resolver

`src/main/resources/graphql/schema.graphqls`
```graphql
type Mutation {
    scheduleAppointment(input: ScheduleInput!): Appointment!
    editAppointment(input: EditInput!): Appointment!
}

input ScheduleInput {
    patientId: ID!
    doctorId: ID!
    scheduledAt: String!     # ISO-8601, ex "2026-09-01T14:30:00"
}
input EditInput {
    appointmentId: ID!
    scheduledAt: String
    status: String
}
type Appointment {
    id: ID!
    patientId: ID!
    doctorId: ID!
    scheduledAt: String!
    status: String!
}
type Query { _ping: String }   # GraphQL exige ao menos uma Query
```

`infrastructure/web/graphql/AppointmentMutationController.java`
```java
package com.biadevcosta.scheduling.infrastructure.web.graphql;

import com.biadevcosta.scheduling.application.command.*;
import com.biadevcosta.scheduling.application.usecase.*;
import com.biadevcosta.scheduling.domain.Appointment;
import org.springframework.graphql.data.method.annotation.*;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.stereotype.Controller;
import java.time.LocalDateTime;

@Controller
public class AppointmentMutationController {

    private final ScheduleAppointmentUseCase scheduleUseCase;
    private final EditAppointmentUseCase editUseCase;

    public AppointmentMutationController(ScheduleAppointmentUseCase s, EditAppointmentUseCase e) {
        this.scheduleUseCase = s; this.editUseCase = e;
    }

    @MutationMapping
    @PreAuthorize("hasAnyRole('DOCTOR','NURSE')")
    public AppointmentView scheduleAppointment(@Argument ScheduleInput input) {
        Appointment a = scheduleUseCase.execute(new ScheduleAppointmentCommand(
                input.patientId(), input.doctorId(), LocalDateTime.parse(input.scheduledAt())));
        return AppointmentView.from(a);
    }

    @MutationMapping
    @PreAuthorize("hasRole('DOCTOR')")
    public AppointmentView editAppointment(@Argument EditInput input, @AuthenticationPrincipal Jwt jwt) {
        String callerId = jwt.getSubject();   // userId do médico logado
        Appointment a = editUseCase.execute(new EditAppointmentCommand(
                input.appointmentId(),
                input.scheduledAt() == null ? null : LocalDateTime.parse(input.scheduledAt()),
                input.status(), callerId));
        return AppointmentView.from(a);
    }

    @QueryMapping public String _ping() { return "ok"; }

    // records de entrada/saída do GraphQL
    public record ScheduleInput(String patientId, String doctorId, String scheduledAt) {}
    public record EditInput(String appointmentId, String scheduledAt, String status) {}
    public record AppointmentView(String id, String patientId, String doctorId, String scheduledAt, String status) {
        static AppointmentView from(Appointment a) {
            return new AppointmentView(a.id(), a.patientId(), a.doctorId(),
                    a.scheduledAt().toString(), a.status().name());
        }
    }
}
```

## 1.7 Como rodar e testar manualmente

```bash
./mvnw spring-boot:run          # sobe na 8081
```

Abra `http://localhost:8081/graphiql`. Como a rota exige JWT, gere um token de teste (o identity fará isso de verdade; para testar isolado, veja o helper de teste em 1.8). No GraphiQL, adicione o header:
```json
{ "Authorization": "Bearer SEU_TOKEN" }
```
E rode:
```graphql
mutation {
  scheduleAppointment(input: {
    patientId: "pac-1", doctorId: "doc-1", scheduledAt: "2026-12-01T10:00:00"
  }) { id status }
}
```
Confira a mensagem na fila (UI do Rabbit) e o registro na tabela `appointments`.

## 1.8 Testes a fundo (mirando 80%)

### Unitário do caso de uso — `ScheduleAppointmentUseCaseTest`
```java
package com.biadevcosta.scheduling.application.usecase;

import com.biadevcosta.scheduling.application.command.ScheduleAppointmentCommand;
import com.biadevcosta.scheduling.application.port.*;
import com.biadevcosta.scheduling.domain.Appointment;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.LocalDateTime;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class ScheduleAppointmentUseCaseTest {

    @Mock AppointmentRepository repository;
    @Mock ReminderPublisher reminderPublisher;
    @Mock AppointmentEventPublisher eventPublisher;
    @InjectMocks ScheduleAppointmentUseCase useCase;

    @Test
    void schedules_persists_and_publishes() {
        when(repository.save(any())).thenAnswer(i -> i.getArgument(0));
        var cmd = new ScheduleAppointmentCommand("pac-1", "doc-1", LocalDateTime.now().plusDays(1));

        Appointment result = useCase.execute(cmd);

        assertThat(result.patientId()).isEqualTo("pac-1");
        assertThat(result.status().name()).isEqualTo("SCHEDULED");
        InOrder inOrder = inOrder(repository, reminderPublisher, eventPublisher);
        inOrder.verify(repository).save(any());        // persiste antes
        inOrder.verify(reminderPublisher).publishReminder(any());
        inOrder.verify(eventPublisher).publishCreated(any());
    }

    @Test
    void rejects_past_date() {
        var cmd = new ScheduleAppointmentCommand("pac-1", "doc-1", LocalDateTime.now().minusDays(1));
        assertThatThrownBy(() -> useCase.execute(cmd))
            .hasMessageContaining("future");
        verifyNoInteractions(repository);
    }
}
```

### Unitário de domínio — `AppointmentTest`
```java
package com.biadevcosta.scheduling.domain;

import com.biadevcosta.scheduling.domain.exception.*;
import org.junit.jupiter.api.Test;
import java.time.LocalDateTime;
import static org.assertj.core.api.Assertions.*;

class AppointmentTest {
    @Test void only_owner_can_edit() {
        Appointment a = Appointment.schedule("pac-1", "doc-1", LocalDateTime.now().plusDays(1));
        assertThatThrownBy(() -> a.edit("doc-999", LocalDateTime.now().plusDays(2), null))
            .isInstanceOf(NotAppointmentOwnerException.class);
    }
    @Test void owner_edits_status() {
        Appointment a = Appointment.schedule("pac-1", "doc-1", LocalDateTime.now().plusDays(1));
        a.edit("doc-1", null, AppointmentStatus.CANCELLED);
        assertThat(a.status()).isEqualTo(AppointmentStatus.CANCELLED);
    }
}
```

### Integração com Testcontainers (MySQL + Rabbit + Kafka) + GraphQL

`src/test/.../IntegrationTestConfig.java` — sobe os containers e injeta as propriedades:
```java
package com.biadevcosta.scheduling;

import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.context.annotation.Bean;
import org.testcontainers.containers.*;
import org.testcontainers.utility.DockerImageName;

@TestConfiguration(proxyBeanMethods = false)
public class IntegrationTestConfig {
    @Bean @ServiceConnection
    MySQLContainer<?> mysql() { return new MySQLContainer<>("mysql:8"); }
    @Bean @ServiceConnection
    RabbitMQContainer rabbit() { return new RabbitMQContainer("rabbitmq:3-management"); }
    @Bean @ServiceConnection
    KafkaContainer kafka() { return new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.1")); }
}
```

`SchedulingIntegrationTest` — testa a mutation ponta a ponta com um JWT de teste:
```java
package com.biadevcosta.scheduling;

import com.nimbusds.jose.*;
import com.nimbusds.jose.crypto.RSASSASigner;
import com.nimbusds.jwt.*;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;
import org.springframework.graphql.test.tester.HttpGraphQlTester;
import org.springframework.test.web.reactive.server.WebTestClient;
import org.springframework.web.context.WebApplicationContext;

import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.time.Instant;
import java.util.Date;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Import(IntegrationTestConfig.class)
class SchedulingIntegrationTest {

    @Autowired WebApplicationContext context;

    // Para o teste rodar isolado, geramos um par de chaves em memória e apontamos
    // o decoder para a pública (via @DynamicPropertySource ou um perfil de teste).
    // Aqui, assinamos um JWT com a MESMA chave que o serviço valida no perfil de teste.

    @Test
    void schedules_appointment_with_valid_token() throws Exception {
        String token = testJwt("doc-1", "DOCTOR");
        HttpGraphQlTester tester = HttpGraphQlTester.builder(
                WebTestClient.bindToApplicationContext(context)
                    .configureClient().baseUrl("/graphql")
                    .defaultHeader("Authorization", "Bearer " + token).build()).build();

        tester.document("""
            mutation {
              scheduleAppointment(input: {
                patientId: "pac-1", doctorId: "doc-1", scheduledAt: "2026-12-01T10:00:00"
              }) { id status }
            }""")
          .execute()
          .path("scheduleAppointment.status").entity(String.class).isEqualTo("SCHEDULED");
    }

    static KeyPair KEYS;
    static String testJwt(String subject, String role) throws Exception {
        if (KEYS == null) { var g = KeyPairGenerator.getInstance("RSA"); g.initialize(2048); KEYS = g.generateKeyPair(); }
        var claims = new JWTClaimsSet.Builder()
            .subject(subject).claim("role", role)
            .issueTime(Date.from(Instant.now()))
            .expirationTime(Date.from(Instant.now().plusSeconds(600))).build();
        var jwt = new SignedJWT(new JWSHeader(JWSAlgorithm.RS256), claims);
        jwt.sign(new RSASSASigner(KEYS.getPrivate()));
        return jwt.serialize();
    }
}
```
> Dica: exponha `KEYS.getPublic()` como o `JwtDecoder` via um `@TestConfiguration`/perfil `test`, para o serviço validar exatamente o token que o teste assina. Assim você testa o caminho de segurança real, sem depender do identity.

### JaCoCo com mínimo de 80% (no `pom.xml`)
```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.12</version>
  <executions>
    <execution><goals><goal>prepare-agent</goal></goals></execution>
    <execution>
      <id>report</id><phase>test</phase><goals><goal>report</goal></goals>
    </execution>
    <execution>
      <id>check</id><goals><goal>check</goal></goals>
      <configuration>
        <rules>
          <rule>
            <element>BUNDLE</element>
            <limits>
              <limit><counter>LINE</counter><value>COVEREDRATIO</value><minimum>0.80</minimum></limit>
            </limits>
            <excludes>
              <exclude>**/*Application.*</exclude>
              <exclude>**/config/**</exclude>
            </excludes>
          </rule>
        </rules>
      </configuration>
    </execution>
  </executions>
</plugin>
```
Rode e veja o relatório:
```bash
./mvnw clean verify
# relatório HTML em target/site/jacoco/index.html
```

### Allure (relatório de testes)
```xml
<plugin>
  <groupId>io.qameta.allure</groupId>
  <artifactId>allure-maven</artifactId>
  <version>2.12.0</version>
</plugin>
```
```bash
./mvnw test
./mvnw allure:serve   # abre o relatório no navegador
```

---

# 2. identity-service (auth + usuários)

Só as partes distintas — camadas/`@Bean`/mapper seguem o molde do scheduling.

## 2.1 Dependências e `application.yml`
Deps: `web`, `security`, `data-jdbc`, `validation`, `actuator`, `mysql`, `flyway`, `lombok`, e para assinar o JWT: **jjwt** (ou Nimbus). Testes: `starter-test`, `security-test`, testcontainers `mysql`.

```xml
<dependency><groupId>io.jsonwebtoken</groupId><artifactId>jjwt-api</artifactId><version>0.12.6</version></dependency>
<dependency><groupId>io.jsonwebtoken</groupId><artifactId>jjwt-impl</artifactId><version>0.12.6</version><scope>runtime</scope></dependency>
<dependency><groupId>io.jsonwebtoken</groupId><artifactId>jjwt-jackson</artifactId><version>0.12.6</version><scope>runtime</scope></dependency>
```
```yaml
server: { port: 8080 }
spring:
  datasource: { url: jdbc:mysql://localhost:3306/identity_db, username: root, password: root }
security:
  jwt:
    private-key: classpath:private.pem
    issuer: hospital-identity
    ttl-seconds: 3600
```

## 2.2 Migration
`V1__create_users.sql`
```sql
CREATE TABLE users (
    id VARCHAR(36) PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(100) NOT NULL,
    role VARCHAR(20) NOT NULL,          -- DOCTOR | NURSE | PATIENT
    full_name VARCHAR(255) NOT NULL,
    phone VARCHAR(30),
    crm VARCHAR(20),
    specialty VARCHAR(100)
);
```

## 2.3 Emissão do JWT (RS256, chave privada) — porta + adapter
`application/port/TokenIssuer.java`
```java
public interface TokenIssuer {
    String issue(String userId, String role, String patientId /* null se não paciente */);
}
```
`infrastructure/security/JwtTokenIssuer.java`
```java
package com.biadevcosta.identity.infrastructure.security;

import com.biadevcosta.identity.application.port.TokenIssuer;
import io.jsonwebtoken.Jwts;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Component;

import java.security.KeyFactory;
import java.security.PrivateKey;
import java.security.spec.PKCS8EncodedKeySpec;
import java.time.Instant;
import java.util.*;

@Component
public class JwtTokenIssuer implements TokenIssuer {
    @Value("${security.jwt.private-key}") Resource privateKeyRes;
    @Value("${security.jwt.issuer}") String issuer;
    @Value("${security.jwt.ttl-seconds}") long ttl;

    @Override public String issue(String userId, String role, String patientId) {
        Instant now = Instant.now();
        var builder = Jwts.builder()
            .issuer(issuer).subject(userId)
            .claim("role", role)
            .issuedAt(Date.from(now))
            .expiration(Date.from(now.plusSeconds(ttl)))
            .signWith(loadPrivateKey(), Jwts.SIG.RS256);
        if (patientId != null) builder.claim("patientId", patientId);
        return builder.compact();
    }

    private PrivateKey loadPrivateKey() {
        try {
            String pem = new String(privateKeyRes.getContentAsByteArray())
                .replaceAll("-----\\w+ PRIVATE KEY-----", "").replaceAll("\\s", "");
            var spec = new PKCS8EncodedKeySpec(Base64.getDecoder().decode(pem));
            return KeyFactory.getInstance("RSA").generatePrivate(spec);
        } catch (Exception e) { throw new IllegalStateException("cannot load private key", e); }
    }
}
```

## 2.4 Login e registro (casos de uso + controller REST)
**Clean Architecture:** o core define uma porta `PasswordHasher`; a infra a implementa com BCrypt. Assim o `application` não importa nada do Spring Security.

`application/port/PasswordHasher.java` — porta no core
```java
package com.biadevcosta.identity.application.port;

public interface PasswordHasher {
    boolean matches(String raw, String hash);
    String hash(String raw);
}
```

`infrastructure/security/BCryptPasswordHasher.java` — adapter na infra
```java
package com.biadevcosta.identity.infrastructure.security;

import com.biadevcosta.identity.application.port.PasswordHasher;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.stereotype.Component;

@Component
public class BCryptPasswordHasher implements PasswordHasher {
    private final BCryptPasswordEncoder encoder = new BCryptPasswordEncoder();
    @Override public boolean matches(String raw, String hash) { return encoder.matches(raw, hash); }
    @Override public String hash(String raw) { return encoder.encode(raw); }
}
```

`application/usecase/AuthenticateUseCase.java` — depende só de portas
```java
public class AuthenticateUseCase {
    private final UserRepository users;
    private final PasswordHasher passwordHasher;   // porta, não o PasswordEncoder do Spring
    private final TokenIssuer tokenIssuer;
    // construtor...

    public String login(String email, String rawPassword) {
        User u = users.findByEmail(email).orElseThrow(() -> new InvalidCredentialsException());
        if (!passwordHasher.matches(rawPassword, u.passwordHash())) throw new InvalidCredentialsException();
        String patientId = u.role() == Role.PATIENT ? u.id() : null;
        return tokenIssuer.issue(u.id(), u.role().name(), patientId);
    }
}
```
`infrastructure/web/AuthController.java`
```java
@RestController
@RequestMapping("/auth")
public class AuthController {
    private final AuthenticateUseCase auth;
    public AuthController(AuthenticateUseCase auth) { this.auth = auth; }

    @PostMapping("/login")
    public Map<String,String> login(@RequestBody LoginRequest req) {
        return Map.of("token", auth.login(req.email(), req.password()));
    }
    public record LoginRequest(String email, String password) {}
}
```
`SecurityConfig`: `passwordEncoder` = `BCryptPasswordEncoder`; `/auth/login` e `/users/**` (leitura interna) liberados conforme sua política; demais autenticados.

## 2.5 API que os outros consultam
`infrastructure/web/UserQueryController.java`
```java
@RestController
@RequestMapping("/users")
public class UserQueryController {
    private final UserRepository users;
    public UserQueryController(UserRepository users) { this.users = users; }

    @GetMapping("/{id}")
    public UserView getById(@PathVariable String id) {
        User u = users.findById(id).orElseThrow(() -> new UserNotFoundException(id));
        return new UserView(u.id(), u.fullName(), u.email(), u.role().name());
    }
    public record UserView(String id, String name, String email, String role) {}
}
```

## 2.6 Testes
- Unitário `AuthenticateUseCaseTest` (Mockito): senha certa → token; senha errada → `InvalidCredentialsException`.
- Unitário do `JwtTokenIssuer`: gera token e valida assinatura com a chave pública (Nimbus) + confere claims (`role`, `sub`, `exp`).
- Integração com Testcontainers MySQL: `POST /auth/login` retorna 200 e um token; `GET /users/{id}` retorna o usuário. JaCoCo igual ao molde.

---

# 3. notification-service

Consome o Rabbit, busca o contato no identity (com cache) e "envia". Sem GraphQL, sem JWT.

## 3.1 Deps e `application.yml`
Deps: `amqp`, `actuator`, `web` (para o WebClient/RestClient), `spring-boot-starter-cache` + `caffeine`, `lombok`. Testes: `starter-test`, testcontainers `rabbitmq`, `wiremock` (para simular o identity).
```yaml
server: { port: 8082 }
spring:
  rabbitmq: { host: localhost, port: 5672, username: guest, password: guest }
app:
  identity-base-url: http://localhost:8080
```

## 3.2 DLQ + listener
`infrastructure/messaging/RabbitConfig.java` — declara a DLQ:
```java
@Bean Queue dlq() { return QueueBuilder.durable("reminder.dlq").build(); }
```
Habilite retry no `application.yml` (o starter faz retry e, esgotado, manda pra DLQ configurada na fila principal):
```yaml
spring:
  rabbitmq:
    listener:
      simple:
        retry: { enabled: true, max-attempts: 3, initial-interval: 2000 }
        default-requeue-rejected: false   # rejeitado vai pra DLQ, não volta pra fila
```
`infrastructure/messaging/ReminderListener.java`
```java
@Component
public class ReminderListener {
    private final SendReminderUseCase useCase;
    public ReminderListener(SendReminderUseCase useCase) { this.useCase = useCase; }

    @RabbitListener(queues = "${app.rabbit.queue:reminder.queue}")
    public void onReminder(AppointmentReminder msg) {
        useCase.execute(msg.appointmentId(), msg.patientId(), msg.scheduledAt());
    }
}
```

## 3.3 Consulta ao identity (porta no core + adapter na infra)

**Clean Architecture:** o caso de uso depende de uma **porta** (`UserDirectory`), não de um cliente HTTP concreto. O detalhe (HTTP + cache) fica na infra.

`application/port/UserDirectory.java` — a porta, no core
```java
package com.biadevcosta.notification.application.port;

public interface UserDirectory {
    UserInfo findById(String id);
    record UserInfo(String id, String name, String email, String role) {}  // tipo do core
}
```

`infrastructure/client/IdentityUserDirectory.java` — o adapter, na infra
```java
package com.biadevcosta.notification.infrastructure.client;

import com.biadevcosta.notification.application.port.UserDirectory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;

@Component
public class IdentityUserDirectory implements UserDirectory {
    private final RestClient rest;
    public IdentityUserDirectory(@Value("${app.identity-base-url}") String baseUrl) {
        this.rest = RestClient.builder().baseUrl(baseUrl).build();
    }
    @Override
    @Cacheable("users")   // Caffeine: evita bater no identity a cada mensagem
    public UserInfo findById(String id) {
        var r = rest.get().uri("/users/{id}", id).retrieve().body(Response.class);
        return new UserInfo(r.id(), r.name(), r.email(), r.role());
    }
    private record Response(String id, String name, String email, String role) {}
}
```
Habilite o cache: `@EnableCaching` numa `@Configuration`, e Caffeine no `application.yml`:
```yaml
spring:
  cache:
    cache-names: users
    caffeine:
      spec: maximumSize=1000,expireAfterWrite=10m
```

## 3.4 Caso de uso + canal de envio
O caso de uso depende só de portas (`UserDirectory`, `ReminderChannel`) — não sabe que há HTTP ou cache do outro lado.
```java
public class SendReminderUseCase {
    private final UserDirectory users;       // porta (não o IdentityClient concreto)
    private final ReminderChannel channel;   // porta
    public SendReminderUseCase(UserDirectory users, ReminderChannel channel) {
        this.users = users; this.channel = channel;
    }
    public void execute(String appointmentId, String patientId, LocalDateTime when) {
        var user = users.findById(patientId);   // resolve nome/e-mail (adapter cuida do cache/fallback)
        channel.send(user.email(), "Lembrete de consulta em " + when + " — " + user.name());
    }
}
```
`LogReminderChannel implements ReminderChannel` apenas loga (canal real seria plugável). O `UseCaseConfig` monta o use case via `@Bean`, injetando `IdentityUserDirectory` e o canal.

## 3.5 Testes
- Unitário `SendReminderUseCaseTest` (Mockito): mocka as portas `UserDirectory` e `ReminderChannel`, verifica que envia com nome/e-mail resolvidos. (Testar o caso de uso mockando portas, sem subir Spring, é exatamente o benefício da Clean Architecture.)
- Integração com Testcontainers RabbitMQ + **WireMock** stubando `GET /users/{id}`: publica uma mensagem na fila e verifica que o canal foi chamado; teste separado força falha no envio e confirma que a mensagem cai na `reminder.dlq`.

---

# 4. history-service (leitura / CQRS)

Consome o Kafka com idempotência, monta o read model e serve GraphQL. Valida JWT (molde do scheduling) e resolve nomes no identity (cliente com cache igual ao notification).

## 4.1 Deps e `application.yml`
Deps: `web`, `graphql`, `security`, `oauth2-resource-server`, `data-jdbc`, `validation`, `actuator`, `spring-kafka`, `cache`+`caffeine`, `mysql`, `flyway`. Testes: `starter-test`, `graphql-test`, `security-test`, testcontainers `mysql`+`kafka`.
```yaml
server: { port: 8083 }
spring:
  datasource: { url: jdbc:mysql://localhost:3306/history_db, username: root, password: root }
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: history
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "*"
security: { jwt: { public-key: classpath:public.pem } }
app: { identity-base-url: http://localhost:8080 }
```

## 4.2 Migration
`V1__create_history.sql`
```sql
CREATE TABLE appointment_history (
    id VARCHAR(36) PRIMARY KEY,
    patient_id VARCHAR(36) NOT NULL,
    doctor_id VARCHAR(36) NOT NULL,
    scheduled_at DATETIME NOT NULL,
    status VARCHAR(20) NOT NULL
);
CREATE TABLE processed_events ( event_id VARCHAR(36) PRIMARY KEY );
CREATE INDEX idx_history_patient ON appointment_history (patient_id);
```

## 4.3 Consumo idempotente do Kafka
`infrastructure/messaging/AppointmentEventListener.java`
```java
@Component
public class AppointmentEventListener {
    private final RecordAppointmentUseCase useCase;
    public AppointmentEventListener(RecordAppointmentUseCase useCase) { this.useCase = useCase; }

    @KafkaListener(topics = "appointment-events", groupId = "history")
    public void onEvent(AppointmentEvent event) {
        useCase.apply(event);   // o caso de uso checa processed_events e ignora repetição
    }
}
```
`application/usecase/RecordAppointmentUseCase.java`
```java
public class RecordAppointmentUseCase {
    private final HistoryRepository history;
    private final ProcessedEventStore processed;   // porta de idempotência
    // construtor...

    public void apply(AppointmentEvent e) {
        if (processed.exists(e.eventId())) return;   // já processado → ignora
        history.upsert(new HistoryRecord(
            e.appointmentId(), e.patientId(), e.doctorId(), e.scheduledAt(), e.status()));
        processed.markProcessed(e.eventId());
    }
}
```

## 4.4 Queries GraphQL + regra de propriedade
`schema.graphqls`
```graphql
type Query {
    history(patientId: ID!): [HistoryItem!]!
    futureAppointments(patientId: ID!): [HistoryItem!]!
}
type HistoryItem { id: ID!, patientName: String!, doctorName: String!, scheduledAt: String!, status: String! }
```
`infrastructure/web/graphql/HistoryQueryController.java`
```java
@Controller
public class HistoryQueryController {
    private final QueryHistoryUseCase useCase;
    public HistoryQueryController(QueryHistoryUseCase useCase) { this.useCase = useCase; }

    @QueryMapping
    @PreAuthorize("hasAnyRole('DOCTOR','NURSE','PATIENT')")
    public List<HistoryItemView> history(@Argument String patientId, @AuthenticationPrincipal Jwt jwt) {
        return useCase.history(callerContext(jwt), patientId);
    }

    @QueryMapping
    @PreAuthorize("hasAnyRole('DOCTOR','NURSE','PATIENT')")
    public List<HistoryItemView> futureAppointments(@Argument String patientId, @AuthenticationPrincipal Jwt jwt) {
        return useCase.future(callerContext(jwt), patientId);
    }

    private CallerContext callerContext(Jwt jwt) {
        return new CallerContext(jwt.getClaimAsString("role"), jwt.getClaimAsString("patientId"));
    }
}
```
A **regra de propriedade** vive no caso de uso — nunca confie no `patientId` do argumento para paciente:
```java
public List<HistoryItemView> history(CallerContext caller, String requestedPatientId) {
    String effectivePatientId = "PATIENT".equals(caller.role())
        ? caller.patientId()          // paciente: ignora o argumento, usa o do token
        : requestedPatientId;         // profissional: pode consultar qualquer paciente
    return repository.findByPatient(effectivePatientId).stream()
        .map(this::enrichWithNames)   // resolve nomes via porta UserDirectory (adapter cuida do HTTP/cache)
        .toList();
}
```
> O `history-service` tem a **mesma porta `UserDirectory`** do notification (item 3.3): interface no `application`, adapter `IdentityUserDirectory` na infra. O `enrichWithNames` chama `users.findById(...)` pela porta — nunca o cliente HTTP direto.

## 4.5 Testes
- Unitário `RecordAppointmentUseCaseTest`: primeiro evento grava; **evento repetido é ignorado** (verifica idempotência via mock do `ProcessedEventStore`).
- Unitário `QueryHistoryUseCaseTest`: paciente recebe só as suas (o `patientId` do argumento é ignorado quando o papel é PATIENT); profissional recebe o solicitado.
- Integração com Testcontainers Kafka + MySQL: publica um `AppointmentEvent` no tópico, aguarda o consumo e valida via GraphQL (`HttpGraphQlTester` com JWT de teste, igual ao scheduling) que a query devolve o item. WireMock para o identity resolver nomes.

---

# 5. Fechamento comum aos quatro

- **JaCoCo 80%**: o mesmo bloco do item 1.8 em cada `pom.xml`. Exclua `*Application`, `config` e records de DTO puros (sem lógica) via `<excludes>` para não penalizar a métrica com código sem comportamento.
- **Onde a cobertura mora**: casos de uso e regras de domínio (unitários, rápidos) + os fluxos de entrada/saída (integração). Priorize testar comportamento (regras, idempotência, propriedade, ordem persistir→publicar), não getters.
- **Allure** em todos, com `./mvnw allure:serve`.
- **Collection Insomnia**: exporte requisições para login (identity), agendar/editar (scheduling, com o token no header) e as queries (history). Versione o arquivo no repo.
- **README por serviço**: como configurar (`.env`/`application.yml`), como rodar (`./mvnw spring-boot:run` ou Docker), os endpoints/queries e o desenho da arquitetura.

## Ordem de execução para testar o sistema todo
1. Suba MySQL, Rabbit, Kafka (item 0).
2. Suba `identity` (8080) → registre um médico e um paciente → faça login e copie o token.
3. Suba `scheduling` (8081) → agende uma consulta com o token.
4. Suba `notification` (8082) → veja o lembrete logado (com nome/e-mail buscados no identity).
5. Suba `history` (8083) → rode a query e veja a consulta aparecer.
