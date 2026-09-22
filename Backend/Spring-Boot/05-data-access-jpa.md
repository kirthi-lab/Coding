# 5. Data Access with Spring Data JPA

[← Back](04-rest-api.md) | [Index](README.md) | Next → [6. Validation & Exception Handling](06-validation-exception-handling.md)

Time to replace the in-memory store with a real database. **Spring Data JPA** lets you persist objects with almost no boilerplate.

---

## 5.1 The concepts

- **JPA** (Jakarta Persistence API) is a specification for mapping Java objects to database tables (ORM — Object-Relational Mapping).
- **Hibernate** is the default JPA implementation Spring Boot uses.
- **Spring Data JPA** adds repositories: interfaces that generate queries for you.

```
  @Entity Task  ◄──maps to──►  table `tasks`
       ▲
       │  managed by
  Spring Data JPA repository (interface you write, impl Spring generates)
       │  uses
  Hibernate (JPA provider)
       │  talks to
  Database (H2 / MySQL / PostgreSQL) via JDBC
```

---

## 5.2 Add dependencies

Add to `pom.xml` (or select on Spring Initializr):

```xml
<!-- Spring Data JPA + Hibernate -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- H2 in-memory DB for development -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- MySQL driver (for production) -->
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- PostgreSQL driver (alternative) -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

---

## 5.3 Configure the database

### Development: H2 (zero setup)

H2 is an in-memory database — perfect for learning. Add to `application.properties`:

```properties
spring.datasource.url=jdbc:h2:mem:tasksdb
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

# Enable the H2 web console at http://localhost:8080/h2-console
spring.h2.console.enabled=true
```

### Production: MySQL

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/tasksdb
spring.datasource.username=root
spring.datasource.password=secret
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQLDialect
```

### Production: PostgreSQL

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/tasksdb
spring.datasource.username=postgres
spring.datasource.password=secret
```

> **`ddl-auto` values:** `update` (adjust schema to entities — convenient in dev), `validate` (check only — safe in prod), `none` (do nothing — use with migrations like Flyway/Liquibase in real projects), `create-drop` (recreate each run — tests only). Never use `update`/`create-drop` against a production database.

---

## 5.4 Define the entity

Annotate the model so JPA maps it to a table.

```java
package com.example.demo.model;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "tasks")
public class Task {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(nullable = false)
    private boolean completed = false;

    @Column(name = "created_at", updatable = false)
    private Instant createdAt = Instant.now();

    protected Task() {}   // JPA requires a no-arg constructor

    public Task(String title) { this.title = title; }

    // getters and setters
    public Long getId() { return id; }
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
    public boolean isCompleted() { return completed; }
    public void setCompleted(boolean completed) { this.completed = completed; }
    public Instant getCreatedAt() { return createdAt; }
}
```

Note the imports use **`jakarta.persistence`** (not `javax`) — that's the Spring Boot 3.x / Jakarta EE 9+ namespace.

---

## 5.5 The repository

Extend `JpaRepository<Entity, IdType>` and you get CRUD methods for free.

```java
package com.example.demo.repository;

import com.example.demo.model.Task;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface TaskRepository extends JpaRepository<Task, Long> {
    // Derived query — Spring generates the SQL from the method name
    List<Task> findByCompleted(boolean completed);

    List<Task> findByTitleContainingIgnoreCase(String keyword);
}
```

`JpaRepository` already provides `save`, `findById`, `findAll`, `deleteById`, `count`, and more. The two methods above are **derived queries** — Spring parses the method name and builds the query.

### Custom queries with `@Query`

```java
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

@Query("SELECT t FROM Task t WHERE t.completed = false ORDER BY t.createdAt DESC")
List<Task> findPendingNewestFirst();

@Query(value = "SELECT * FROM tasks WHERE title LIKE %:kw%", nativeQuery = true)
List<Task> searchNative(@Param("kw") String keyword);
```

---

## 5.6 Update the service to use JPA

```java
package com.example.demo.service;

