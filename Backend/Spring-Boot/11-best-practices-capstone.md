# 11. Best Practices, Performance & Capstone Project

[← Back](10-deployment.md) | [Index](README.md)

The final section: how to keep a Spring Boot codebase healthy, how to make it fast and observable, and a **capstone project** that combines everything you've learned.

---

## 11.1 Code organization

Keep a clear, layered package structure. Two common styles:

**Layer-based** (fine for small/medium apps):

```
com.example.app
├── controller/
├── service/
├── repository/
├── model/       (entities)
├── dto/
├── exception/
├── config/
└── security/
```

**Feature-based** (scales better for large apps — group by domain, not by layer):

```
com.example.app
├── task/
│   ├── TaskController.java
│   ├── TaskService.java
│   ├── TaskRepository.java
│   └── Task.java
├── user/
│   └── ...
└── shared/
    ├── exception/
    └── config/
```

**Principles:**

- One responsibility per class; keep methods short.
- Controllers stay thin — no business logic. Services hold the rules. Repositories only do data access.
- Use DTOs at the boundary; never expose JPA entities directly in the API.
- Prefer constructor injection and immutability (`final` fields, records for DTOs).
- Program to interfaces where it aids testing and flexibility.

---

## 11.2 Logging

Spring Boot uses **SLF4J** + Logback by default. Never use `System.out.println` in real code.

```java
import org.slf4j.*;

@Service
public class TaskService {
    private static final Logger log = LoggerFactory.getLogger(TaskService.class);

    public TaskResponse create(CreateTaskRequest request) {
        log.info("Creating task with title '{}'", request.title());   // parameterized — efficient
        // ...
        log.debug("Task persisted with id {}", saved.getId());
        return response;
    }
}
```

**Guidelines:**

- Use levels correctly: `ERROR` (failures), `WARN` (recoverable oddities), `INFO` (key events), `DEBUG` (diagnostics).
- Use `{}` placeholders, not string concatenation — the arguments are only evaluated if the level is enabled.
- Never log secrets, passwords, tokens, or full PII.
- Configure per-package levels in `application.properties`:
  ```properties
  logging.level.root=INFO
  logging.level.com.example.app=DEBUG
  logging.file.name=logs/app.log
  ```
- For production, consider **structured (JSON) logging** so log aggregators (ELK, Loki, CloudWatch) can parse fields.

---

## 11.3 Monitoring with Actuator

Spring Boot **Actuator** exposes production-ready endpoints for health, metrics, and info.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```properties
# expose selected endpoints
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.endpoint.health.show-details=when-authorized
```

| Endpoint | Shows |
|----------|-------|
| `/actuator/health` | UP/DOWN + component checks (DB, disk) — used by load balancers and Kubernetes probes |
| `/actuator/info` | App metadata |
| `/actuator/metrics` | JVM, HTTP, DB pool metrics |
| `/actuator/prometheus` | Metrics in Prometheus format for scraping |

**Use case:** point Kubernetes liveness/readiness probes at `/actuator/health`, and scrape `/actuator/prometheus` into Grafana dashboards.

> Actuator endpoints can reveal sensitive internals. Expose only what you need and secure them (they sit behind your security config).

---

## 11.4 Performance optimization

- **Fix N+1 queries.** The most common JPA performance bug. Use `JOIN FETCH`, `@EntityGraph`, or batch fetching to load related data in one query instead of one-per-row.
- **Always paginate** large result sets (Section 5). Never `findAll()` an unbounded table.
- **Cache** hot, rarely-changing data with `@Cacheable` (Section 8), backed by Redis in production.
- **Tune the connection pool.** Spring Boot uses HikariCP. Right-size it:
  ```properties
  spring.datasource.hikari.maximum-pool-size=10
  spring.datasource.hikari.minimum-idle=2
  ```
- **Index your database** columns used in `WHERE`/`JOIN`/`ORDER BY`.
- **Select only what you need.** Use projections (interface- or DTO-based) instead of loading full entities for read-only views.
- **Do slow work asynchronously** (`@Async`) or offload to a queue.
- **Compress responses** and enable HTTP caching headers for static/read-heavy endpoints:
  ```properties
  server.compression.enabled=true
  ```
- **Measure before optimizing.** Use Actuator metrics and a profiler to find the real bottleneck rather than guessing.

