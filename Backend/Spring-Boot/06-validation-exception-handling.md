# 6. Validation & Exception Handling

[← Back](05-data-access-jpa.md) | [Index](README.md) | Next → [7. Security with JWT](07-security-jwt.md)

Real APIs must reject bad input and return clear, consistent errors. This section covers **bean validation** and **global exception handling**.

---

## 6.1 Bean validation

Add the validation starter:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

Annotate your request DTO with constraints:

```java
package com.example.demo.dto;

import jakarta.validation.constraints.*;

public record CreateTaskRequest(

    @NotBlank(message = "Title is required")
    @Size(min = 3, max = 100, message = "Title must be 3-100 characters")
    String title,

    @Size(max = 500, message = "Description too long")
    String description,

    @NotNull(message = "Priority is required")
    @Min(value = 1, message = "Priority must be at least 1")
    @Max(value = 5, message = "Priority must be at most 5")
    Integer priority
) {}
```

### Common constraints

| Annotation | Checks |
|------------|--------|
| `@NotNull` | Not null |
| `@NotBlank` | Not null and not empty/whitespace (strings) |
| `@NotEmpty` | Not null and not empty (collections/strings) |
| `@Size(min, max)` | Length/size in range |
| `@Min` / `@Max` | Numeric bounds |
| `@Email` | Valid email format |
| `@Pattern(regexp)` | Matches a regex |
| `@Past` / `@Future` | Date/time constraints |
| `@Positive` / `@Negative` | Sign checks |

### Trigger validation with `@Valid`

```java
@PostMapping
public ResponseEntity<TaskResponse> create(@Valid @RequestBody CreateTaskRequest request) {
    return ResponseEntity.status(HttpStatus.CREATED).body(service.create(request));
}
```

When validation fails, Spring throws `MethodArgumentNotValidException` and returns **400 Bad Request**. Next we'll shape that into a clean response.

---

## 6.2 Custom exceptions

Define meaningful, domain-specific exceptions instead of raw `RuntimeException`.

```java
package com.example.demo.exception;

public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

public class DuplicateResourceException extends RuntimeException {
    public DuplicateResourceException(String message) {
        super(message);
    }
}
```

Use them in the service:

```java
public TaskResponse getById(Long id) {
    Task task = repository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("Task not found with id " + id));
    return toResponse(task);
}
```

---

## 6.3 A consistent error response

Define one shape for all errors so clients can rely on it.

```java
package com.example.demo.exception;

import java.time.Instant;
import java.util.Map;

public record ApiError(
    Instant timestamp,
    int status,
    String error,
    String message,
    String path,
    Map<String, String> fieldErrors   // populated for validation errors
) {}
```

---

## 6.4 Global exception handling with `@RestControllerAdvice`

Instead of try/catch in every controller, centralize error handling in one class. `@RestControllerAdvice` intercepts exceptions thrown by any controller.

```java
package com.example.demo.exception;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.http.*;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

import java.time.Instant;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    // 404 — our custom not-found exception
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(ResourceNotFoundException ex,
                                                   HttpServletRequest req) {
        ApiError error = new ApiError(
            Instant.now(), HttpStatus.NOT_FOUND.value(), "Not Found",
            ex.getMessage(), req.getRequestURI(), Map.of());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    // 409 — duplicate
    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ApiError> handleDuplicate(DuplicateResourceException ex,
                                                    HttpServletRequest req) {
        ApiError error = new ApiError(
            Instant.now(), HttpStatus.CONFLICT.value(), "Conflict",
            ex.getMessage(), req.getRequestURI(), Map.of());
        return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
    }

    // 400 — bean validation errors, with per-field messages
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> handleValidation(MethodArgumentNotValidException ex,
                                                     HttpServletRequest req) {
        Map<String, String> fieldErrors = new HashMap<>();
        for (FieldError fe : ex.getBindingResult().getFieldErrors()) {
            fieldErrors.put(fe.getField(), fe.getDefaultMessage());
        }
        ApiError error = new ApiError(
            Instant.now(), HttpStatus.BAD_REQUEST.value(), "Bad Request",
            "Validation failed", req.getRequestURI(), fieldErrors);
        return ResponseEntity.badRequest().body(error);
    }

    // 500 — catch-all safety net
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleGeneric(Exception ex, HttpServletRequest req) {
        ApiError error = new ApiError(
            Instant.now(), HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "Internal Server Error", "Something went wrong",
            req.getRequestURI(), Map.of());
        // log the real exception server-side; don't leak details to the client
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

### The flow

```
  Controller throws ResourceNotFoundException
              │
              ▼
  @RestControllerAdvice intercepts it
              │
              ▼
  Matching @ExceptionHandler builds an ApiError
              │
              ▼
  Client receives 404 + clean JSON body
```

### Example error responses

**Validation failure (400):**

```json
{
  "timestamp": "2026-09-22T10:15:30Z",
  "status": 400,
  "error": "Bad Request",
  "message": "Validation failed",
  "path": "/api/tasks",
  "fieldErrors": {
    "title": "Title must be 3-100 characters",
    "priority": "Priority is required"
  }
}
```

**Not found (404):**

```json
{
  "timestamp": "2026-09-22T10:16:00Z",
  "status": 404,
  "error": "Not Found",
  "message": "Task not found with id 99",
  "path": "/api/tasks/99",
  "fieldErrors": {}
}
```

---

## 6.5 Validating path variables and request params

For constraints directly on method parameters, add `@Validated` to the controller class:

```java
import org.springframework.validation.annotation.Validated;
import jakarta.validation.constraints.Positive;

@RestController
@Validated
@RequestMapping("/api/tasks")
public class TaskController {

    @GetMapping("/{id}")
    public TaskResponse get(@PathVariable @Positive Long id) {
        return service.getById(id);
    }
}
```

A violation throws `ConstraintViolationException` — add a handler for it in your advice class if you use this.

---

## Mini-project: a robust User registration endpoint

1. Create `RegisterRequest { username, email, password, age }` with validation:
   - `username`: `@NotBlank`, 3–20 chars
   - `email`: `@Email`, `@NotBlank`
   - `password`: `@Size(min = 8)`, `@Pattern` requiring a digit and a letter
   - `age`: `@Min(13)`
2. In the service, throw `DuplicateResourceException` if the email already exists.
3. Wire up the `GlobalExceptionHandler`.
4. Test: send invalid data and confirm you get a 400 with per-field messages; send a duplicate email and confirm a 409.

## Key takeaways

- Add constraints to DTOs and trigger them with `@Valid`.
- Create domain-specific exceptions instead of throwing raw `RuntimeException`.
- Centralize error handling in a `@RestControllerAdvice` class for consistent responses.
- Return a stable error shape (`ApiError`) with status, message, path, and field errors.
- Never leak internal exception details to clients; log them server-side instead.

Next → [7. Security with JWT](07-security-jwt.md)
