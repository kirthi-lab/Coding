# Spring Boot Tutorial: Beginner to Advanced

A complete, hands-on guide to building production-grade applications with **Spring Boot 3.x** and **Java 17+**. Every section explains the *why*, shows runnable code, and ends with a mini-project. The tutorial finishes with a capstone that ties everything together.

## Who this is for

- Java developers who know the basics (see the [Java Tutorial](../Java-Tutorial/README.md)) and want to build web apps and APIs
- Backend developers moving to Spring Boot from another framework
- Anyone preparing for real-world Spring Boot work or interviews

## What you'll build

Along the way you'll build focused mini-projects, then a **capstone Task Management REST API** with authentication, a database, validation, tests, and a Docker deployment.

## Prerequisites

- Java fundamentals (classes, interfaces, generics, collections, lambdas/streams)
- Comfort with the command line
- JDK 17 or newer, and an IDE (IntelliJ IDEA, VS Code, or Eclipse)

## Table of Contents

| # | Section | What you'll learn |
|---|---------|-------------------|
| 1 | [Introduction to Spring Boot](01-introduction.md) | What it is, vs Spring Framework, advantages, use cases |
| 2 | [Setup & Installation](02-setup-installation.md) | Java, Maven/Gradle, Spring Initializr |
| 3 | [Basic Concepts](03-basic-concepts.md) | Project structure, configuration, `@SpringBootApplication`, running |
| 4 | [REST API Development](04-rest-api.md) | Controllers, services, repositories, HTTP, DTOs |
| 5 | [Data Access with Spring Data JPA](05-data-access-jpa.md) | JPA, MySQL/PostgreSQL, CRUD, pagination, sorting |
| 6 | [Validation & Exception Handling](06-validation-exception-handling.md) | Bean validation, custom exceptions, global handlers |
| 7 | [Security with JWT](07-security-jwt.md) | Spring Security, authentication, authorization, JWT |
| 8 | [Advanced Features](08-advanced-features.md) | Profiles, config properties, scheduling, async |
| 9 | [Testing](09-testing.md) | JUnit 5, Mockito, integration testing |
| 10 | [Deployment](10-deployment.md) | JAR/WAR, Tomcat, Docker, AWS/Azure/GCP |
| 11 | [Best Practices, Performance & Capstone](11-best-practices-capstone.md) | Organization, logging, monitoring, optimization, full capstone |

## The big picture: Spring Boot application architecture

A typical Spring Boot REST application is organized in layers. A request flows top to bottom; data flows back up.

```
                 HTTP request (JSON)
                        │
                        ▼
        ┌───────────────────────────────┐
        │   Controller  (@RestController)│  ← handles HTTP, maps URLs
        └───────────────┬───────────────┘
                        │ calls
                        ▼
        ┌───────────────────────────────┐
        │      Service  (@Service)       │  ← business logic, transactions
        └───────────────┬───────────────┘
                        │ calls
                        ▼
        ┌───────────────────────────────┐
        │   Repository  (@Repository)    │  ← data access (Spring Data JPA)
        └───────────────┬───────────────┘
                        │ SQL
                        ▼
        ┌───────────────────────────────┐
        │          Database              │
        └───────────────────────────────┘

  Cross-cutting: Security filter chain, Validation,
  Exception handling (@ControllerAdvice), Config/Profiles
```

**Why layers?** Each layer has one responsibility. Controllers speak HTTP, services hold business rules, repositories talk to the database. This separation makes the code testable, swappable, and easy to reason about.

## How to use this tutorial

Work through the sections in order — each builds on the last. Type the code yourself and run it. Every mini-project is self-contained and runnable.

## Versions used

| Tool | Version |
|------|---------|
| Java | 17+ (LTS) |
| Spring Boot | 3.x |
| Build tool | Maven (Gradle notes included) |
| Database | H2 (dev), MySQL / PostgreSQL (prod) |

---

Start here → [1. Introduction to Spring Boot](01-introduction.md)