import com.example.demo.dto.*;
import com.example.demo.model.Task;
import com.example.demo.repository.TaskRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;

@Service
@Transactional
public class TaskService {
    private final TaskRepository repository;

    public TaskService(TaskRepository repository) {
        this.repository = repository;
    }

    @Transactional(readOnly = true)
    public List<TaskResponse> getAll() {
        return repository.findAll().stream().map(this::toResponse).toList();
    }

    public TaskResponse create(CreateTaskRequest request) {
        Task task = new Task(request.title());
        return toResponse(repository.save(task));
    }

    public TaskResponse toggleComplete(Long id) {
        Task task = repository.findById(id)
            .orElseThrow(() -> new RuntimeException("Task not found: " + id));
        task.setCompleted(!task.isCompleted());
        return toResponse(repository.save(task));
    }

    public void delete(Long id) { repository.deleteById(id); }

    private TaskResponse toResponse(Task t) {
        return new TaskResponse(t.getId(), t.getTitle(), t.isCompleted());
    }
}
```

`@Transactional` wraps each method in a database transaction: changes commit together or roll back on error. Mark read methods `readOnly = true` for a performance hint.

---

## 5.7 Pagination and sorting

For large datasets, never return everything at once. Spring Data has built-in paging.

### Repository — no changes needed

`JpaRepository` already has `findAll(Pageable pageable)`.

### Controller

```java
import org.springframework.data.domain.*;
import org.springframework.web.bind.annotation.*;

@GetMapping("/api/tasks/page")
public Page<TaskResponse> getPage(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size,
        @RequestParam(defaultValue = "createdAt") String sortBy,
        @RequestParam(defaultValue = "desc") String direction) {

    Sort sort = direction.equalsIgnoreCase("asc")
            ? Sort.by(sortBy).ascending()
            : Sort.by(sortBy).descending();

    Pageable pageable = PageRequest.of(page, size, sort);
    return repository.findAll(pageable).map(this::toResponse);
}
```

Call it: `GET /api/tasks/page?page=0&size=5&sortBy=title&direction=asc`.

The `Page<T>` response includes useful metadata:

```json
{
  "content": [ ... ],
  "totalElements": 42,
  "totalPages": 9,
  "number": 0,
  "size": 5,
  "first": true,
  "last": false
}
```

---

## 5.8 Relationships

**Use case: a Task belongs to a User.**

```java
@Entity
public class Task {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;
    // ...
}

@Entity
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Task> tasks = new ArrayList<>();
    // ...
}
```

| Annotation | Relationship |
|------------|--------------|
| `@OneToMany` | One user → many tasks |
| `@ManyToOne` | Many tasks → one user |
| `@OneToOne` | One-to-one (e.g., user ↔ profile) |
| `@ManyToMany` | Many-to-many (e.g., tasks ↔ tags) |

> **Tip:** default to `FetchType.LAZY` for `@ManyToOne`/`@OneToMany` to avoid loading data you don't need. Beware the "N+1 query" problem — use `JOIN FETCH` or entity graphs when you know you need related data.

---

## Mini-project: a Product Catalog with JPA

1. Create an entity `Product { id, name, price, category, stock }`.
2. Create `ProductRepository extends JpaRepository<Product, Long>` with:
   - `findByCategory(String category)`
   - `findByPriceLessThan(BigDecimal max)`
3. Build CRUD endpoints plus a paginated `GET /api/products/page`.
4. Use H2 and open the H2 console to inspect the data.
5. Seed a few products at startup with a `CommandLineRunner` bean.

## Key takeaways

- Spring Data JPA maps `@Entity` classes to tables and generates repository implementations.
- Extend `JpaRepository` for instant CRUD; add derived queries by method name or `@Query` for custom SQL/JPQL.
- Use `@Transactional` at the service layer.
- Page large results with `Pageable`/`PageRequest` and sort with `Sort`.
- Use H2 for development, MySQL/PostgreSQL for production; control schema with `ddl-auto` (and migrations in real projects).

Next → [6. Validation & Exception Handling](06-validation-exception-handling.md)