---

## 11.5 Security best practices recap

- Hash passwords with BCrypt; never store or log plain text.
- Keep secrets out of source control — use environment variables or a secrets manager.
- Validate and sanitize all input (Section 6).
- Use HTTPS in production (typically terminated at a load balancer or gateway).
- Apply least privilege with roles and `@PreAuthorize`.
- Keep dependencies patched; scan them (e.g., OWASP Dependency-Check, `mvn versions:display-dependency-updates`).

---

## 11.6 API design best practices

- Use nouns for resources and HTTP methods for actions: `GET /api/tasks`, not `GET /api/getTasks`.
- Return correct status codes (200/201/204/400/401/403/404/409/500).
- Version your API when it changes incompatibly: `/api/v1/...`.
- Document it with **OpenAPI/Swagger** via `springdoc-openapi`:
  ```xml
  <dependency>
      <groupId>org.springdoc</groupId>
      <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
      <version>2.6.0</version>
  </dependency>
  ```
  Then browse interactive docs at `/swagger-ui.html`.

---

# 11.7 Capstone Project: Task Management API

Bring together every concept into one runnable, production-shaped application.

## What you'll build

A secure, multi-user **Task Management REST API** where users register, log in, and manage their own tasks — with validation, a real database, tests, and Docker deployment.

## Requirements

- **Auth:** register/login returning a JWT; all task endpoints require authentication (Section 7).
- **Users own their tasks:** a user only sees and edits their own tasks; admins see all.
- **Tasks:** title, description, priority (1–5), status (`TODO`/`IN_PROGRESS`/`DONE`), due date, timestamps.
- **CRUD + list** with pagination, sorting, and filtering by status (Sections 4–5).
- **Validation** on all input with a global error handler (Section 6).
- **Advanced:** a scheduled job that emails a daily digest of due tasks asynchronously (Section 8).
- **Tests:** unit (service), web slice (controller), data slice (repository), and one integration test (Section 9).
- **Deploy:** Dockerfile + docker-compose with PostgreSQL and Actuator health checks (Sections 10–11).

## Architecture

```
   Client (web / mobile / curl)
        │  Authorization: Bearer <JWT>
        ▼
 ┌──────────────────────────────────────────────┐
 │              Security Filter Chain              │  JWT validation + role checks
 └──────────────────────────────────────────────┘
        ▼
 ┌───────────────┐   ┌───────────────┐   ┌─────────────────┐
 │ AuthController │   │ TaskController │   │ AdminController  │
 └───────┬────────┘   └───────┬───────┘   └────────┬────────┘
         ▼                    ▼                     ▼
 ┌───────────────┐   ┌───────────────┐   ┌─────────────────┐
 │ AuthService    │   │ TaskService    │   │ (uses services) │
 └───────┬────────┘   └───────┬───────┘   └─────────────────┘
         ▼                    ▼
 ┌───────────────┐   ┌───────────────┐
 │ UserRepository │   │ TaskRepository │
 └───────┬────────┘   └───────┬───────┘
         └─────────┬──────────┘
                   ▼
            ┌─────────────┐        ┌──────────────────────────┐
            │ PostgreSQL   │        │ DigestScheduler (@Async  │
            │              │        │  + @Scheduled cron)      │
            └─────────────┘        └──────────────────────────┘

 Cross-cutting: @Valid validation · GlobalExceptionHandler · Actuator · Profiles
```

## Suggested package layout (feature-based)

```
com.example.taskmanager
├── TaskManagerApplication.java
├── auth/          AuthController, AuthService, dto/
├── user/          User, Role, UserRepository, AppUserDetailsService
├── task/          Task, TaskStatus, TaskController, TaskService, TaskRepository, dto/
├── security/      JwtService, JwtAuthFilter, SecurityConfig
├── notification/  DigestScheduler, EmailService, NotificationProperties
└── shared/        GlobalExceptionHandler, ApiError, exceptions, config/
```

## Data model

```java
@Entity @Table(name = "tasks")
public class Task {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(length = 1000)
    private String description;

    @Enumerated(EnumType.STRING)
    private TaskStatus status = TaskStatus.TODO;

    @Column(nullable = false)
    private int priority = 3;

    private LocalDate dueDate;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "owner_id")
    private User owner;

    @Column(updatable = false)
    private Instant createdAt = Instant.now();
    private Instant updatedAt;
    // getters/setters
}

public enum TaskStatus { TODO, IN_PROGRESS, DONE }
```

