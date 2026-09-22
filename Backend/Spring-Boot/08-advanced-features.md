# 8. Advanced Features

[← Back](07-security-jwt.md) | [Index](README.md) | Next → [9. Testing](09-testing.md)

These features make your app configurable across environments and capable of background work.

---

## 8.1 Profiles

**Profiles** let you have different configuration per environment (dev, test, prod).

### Profile-specific property files

Spring Boot loads `application.properties` always, then overlays a profile file if active:

```
application.properties          # shared defaults
application-dev.properties      # dev overrides
application-prod.properties     # prod overrides
```

`application-dev.properties`:

```properties
spring.datasource.url=jdbc:h2:mem:devdb
spring.jpa.show-sql=true
logging.level.com.example=DEBUG
```

`application-prod.properties`:

```properties
spring.datasource.url=jdbc:postgresql://db:5432/appdb
spring.jpa.show-sql=false
logging.level.com.example=WARN
```

### Activating a profile

```properties
# in application.properties
spring.profiles.active=dev
```

Or at runtime (overrides the file):

```bash
java -jar app.jar --spring.profiles.active=prod
# or via environment variable
export SPRING_PROFILES_ACTIVE=prod
```

### Profile-specific beans

```java
@Configuration
public class DataSeedConfig {

    @Bean
    @Profile("dev")   // only created when 'dev' is active
    public CommandLineRunner seedData(TaskRepository repo) {
        return args -> {
            repo.save(new Task("Sample task 1"));
            repo.save(new Task("Sample task 2"));
        };
    }
}
```

---

## 8.2 Type-safe configuration with `@ConfigurationProperties`

`@Value` is fine for one-off values. For groups of related settings, bind them to a typed class.

```properties
app.notification.enabled=true
app.notification.sender=noreply@example.com
app.notification.retry-count=3
```

```java
package com.example.demo.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "app.notification")
public class NotificationProperties {
    private boolean enabled;
    private String sender;
    private int retryCount;

    // getters and setters (required for binding)
    public boolean isEnabled() { return enabled; }
    public void setEnabled(boolean enabled) { this.enabled = enabled; }
    public String getSender() { return sender; }
    public void setSender(String sender) { this.sender = sender; }
    public int getRetryCount() { return retryCount; }
    public void setRetryCount(int retryCount) { this.retryCount = retryCount; }
}
```

Inject and use it:

```java
@Service
public class NotificationService {
    private final NotificationProperties props;
    public NotificationService(NotificationProperties props) { this.props = props; }

    public void notifyUser(String message) {
        if (!props.isEnabled()) return;
        // send from props.getSender(), retry props.getRetryCount() times
    }
}
```

**Benefits over `@Value`:** grouping, type safety, IDE autocomplete, and validation (add `@Validated` + constraints).

---

## 8.3 Scheduling tasks

Run code on a timer. Enable scheduling once:

```java
@SpringBootApplication
@EnableScheduling
public class DemoApplication { ... }
```

Then annotate methods with `@Scheduled`:

```java
@Component
public class ReportScheduler {

    // fixed rate: every 60 seconds (measured from start of each run)
    @Scheduled(fixedRate = 60000)
    public void everyMinute() {
        System.out.println("Running periodic job");
    }

    // fixed delay: 30s AFTER the previous run finishes
    @Scheduled(fixedDelay = 30000)
    public void afterPrevious() { ... }

    // cron: every day at 2:00 AM
    @Scheduled(cron = "0 0 2 * * *")
    public void nightlyCleanup() {
        System.out.println("Cleaning up old data");
    }
}
```

### Cron format

```
 ┌─ second (0-59)
 │ ┌─ minute (0-59)
 │ │ ┌─ hour (0-23)
 │ │ │ ┌─ day of month (1-31)
 │ │ │ │ ┌─ month (1-12)
 │ │ │ │ │ ┌─ day of week (0-7, 0/7 = Sunday)
 │ │ │ │ │ │
 0 0 2 * * *     → 2:00:00 AM every day
 0 */15 * * * *  → every 15 minutes
```

**Use case:** send a daily digest email, purge expired tokens, or refresh a cache overnight.

---

## 8.4 Asynchronous processing

Long tasks (sending email, calling a slow API) shouldn't block the request thread. Run them asynchronously.

Enable async:

```java
@SpringBootApplication
@EnableAsync
public class DemoApplication { ... }
```

Mark a method `@Async` — it returns immediately and runs on a separate thread:

```java
@Service
public class EmailService {

    @Async
    public void sendWelcomeEmail(String to) {
        // simulate a slow send
        try { Thread.sleep(3000); } catch (InterruptedException ignored) {}
        System.out.println("Sent welcome email to " + to);
    }

    // Return a CompletableFuture when you need the result later
    @Async
    public CompletableFuture<String> fetchReport(Long id) {
        // ... slow work ...
        return CompletableFuture.completedFuture("Report " + id);
    }
}
```

Now a controller can return instantly while the email sends in the background:

```java
@PostMapping("/register")
public String register(@RequestBody RegisterRequest req) {
    userService.create(req);
    emailService.sendWelcomeEmail(req.email());   // fire and forget
    return "Registered — check your inbox shortly";
}
```

### Configure the thread pool

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}
```

> **Caveat:** `@Async` and `@Scheduled` only work when called through a Spring proxy — calling an `@Async` method from *within the same class* runs it synchronously. Call it from another bean.

---

## 8.5 Caching (bonus)

Speed up repeat calls by caching results.

```java
@SpringBootApplication
@EnableCaching
public class DemoApplication { ... }

@Service
public class TaskService {
    @Cacheable("tasks")               // cache result by argument
    public TaskResponse getById(Long id) { ... }

    @CacheEvict(value = "tasks", key = "#id")   // clear on update
    public void delete(Long id) { ... }
}
```

For production, back this with Redis (`spring-boot-starter-data-redis`) instead of the default in-memory cache.

---

## Mini-project: a scheduled digest with async email

1. Add a `NotificationProperties` config bound to `app.notification.*`.
2. Create an `@Async` `EmailService.sendDigest(String user, List<Task> pending)`.
3. Add a `@Scheduled(cron = "0 0 8 * * *")` job that finds each user's pending tasks and sends a digest asynchronously.
4. Use a `dev` profile that runs the schedule every 30 seconds instead, so you can see it work.

## Key takeaways

- Profiles provide per-environment config via `application-{profile}.properties` and `@Profile` beans.
- `@ConfigurationProperties` binds grouped settings to type-safe classes — prefer it over scattered `@Value`.
- `@Scheduled` runs jobs on fixed rate/delay or cron schedules (needs `@EnableScheduling`).
- `@Async` runs work off the request thread (needs `@EnableAsync`); configure a thread pool for control.
- Remember the self-invocation caveat: proxy-based features must be called from another bean.

Next → [9. Testing](09-testing.md)
