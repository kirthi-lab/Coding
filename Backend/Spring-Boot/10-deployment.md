# 10. Deployment

[← Back](09-testing.md) | [Index](README.md) | Next → [11. Best Practices & Capstone](11-best-practices-capstone.md)

Your app works locally — now ship it. This section covers packaging, running on a server, containerizing with Docker, and deploying to the cloud.

---

## 10.1 Package as an executable JAR

Spring Boot's default (and recommended) artifact is a self-contained "fat JAR" that includes your code, dependencies, and an embedded server.

```bash
./mvnw clean package        # Maven → target/demo-0.0.1-SNAPSHOT.jar
# or
./gradlew clean bootJar     # Gradle → build/libs/demo-0.0.1-SNAPSHOT.jar
```

Run it anywhere with a JVM:

```bash
java -jar target/demo-0.0.1-SNAPSHOT.jar
```

Pass configuration at runtime:

```bash
java -jar app.jar --spring.profiles.active=prod --server.port=9000
# or with environment variables
SERVER_PORT=9000 SPRING_PROFILES_ACTIVE=prod java -jar app.jar
```

> **Skip tests during packaging** (only if you've already run them): `./mvnw clean package -DskipTests`.

---

## 10.2 WAR packaging for an external Tomcat

Most deployments use the JAR. But if you must deploy to a **shared external Tomcat/JBoss**, build a WAR instead.

1. In `pom.xml`, change packaging:
   ```xml
   <packaging>war</packaging>
   ```
2. Make the embedded server "provided" (the external server supplies it):
   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-tomcat</artifactId>
       <scope>provided</scope>
   </dependency>
   ```
3. Extend `SpringBootServletInitializer` so the external container can launch the app:
   ```java
   @SpringBootApplication
   public class DemoApplication extends SpringBootServletInitializer {
       @Override
       protected SpringApplicationBuilder configure(SpringApplicationBuilder app) {
           return app.sources(DemoApplication.class);
       }
       public static void main(String[] args) {
           SpringApplication.run(DemoApplication.class, args);
       }
   }
   ```
4. Build with `./mvnw clean package` → produces a `.war`. Drop it in Tomcat's `webapps/`.

**Recommendation:** prefer the JAR + embedded server. It's simpler, more portable, and the standard for cloud and container deployments.

---

## 10.3 Containerize with Docker

Docker packages your app and its runtime into a portable image that runs identically everywhere.

### Write a Dockerfile

Use a multi-stage build: one stage compiles, a smaller stage runs. This keeps the final image lean.

```dockerfile
# ---- Build stage ----
FROM eclipse-temurin:17-jdk-alpine AS build
WORKDIR /app
COPY . .
RUN ./mvnw clean package -DskipTests

# ---- Run stage ----
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Build and run

```bash
docker build -t demo-app:1.0 .
docker run -p 8080:8080 -e SPRING_PROFILES_ACTIVE=prod demo-app:1.0
```

> Spring Boot can also build an optimized image without a Dockerfile using buildpacks: `./mvnw spring-boot:build-image`.

### Multi-container setup with Docker Compose

**Use case: run the app plus a PostgreSQL database together.**

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"

  app:
    build: .
    depends_on:
      - db
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/appdb
      SPRING_DATASOURCE_USERNAME: appuser
      SPRING_DATASOURCE_PASSWORD: secret
      SPRING_PROFILES_ACTIVE: prod
    ports:
      - "8080:8080"
```

```bash
docker compose up --build
```

Note how Spring Boot maps environment variables to properties: `SPRING_DATASOURCE_URL` → `spring.datasource.url`. This "relaxed binding" is how you configure containers cleanly without editing files.

---

## 10.4 Deploy to the cloud

The same JAR or Docker image runs on all major clouds. Here's the lay of the land.

```
   Your artifact (JAR or Docker image)
            │
   ┌────────┼────────────┬───────────────┐
   ▼        ▼            ▼               ▼
  AWS      Azure         GCP        Any Kubernetes
```

### AWS

| Option | Best for |
|--------|----------|
| **Elastic Beanstalk** | Easiest — upload the JAR, AWS handles servers/scaling |
| **ECS / Fargate** | Run the Docker image without managing servers |
| **EKS** | Kubernetes at scale |
| **Elastic Container Registry (ECR)** | Store your Docker images |

Quick path with Beanstalk: create a Java or Docker environment, upload the JAR/image, set env vars (DB URL, secrets) in the console.

### Azure

| Option | Best for |
|--------|----------|
| **Azure App Service** | Deploy JAR or container directly |
| **Azure Container Apps** | Serverless containers |
| **AKS** | Managed Kubernetes |

The Azure CLI or the Maven plugin (`azure-webapp-maven-plugin`) can deploy in one command.

### GCP

| Option | Best for |
|--------|----------|
| **Cloud Run** | Deploy a container; scales to zero — great value |
| **App Engine** | Managed platform for the JAR |
| **GKE** | Managed Kubernetes |

Cloud Run example: `gcloud run deploy --source .` builds and deploys straight from your project.

### Universal principles for cloud

- **Externalize config** via environment variables — never bake secrets into the image.
- **Use a managed database** (RDS, Azure Database, Cloud SQL) rather than a container for production data.
- **Expose health endpoints** (Actuator, next section) so the platform can check liveness/readiness.
- **Store secrets** in a secrets manager (AWS Secrets Manager, Azure Key Vault, GCP Secret Manager), not in files.
- **Set the active profile** to `prod` via `SPRING_PROFILES_ACTIVE`.

> Deploying to production is a high-impact action. Double-check that the target environment, database, and secrets are the intended ones before you deploy, and prefer a staging environment first.

---

## Mini-project: dockerize and run with a database

1. Add the multi-stage `Dockerfile` above to your Task API project.
2. Add a `docker-compose.yml` with the app + PostgreSQL.
3. Configure the app via environment variables (no hard-coded DB credentials).
4. Run `docker compose up --build` and hit `http://localhost:8080/api/tasks`.
5. Stop, restart, and confirm data persists (add a named volume for `db` to make it durable).

## Key takeaways

- The default deployable is a self-contained executable JAR — run it with `java -jar`.
- Use WAR only when a shared external servlet container is mandated.
- Docker (ideally a multi-stage build) makes the app portable; Compose wires it to a database.
- Configure containers and cloud with environment variables via relaxed binding.
- All major clouds run the same JAR/image; externalize config, use managed databases, and store secrets securely.

Next → [11. Best Practices, Performance & Capstone](11-best-practices-capstone.md)