## Endpoints

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| POST | `/auth/register` | Public | Create account, get JWT |
| POST | `/auth/login` | Public | Log in, get JWT |
| GET | `/api/tasks` | USER | List own tasks (paged, sortable, `?status=`) |
| POST | `/api/tasks` | USER | Create a task |
| GET | `/api/tasks/{id}` | USER (owner) | Get one task |
| PUT | `/api/tasks/{id}` | USER (owner) | Update a task |
| PATCH | `/api/tasks/{id}/status` | USER (owner) | Change status |
| DELETE | `/api/tasks/{id}` | USER (owner) | Delete a task |
| GET | `/api/admin/tasks` | ADMIN | List all users' tasks |
| GET | `/actuator/health` | Public | Health check |

## Ownership check example

Enforce that users touch only their own tasks:

```java
@Transactional(readOnly = true)
public TaskResponse getOwnedTask(Long id, String username) {
    Task task = taskRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("Task not found: " + id));
    if (!task.getOwner().getUsername().equals(username)) {
        throw new AccessDeniedException("You don't own this task");
    }
    return toResponse(task);
}
```

The controller passes the authenticated user:

```java
@GetMapping("/{id}")
public TaskResponse get(@PathVariable Long id, java.security.Principal principal) {
    return taskService.getOwnedTask(id, principal.getName());
}
```

## Build order (step by step)

1. **Scaffold** with Spring Initializr: Web, JPA, Validation, Security, Actuator, PostgreSQL driver, DevTools.
2. **Model + persistence:** `User`, `Task`, `TaskStatus`, repositories. Start on H2, switch to PostgreSQL via a `prod` profile.
3. **Auth:** JWT security stack from Section 7; register/login.
4. **Task CRUD:** controller → service → repository with DTOs and ownership checks.
5. **List features:** pagination, sorting, `?status=` filtering.
6. **Validation + errors:** constraints on DTOs + `GlobalExceptionHandler`.
7. **Digest job:** `@Scheduled` + `@Async` email of tasks due today.
8. **Tests:** service (Mockito), controller (`@WebMvcTest`), repository (`@DataJpaTest`), one `@SpringBootTest`.
9. **Docs:** add springdoc; verify `/swagger-ui.html`.
10. **Deploy:** Dockerfile + docker-compose with PostgreSQL; confirm `/actuator/health` is UP.

## Acceptance checklist

- [ ] Register and log in; receive a JWT.
- [ ] Task endpoints reject requests without a valid token (401).
- [ ] A user cannot access another user's task (403/404).
- [ ] Admin can list all tasks; a normal user cannot (403).
- [ ] Invalid input returns 400 with per-field messages.
- [ ] `GET /api/tasks?status=TODO&page=0&size=5&sortBy=dueDate` works.
- [ ] The digest job logs/sends for tasks due today.
- [ ] All test slices pass with `./mvnw test`.
- [ ] `docker compose up` runs the app + PostgreSQL; `/actuator/health` returns UP.

## Stretch goals

- Add **tags** (`@ManyToMany` between Task and Tag).
- Add **refresh tokens** and token revocation.
- Add **rate limiting** (e.g., Bucket4j) on auth endpoints.
- Add **CI** (GitHub Actions) that runs tests and builds the Docker image.
- Add **OpenAPI** examples and deploy to a cloud (Cloud Run, App Service, or Beanstalk).

---

## Congratulations

You've gone from "what is Spring Boot" to designing, securing, testing, and deploying a complete REST API. The capstone mirrors what real backend teams build every day.

### Where to keep learning

- **Spring official guides:** [spring.io/guides](https://spring.io/guides)
- **Reference docs:** [Spring Boot](https://docs.spring.io/spring-boot/index.html), [Spring Security](https://docs.spring.io/spring-security/reference/), [Spring Data JPA](https://docs.spring.io/spring-data/jpa/reference/)
- **Next topics:** microservices with Spring Cloud, messaging with Kafka/RabbitMQ, reactive stacks with WebFlux, and observability with Micrometer + Grafana.

[← Back to index](README.md)
