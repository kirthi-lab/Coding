# 4. REST API Development

[← Back](03-basic-concepts.md) | [Index](README.md) | Next → [5. Data Access with Spring Data JPA](05-data-access-jpa.md)

This is where Spring Boot shines. You'll build a full REST API with the classic three-layer design: controller → service → repository.

---

## 4.1 REST refresher

REST APIs expose **resources** (like tasks or users) over HTTP. Each resource has a URL, and you act on it with HTTP methods:

| Method | Purpose | Example | Success status |
|--------|---------|---------|----------------|
| `GET` | Read | `GET /api/tasks` | 200 OK |
| `POST` | Create | `POST /api/tasks` | 201 Created |
| `PUT` | Replace | `PUT /api/tasks/1` | 200 OK |
| `PATCH` | Partial update | `PATCH /api/tasks/1` | 200 OK |
| `DELETE` | Remove | `DELETE /api/tasks/1` | 204 No Content |

---

## 4.2 The layered flow

```
   Client
     │  POST /api/tasks  { "title": "Buy milk" }
     ▼
 ┌─────────────┐   request DTO    ┌──────────┐    entity    ┌──────────────┐
 │ Controller  │ ───────────────► │ Service  │ ───────────► │ Repository   │
 │ (HTTP layer)│ ◄─────────────── │ (logic)  │ ◄─────────── │ (data access)│
 └─────────────┘   response DTO   └──────────┘    entity    └──────────────┘
     │  201 Created  { "id": 1, "title": "Buy milk" }
     ▼
   Client
```

We'll build a **Task API** in memory first (no database yet — that comes in Section 5), so this section stays self-contained and runnable.

---

## 4.3 The model

```java
package com.example.demo.model;

public class Task {
    private Long id;
    private String title;
    private boolean completed;

    public Task() {}
    public Task(Long id, String title, boolean completed) {
        this.id = id; this.title = title; this.completed = completed;
    }
    // getters and setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
    public boolean isCompleted() { return completed; }
    public void setCompleted(boolean completed) { this.completed = completed; }
}
```

---

## 4.4 DTOs: why not expose the entity directly?

A **DTO** (Data Transfer Object) is the shape of data you accept and return over HTTP. Keeping DTOs separate from your internal model means the API contract doesn't leak internal details and can evolve independently.

Java **records** make DTOs concise:

```java
package com.example.demo.dto;

// what the client sends to create a task
public record CreateTaskRequest(String title) {}

// what the API returns
public record TaskResponse(Long id, String title, boolean completed) {}
```

---

## 4.5 The repository (in-memory for now)

```java
package com.example.demo.repository;

import com.example.demo.model.Task;
import org.springframework.stereotype.Repository;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

@Repository
public class InMemoryTaskRepository {
    private final Map<Long, Task> store = new ConcurrentHashMap<>();
    private final AtomicLong idSeq = new AtomicLong();

    public List<Task> findAll() { return new ArrayList<>(store.values()); }

    public Optional<Task> findById(Long id) { return Optional.ofNullable(store.get(id)); }

    public Task save(Task task) {
        if (task.getId() == null) task.setId(idSeq.incrementAndGet());
        store.put(task.getId(), task);
        return task;
    }

    public boolean deleteById(Long id) { return store.remove(id) != null; }
}
```

---

## 4.6 The service

Business logic lives here. It maps between entities and DTOs and enforces rules.

```java
package com.example.demo.service;

import com.example.demo.dto.*;
import com.example.demo.model.Task;
import com.example.demo.repository.InMemoryTaskRepository;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class TaskService {
    private final InMemoryTaskRepository repository;

    public TaskService(InMemoryTaskRepository repository) {
        this.repository = repository;
    }

    public List<TaskResponse> getAll() {
        return repository.findAll().stream().map(this::toResponse).toList();
    }

    public TaskResponse getById(Long id) {
        Task task = repository.findById(id)
            .orElseThrow(() -> new RuntimeException("Task not found: " + id));
        return toResponse(task);
    }

    public TaskResponse create(CreateTaskRequest request) {
        Task task = new Task(null, request.title(), false);
        return toResponse(repository.save(task));
    }

    public TaskResponse toggleComplete(Long id) {
        Task task = repository.findById(id)
            .orElseThrow(() -> new RuntimeException("Task not found: " + id));
        task.setCompleted(!task.isCompleted());
        return toResponse(repository.save(task));
    }

    public void delete(Long id) {
        if (!repository.deleteById(id)) {
            throw new RuntimeException("Task not found: " + id);
        }
    }

    private TaskResponse toResponse(Task t) {
        return new TaskResponse(t.getId(), t.getTitle(), t.isCompleted());
    }
}
```

