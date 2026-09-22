# 9. Testing

[← Back](08-advanced-features.md) | [Index](README.md) | Next → [10. Deployment](10-deployment.md)

Tests give you the confidence to change code without breaking it. Spring Boot ships with a rich testing stack via `spring-boot-starter-test` (JUnit 5, Mockito, AssertJ, Spring Test).

---

## 9.1 The testing pyramid

```
              ▲   fewer, slower, broader
              │        ┌───────────────┐
              │        │  E2E / API    │   full app, real HTTP
              │      ┌─┴───────────────┴─┐
              │      │  Integration      │   several layers + DB
              │   ┌──┴───────────────────┴──┐
              │   │      Unit tests          │   one class, mocked deps
              ▼   └──────────────────────────┘
                  many, fast, focused
```

Write many fast unit tests, fewer integration tests, and a small number of end-to-end tests.

The test starter is already included:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

Run tests with `./mvnw test` (Maven) or `./gradlew test` (Gradle).

---

## 9.2 Unit testing with JUnit 5

Unit tests verify one class in isolation. They're plain JUnit — no Spring context needed, so they're fast.

```java
package com.example.demo;

import org.junit.jupiter.api.*;
import static org.assertj.core.api.Assertions.*;

class PriceCalculatorTest {

    PriceCalculator calc;

    @BeforeEach
    void setUp() {
        calc = new PriceCalculator();
    }

    @Test
    @DisplayName("applies 10% discount over $100")
    void appliesDiscount() {
        double result = calc.finalPrice(150.0);
        assertThat(result).isEqualTo(135.0);
    }

    @Test
    void noDiscountUnderThreshold() {
        assertThat(calc.finalPrice(50.0)).isEqualTo(50.0);
    }

    @Test
    void rejectsNegativePrice() {
        assertThatThrownBy(() -> calc.finalPrice(-1))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

### Useful JUnit 5 annotations

| Annotation | Purpose |
|------------|---------|
| `@Test` | Marks a test method |
| `@BeforeEach` / `@AfterEach` | Run before/after each test |
| `@BeforeAll` / `@AfterAll` | Run once before/after all tests (static) |
| `@DisplayName` | Human-readable test name |
| `@Disabled` | Skip a test |
| `@ParameterizedTest` | Run the same test with many inputs |

### Parameterized test

```java
@ParameterizedTest
@ValueSource(ints = {2, 4, 6, 100})
void detectsEvenNumbers(int number) {
    assertThat(number % 2).isZero();
}
```

---

## 9.3 Mocking dependencies with Mockito

To test a service in isolation, replace its dependencies (like the repository) with **mocks** — fakes you program to return canned values.

```java
package com.example.demo.service;

