# 2. Setup & Installation

[← Back](01-introduction.md) | [Index](README.md) | Next → [3. Basic Concepts](03-basic-concepts.md)

This section gets your machine ready and generates your first Spring Boot project.

---

## 2.1 Install Java 17+

Spring Boot 3.x **requires Java 17 or newer**.

1. Download an LTS JDK: [Eclipse Temurin 17/21](https://adoptium.net/) or [Oracle JDK](https://www.oracle.com/java/technologies/downloads/).
2. Install it and verify:

```bash
java -version
```

Expected output (version may differ):

```
openjdk version "21.0.2" 2024-01-16 LTS
```

If the command isn't found, add the JDK's `bin` directory to your `PATH` environment variable and reopen the terminal.

---

## 2.2 Install a build tool

Spring Boot projects use **Maven** or **Gradle** to manage dependencies and build the app. This tutorial uses **Maven** (the most common), with Gradle notes where useful.

### Option A: Maven

Download from [maven.apache.org](https://maven.apache.org/download.cgi), unzip, add its `bin` to `PATH`, then verify:

```bash
mvn -version
```

> **Tip:** You don't strictly need to install Maven globally. Spring Initializr projects include a **Maven Wrapper** (`mvnw` / `mvnw.cmd`) that downloads the right Maven version automatically. Use `./mvnw` (macOS/Linux) or `.\mvnw.cmd` (Windows).

### Option B: Gradle

Download from [gradle.org](https://gradle.org/install/) or use the included Gradle Wrapper (`gradlew`). Verify:

```bash
gradle -version
```

---

## 2.3 Choose an IDE

Any of these work well:

- **IntelliJ IDEA** (Community is free) — best Spring support.
- **VS Code** with the "Extension Pack for Java" and "Spring Boot Extension Pack".
- **Eclipse** with Spring Tools 4.

---

## 2.4 Create a project with Spring Initializr

[Spring Initializr](https://start.spring.io) generates a ready-to-run project. You can use the website or the IDE integration.

### Using the website

1. Go to **https://start.spring.io**.
2. Set the options:

| Field | Value |
|-------|-------|
| Project | **Maven** |
| Language | **Java** |
| Spring Boot | **3.x** (latest stable) |
| Group | `com.example` |
| Artifact | `demo` |
| Packaging | **Jar** |
| Java | **17** (or 21) |

3. Add **Dependencies** (click "Add Dependencies"):
   - **Spring Web** — build REST APIs
   - **Spring Boot DevTools** — auto-restart during development
4. Click **Generate**. A `demo.zip` downloads.
5. Unzip it into your workspace and open it in your IDE.

```
   start.spring.io  ──►  demo.zip  ──►  unzip  ──►  open in IDE  ──►  run
   (pick options)       (download)               (ready project)
```

### Using the command line (curl)

You can generate the same project without the browser:

```bash
curl https://start.spring.io/starter.zip \
  -d dependencies=web,devtools \
  -d javaVersion=17 \
  -d bootVersion=3.3.0 \
  -d groupId=com.example \
  -d artifactId=demo \
  -d type=maven-project \
  -o demo.zip
```

Then unzip and open it.

---

## 2.5 Understand the generated `pom.xml`

The `pom.xml` is Maven's project file. Key parts:

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>

<properties>
    <java.version>17</java.version>
</properties>

<dependencies>
    <!-- Web starter: REST, embedded Tomcat, JSON -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Test starter: JUnit 5, Mockito, Spring Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

- The **parent** manages versions so you rarely specify them yourself.
- **Starters** are the `spring-boot-starter-*` dependencies.
- Notice the web and test dependencies have no `<version>` — the parent decides it.

---

## 2.6 Run the app

From the project root:

```bash
# Maven
./mvnw spring-boot:run          # macOS/Linux
.\mvnw.cmd spring-boot:run      # Windows PowerShell

# Gradle
./gradlew bootRun
```

You should see the Spring Boot banner and a line like:

```
Tomcat started on port(s): 8080 (http)
Started DemoApplication in 1.234 seconds
```

The app is now running at **http://localhost:8080**. There are no endpoints yet, so you'll get a 404 (that's expected). We'll add endpoints in the next sections.

> On Windows, remember to use `.\mvnw.cmd` and PowerShell's `;` to chain commands, not `&&`.

---

## Mini-project: generate and run your first app

1. Create a project on Spring Initializr with **Spring Web** and **DevTools**.
2. Open it in your IDE.
3. Run it and confirm you see "Tomcat started on port(s): 8080".
4. Open http://localhost:8080 in a browser (a 404 error page confirms the server is up).

If you got the server running, your environment is ready.

## Key takeaways

- Spring Boot 3.x needs Java 17+.
- Use Spring Initializr to scaffold projects; add starters for the features you need.
- The Maven/Gradle wrapper means you don't need a global build tool install.
- Run with `spring-boot:run` (Maven) or `bootRun` (Gradle); the embedded server starts on port 8080.

Next → [3. Basic Concepts](03-basic-concepts.md)