(We'll replace the crude `RuntimeException` with proper exception handling in [Section 6](06-validation-exception-handling.md).)

---

## 4.7 The controller

The controller maps HTTP requests to service calls and shapes HTTP responses.

```java
package com.example.demo.controller;

import com.example.demo.dto.*;
import com.example.demo.service.TaskService;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.net.URI;
import java.util.List;

@RestController
@RequestMapping("/api/tasks")
public class TaskController {
    private final TaskService service;

    public TaskController(TaskService service) {
        this.service = service;
    }

    // GET /api/tasks
    @GetMapping
    public List<TaskResponse> getAll() {
        return service.getAll();
    }

    // GET /api/tasks/{id}
    @GetMapping("/{id}")
    public TaskResponse getById(@PathVariable Long id) {
        return service.getById(id);
    }

    // POST /api/tasks
    @PostMapping
    public ResponseEntity<TaskResponse> create(@RequestBody CreateTaskRequest request) {
        TaskResponse created = service.create(request);
        return ResponseEntity
            .created(URI.create("/api/tasks/" + created.id()))  // 201 + Location header
            .body(created);
    }

    // PATCH /api/tasks/{id}/toggle
    @PatchMapping("/{id}/toggle")
    public TaskResponse toggle(@PathVariable Long id) {
        return service.toggleComplete(id);
    }

    // DELETE /api/tasks/{id}
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        service.delete(id);
        return ResponseEntity.noContent().build();  // 204
    }
}
```

### Key annotations explained

| Annotation | Purpose |
|------------|---------|
| `@RestController` | Combines `@Controller` + `@ResponseBody`; returns data as JSON |
| `@RequestMapping("/api/tasks")` | Base path for all methods in the class |
| `@GetMapping`, `@PostMapping`, etc. | Map HTTP methods to handler methods |
| `@PathVariable` | Bind a URL segment (`/{id}`) to a parameter |
| `@RequestBody` | Deserialize the JSON body into an object |
| `@RequestParam` | Bind a query parameter (`?status=done`) |
| `ResponseEntity<T>` | Full control over status code, headers, and body |

---

## 4.8 Test it with curl

Run the app, then:

```bash
# Create a task
curl -X POST http://localhost:8080/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Buy milk"}'
# → 201 Created  {"id":1,"title":"Buy milk","completed":false}

# List tasks
curl http://localhost:8080/api/tasks
# → [{"id":1,"title":"Buy milk","completed":false}]

# Toggle complete
curl -X PATCH http://localhost:8080/api/tasks/1/toggle
# → {"id":1,"title":"Buy milk","completed":true}

# Delete
curl -X DELETE http://localhost:8080/api/tasks/1
# → 204 No Content
```

> On Windows PowerShell, use `curl.exe` (not the `curl` alias) or `Invoke-RestMethod` to avoid quoting issues.

---

## 4.9 Handling query parameters

**Use case: filter tasks by completion status.**

```java
@GetMapping(params = "completed")
public List<TaskResponse> getByStatus(@RequestParam boolean completed) {
    return service.getAll().stream()
        .filter(t -> t.completed() == completed)
        .toList();
}
```

Call it: `GET /api/tasks?completed=true`.

---

## Mini-project: a Bookmarks API

Build a REST API for bookmarks with the same layered design:

- **Model:** `Bookmark { id, url, title, tags }`
- **Endpoints:**
  - `POST /api/bookmarks` — add a bookmark
  - `GET /api/bookmarks` — list all
  - `GET /api/bookmarks/{id}` — get one
  - `GET /api/bookmarks?tag=java` — filter by tag
  - `DELETE /api/bookmarks/{id}` — remove
- Use records for the DTOs and an in-memory repository.
- Test every endpoint with curl or Postman.

## Key takeaways

- Structure APIs into controller (HTTP), service (logic), and repository (data) layers.
- Use DTOs (records) to decouple the API contract from internal models.
- `@RestController` + mapping annotations wire HTTP methods to Java methods.
- Use `ResponseEntity` for precise control of status codes and headers.

Next → [5. Data Access with Spring Data JPA](05-data-access-jpa.md)