import com.example.demo.dto.*;
import com.example.demo.exception.ResourceNotFoundException;
import com.example.demo.model.Task;
import com.example.demo.repository.TaskRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class TaskServiceTest {

    @Mock
    TaskRepository repository;          // fake repository

    @InjectMocks
    TaskService service;                // real service, mock injected

    @Test
    void createSavesAndReturnsTask() {
        var request = new CreateTaskRequest("Write tests");
        var saved = new Task("Write tests");
        // program the mock
        when(repository.save(any(Task.class))).thenReturn(saved);

        TaskResponse result = service.create(request);

        assertThat(result.title()).isEqualTo("Write tests");
        verify(repository).save(any(Task.class));   // confirm the interaction
    }

    @Test
    void getByIdThrowsWhenMissing() {
        when(repository.findById(99L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> service.getById(99L))
            .isInstanceOf(ResourceNotFoundException.class)
            .hasMessageContaining("99");
    }
}
```

**Key Mockito tools:**

- `@Mock` — create a mock; `@InjectMocks` — build the class under test with mocks injected.
- `when(...).thenReturn(...)` — stub a method's return value.
- `verify(...)` — assert a method was called (optionally a number of times).
- `any()`, `eq()` — argument matchers.

---

## 9.4 Testing the web layer with `@WebMvcTest`

`@WebMvcTest` loads only the web layer (controllers, JSON, validation) — not the full app. You mock the service.

```java
package com.example.demo.controller;

import com.example.demo.dto.TaskResponse;
import com.example.demo.service.TaskService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.web.servlet.MockMvc;

import java.util.List;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(TaskController.class)
class TaskControllerTest {

    @Autowired MockMvc mockMvc;      // simulates HTTP calls without a real server
    @MockBean TaskService service;   // replace the real service with a mock

    @Test
    void getAllReturnsJsonArray() throws Exception {
        when(service.getAll()).thenReturn(List.of(
            new TaskResponse(1L, "Buy milk", false)));

        mockMvc.perform(get("/api/tasks"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$[0].title").value("Buy milk"))
            .andExpect(jsonPath("$[0].completed").value(false));
    }

    @Test
    void createReturns201() throws Exception {
        when(service.create(org.mockito.ArgumentMatchers.any()))
            .thenReturn(new TaskResponse(1L, "New task", false));

        mockMvc.perform(post("/api/tasks")
                .contentType("application/json")
                .content("{\"title\":\"New task\"}"))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(1));
    }

    @Test
    void rejectsInvalidInput() throws Exception {
        mockMvc.perform(post("/api/tasks")
                .contentType("application/json")
                .content("{\"title\":\"\"}"))   // blank title fails @NotBlank
            .andExpect(status().isBadRequest());
    }
}
```

---

## 9.5 Testing the data layer with `@DataJpaTest`

`@DataJpaTest` loads only JPA components and uses an in-memory H2 database by default. Perfect for testing repositories and derived queries.

```java
package com.example.demo.repository;

import com.example.demo.model.Task;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;

import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
class TaskRepositoryTest {

    @Autowired TaskRepository repository;

    @Test
    void findByCompletedReturnsOnlyMatching() {
        Task done = new Task("Done task");
        done.setCompleted(true);
        repository.save(done);
        repository.save(new Task("Pending task"));

        List<Task> completed = repository.findByCompleted(true);

        assertThat(completed).hasSize(1);
        assertThat(completed.get(0).getTitle()).isEqualTo("Done task");
    }
}
```

---

## 9.6 Full integration test with `@SpringBootTest`

`@SpringBootTest` starts the **entire** application context. Combine it with `MockMvc` (or a real HTTP client) to test the app end to end.

```java
package com.example.demo;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class TaskApiIntegrationTest {

    @Autowired MockMvc mockMvc;

    @Test
    void createThenFetchTask() throws Exception {
        // create
        mockMvc.perform(post("/api/tasks")
                .contentType("application/json")
                .content("{\"title\":\"Integration task\"}"))
            .andExpect(status().isCreated());

        // fetch and confirm it's there
        mockMvc.perform(get("/api/tasks"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$[0].title").value("Integration task"));
    }
}
```

### Realistic databases with Testcontainers (advanced)

Testing against real MySQL/PostgreSQL (not H2) catches dialect-specific bugs. [Testcontainers](https://testcontainers.com/) spins up a throwaway database in Docker for the test.

```java
@SpringBootTest
@Testcontainers
class TaskApiPostgresTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    // ... tests run against a real PostgreSQL in Docker
}
```

---

## 9.7 Which test annotation to use?

| Goal | Annotation | Loads |
|------|------------|-------|
| Test one class, no Spring | (none, plain JUnit) | Nothing |
| Test a service with mocked deps | `@ExtendWith(MockitoExtension.class)` | Nothing |
| Test controllers + JSON | `@WebMvcTest` | Web layer only |
| Test repositories/queries | `@DataJpaTest` | JPA + H2 |
| Test the whole app | `@SpringBootTest` | Full context |

**Rule of thumb:** use the narrowest slice that covers what you're testing — it's faster and more focused.

---

## Mini-project: test the Task API

1. Write a **unit test** for `TaskService.toggleComplete` using Mockito (verify it flips the flag and saves).
2. Write a **`@WebMvcTest`** for `TaskController` covering: list, create (201), and invalid input (400).
3. Write a **`@DataJpaTest`** for a custom derived query.
4. Write one **`@SpringBootTest`** integration test that creates a task and reads it back.
5. Run `./mvnw test` and confirm all pass.

## Key takeaways

- Follow the testing pyramid: mostly fast unit tests, some integration, few end-to-end.
- Mock dependencies with Mockito (`@Mock`, `@InjectMocks`, `when`, `verify`).
- Use test slices (`@WebMvcTest`, `@DataJpaTest`) for speed; `@SpringBootTest` for full integration.
- `MockMvc` tests HTTP behavior without starting a real server.
- Testcontainers gives you production-like databases in tests.

Next → [10. Deployment](10-deployment.md)
