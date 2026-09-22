# 3. Basic Concepts

[← Back](02-setup-installation.md) | [Index](README.md) | Next → [4. REST API Development](04-rest-api.md)

Now that the app runs, let's understand what's inside it: the project structure, configuration, the main class, and the bean/DI model.

---

## 3.1 Project structure

A generated Maven project looks like this:

```
demo/
├── mvnw, mvnw.cmd              # Maven wrapper scripts
├── pom.xml                     # dependencies & build config
└── src/
    ├── main/
    │   ├── java/
    │   │   └── com/example/demo/
    │   │       └── DemoApplication.java   # entry point
    │   └── resources/
    │       ├── application.properties      # configuration
    │       ├── static/                      # static web assets (css, js)
    │       └── templates/                   # server-side templates
    └── test/
        └── java/
            └── com/example/demo/
                └── DemoApplicationTests.java
```

**Package convention:** put your code under the base package (`com.example.demo`) or sub-packages. Spring Boot auto-scans this package and below for components. A common layout:

```
com.example.demo
├── DemoApplication.java
├── controller/     # REST controllers
├── service/        # business logic
├── repository/     # data access
├── model/ (or entity/)  # domain/JPA entities
├── dto/            # request/response objects
└── config/         # configuration classes
```

We'll fill these in over the next sections.

---

## 3.2 The main class: `@SpringBootApplication`

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

`@SpringBootApplication` is a convenience annotation that combines three:

| Annotation | Role |
|------------|------|
| `@Configuration` | Marks the class as a source of bean definitions |
| `@EnableAutoConfiguration` | Turns on auto-configuration based on the classpath |
| `@ComponentScan` | Scans this package (and sub-packages) for components |

`SpringApplication.run(...)` bootstraps everything: creates the application context (the IoC container), performs auto-configuration, starts the embedded server, and wires your beans.

```
  main() 
    │
    ▼
  SpringApplication.run()
    │
    ├─► create ApplicationContext (IoC container)
    ├─► scan for @Component/@Service/@Controller beans
    ├─► apply auto-configuration
    └─► start embedded Tomcat on :8080
```

---

## 3.3 Beans, components, and stereotypes

A **bean** is an object managed by the Spring container. You mark a class as a bean with a **stereotype annotation**, and Spring creates and injects it wherever needed.

| Annotation | Meaning |
|------------|---------|
| `@Component` | Generic Spring-managed bean |
| `@Service` | A component holding business logic |
| `@Repository` | A component for data access (adds DB exception translation) |
| `@Controller` / `@RestController` | A component handling web requests |
| `@Configuration` + `@Bean` | Define beans manually in a config class |

**Example: a service bean injected into another bean.**

```java
@Service
public class GreetingService {
    public String greet(String name) {
        return "Hello, " + name + "!";
    }
}

@RestController
public class GreetingController {
    private final GreetingService greetingService;

    // constructor injection — Spring passes the GreetingService bean
    public GreetingController(GreetingService greetingService) {
        this.greetingService = greetingService;
    }

    @GetMapping("/greet/{name}")
    public String greet(@PathVariable String name) {
        return greetingService.greet(name);
    }
}
```

You never call `new GreetingService()` — Spring creates it once and injects it. This single instance (a **singleton** by default) is reused everywhere.

### Defining a bean manually with `@Bean`

When you can't annotate a class (e.g., it's from a library), define it in a config class:

```java
@Configuration
public class AppConfig {
    @Bean
    public RestClient restClient() {
        return RestClient.create();
    }
}
```

---

## 3.4 Configuration: `application.properties` and `application.yml`

External configuration lives in `src/main/resources`. You can use either `.properties` or `.yml`.

**`application.properties`:**

```properties
# Server
server.port=8081
server.servlet.context-path=/api

# Application name
spring.application.name=demo

# Logging
logging.level.org.springframework.web=INFO
logging.level.com.example.demo=DEBUG

# A custom property
app.welcome-message=Welcome to the Demo API
```

**Equivalent `application.yml`** (indentation-based, less repetition):

```yaml
server:
  port: 8081
  servlet:
    context-path: /api

spring:
  application:
    name: demo

logging:
  level:
    org.springframework.web: INFO
    com.example.demo: DEBUG

app:
  welcome-message: Welcome to the Demo API
```

### Reading configuration values

Inject a single value with `@Value`:

```java
@RestController
public class InfoController {
    @Value("${app.welcome-message}")
    private String welcomeMessage;

    @GetMapping("/info")
    public String info() {
        return welcomeMessage;
    }
}
```

(For grouped, type-safe config we'll use `@ConfigurationProperties` in [Section 8](08-advanced-features.md).)

---

## 3.5 Dependency injection styles

**Constructor injection (recommended):** dependencies are `final`, guaranteed set, and easy to test.

```java
@Service
public class OrderService {
    private final PaymentService payments;
    public OrderService(PaymentService payments) {  // no @Autowired needed for a single constructor
        this.payments = payments;
    }
}
```

Field injection (`@Autowired` on a field) and setter injection also exist, but constructor injection is preferred because it makes dependencies explicit and supports immutability.

---

## 3.6 DevTools & live reload

If you added **Spring Boot DevTools**, the app restarts automatically when you recompile. In most IDEs, saving and building triggers a fast restart, so you see changes without manually stopping and starting.

---

## Mini-project: a configurable greeting endpoint

Build a tiny app that ties together a controller, a service, and configuration.

1. In `application.properties`, add:
   ```properties
   app.greeting-prefix=Hey there
   ```
2. Create the service:
   ```java
   @Service
   public class GreetingService {
       private final String prefix;
       public GreetingService(@Value("${app.greeting-prefix}") String prefix) {
           this.prefix = prefix;
       }
       public String greet(String name) {
           return prefix + ", " + name + "!";
       }
   }
   ```
3. Create the controller:
   ```java
   @RestController
   public class GreetingController {
       private final GreetingService service;
       public GreetingController(GreetingService service) { this.service = service; }

       @GetMapping("/greet/{name}")
       public String greet(@PathVariable String name) {
           return service.greet(name);
       }
   }
   ```
4. Run the app and visit **http://localhost:8080/greet/Ada** → `Hey there, Ada!`
5. Change the property value, restart, and see the output change without touching Java code.

## Key takeaways

- `@SpringBootApplication` = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`.
- Beans are container-managed objects declared with stereotype annotations (`@Service`, `@Repository`, etc.).
- Prefer constructor injection.
- Externalize configuration in `application.properties`/`.yml`; read values with `@Value`.

Next → [4. REST API Development](04-rest-api.md)
