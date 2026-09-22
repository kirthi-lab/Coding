# 1. Introduction to Spring Boot

[Index](README.md) | Next → [2. Setup & Installation](02-setup-installation.md)

---

## 1.1 What is Spring Boot?

**Spring Boot** is an opinionated framework built on top of the **Spring Framework** that makes it fast and easy to create stand-alone, production-ready applications. Its motto is "just run" — you get a working app with an embedded web server, sensible defaults, and minimal configuration.

Before Spring Boot, starting a Spring project meant writing large amounts of XML configuration, manually choosing compatible library versions, and deploying to an external server. Spring Boot removes that friction with three core ideas:

1. **Auto-configuration** — Spring Boot looks at what's on your classpath and configures beans automatically. Add the web starter, and it configures an embedded Tomcat, JSON serialization, and an MVC stack for you.
2. **Starters** — curated dependency bundles. One dependency (`spring-boot-starter-web`) pulls in everything you need for web development at compatible versions.
3. **Embedded server** — your app packages a web server (Tomcat by default) inside the JAR. You run it with `java -jar app.jar`. No external server to install.

---

## 1.2 Spring Boot vs Spring Framework

Spring Boot does not replace the Spring Framework — it *uses* it and adds a productivity layer on top.

| Aspect | Spring Framework | Spring Boot |
|--------|------------------|-------------|
| Configuration | Manual (XML or Java config) | Auto-configuration with sensible defaults |
| Dependency management | You pick each library + version | Starters bundle compatible versions |
| Web server | Deploy WAR to external Tomcat/JBoss | Embedded server inside an executable JAR |
| Boilerplate | High | Low |
| Getting started | Slow, error-prone | Minutes via Spring Initializr |
| Production features | Add manually | Built-in (Actuator: health, metrics) |

```
        ┌───────────────────────────────────────────┐
        │                Spring Boot                 │
        │  auto-config · starters · embedded server  │
        │  Actuator · sensible defaults              │
        ├───────────────────────────────────────────┤
        │              Spring Framework              │
        │  IoC container · DI · AOP · Spring MVC     │
        │  transactions · data access                │
        └───────────────────────────────────────────┘
              Spring Boot builds ON TOP of Spring
```

**Key takeaway:** everything you know about the Spring Framework (dependency injection, beans, MVC) still applies. Spring Boot just wires it up for you.

---

## 1.3 Core concept: Inversion of Control & Dependency Injection

Spring is built around **Inversion of Control (IoC)**. Instead of your code creating its own dependencies, the Spring **container** creates and injects them. This is **Dependency Injection (DI)**.

```java
// Without DI — tightly coupled, hard to test
public class OrderService {
    private final EmailClient email = new EmailClient(); // created here, fixed
}

// With DI — Spring injects the dependency
@Service
public class OrderService {
    private final EmailClient email;

    // Spring provides an EmailClient bean via the constructor
    public OrderService(EmailClient email) {
        this.email = email;
    }
}
```

**Why this matters:** the `OrderService` no longer knows *how* to build an `EmailClient`. In tests you can inject a fake one. In production Spring injects the real one. Loose coupling makes code flexible and testable. We'll use constructor injection throughout this tutorial — it's the recommended style.

---

## 1.4 Advantages of Spring Boot

- **Rapid development** — go from idea to running API in minutes.
- **Less configuration** — auto-configuration handles the plumbing.
- **Standalone & portable** — one executable JAR runs anywhere with a JVM.
- **Production-ready** — Actuator adds health checks, metrics, and monitoring endpoints.
- **Huge ecosystem** — data, security, messaging, cloud, batch — all integrate cleanly.
- **Strong community & docs** — the most widely used Java backend framework.
- **Microservice-friendly** — small, independently deployable services fit naturally.

---

## 1.5 Real-world use cases

| Use case | Why Spring Boot fits |
|----------|----------------------|
| **REST APIs / microservices** | Fast setup, embedded server, easy JSON handling |
| **Enterprise web applications** | Mature, secure, integrates with everything |
| **Backend for mobile/web apps** | Clean API layer with auth and validation |
| **Batch & scheduled jobs** | Spring Batch + scheduling support |
| **Event-driven systems** | Kafka/RabbitMQ integration |
| **Cloud-native apps** | Works well on AWS, Azure, GCP, Kubernetes |

Companies like Netflix, Amazon, and many banks and startups run Spring Boot in production for exactly these reasons.

---

## 1.6 When *not* to reach for Spring Boot

Spring Boot is powerful but not always the lightest choice:

- For a tiny script or CLI tool, plain Java or a micro-framework may be simpler.
- For ultra-low-memory or fast-cold-start needs (some serverless cases), consider lighter frameworks like Quarkus or Micronaut, or Spring Boot with GraalVM native images.

For the vast majority of backend and API work, though, Spring Boot is an excellent default.

---

## Mini-project: sketch your architecture

No code yet — plan the capstone we'll build. We're going to create a **Task Management API**. On paper, sketch:

1. The **entities**: what is a `Task`? What is a `User`?
2. The **endpoints**: create a task, list tasks, update, delete, mark complete.
3. The **layers** each request passes through (controller → service → repository → DB).

Keep this sketch — you'll implement it piece by piece and assemble it in the capstone.

## Key takeaways

- Spring Boot = Spring Framework + auto-configuration + starters + embedded server.
- It reduces boilerplate so you focus on business logic.
- Dependency injection (via the IoC container) is the foundation; prefer constructor injection.
- Ideal for REST APIs, microservices, and enterprise backends.

Next → [2. Setup & Installation](02-setup-installation.md)
