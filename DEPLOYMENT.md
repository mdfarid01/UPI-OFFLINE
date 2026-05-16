# Deployment Guide

This project can be deployed as a Dockerized Spring Boot web service.

Recommended beginner-friendly option: **Render Web Service with Docker**.

The repository already includes:

- `Dockerfile`
- `.dockerignore`
- `render.yaml`
- `server.port=${PORT:8080}` in `application.properties`

That means the app runs locally on port `8080`, but on deployment platforms it can bind to the platform-provided `PORT`.

---

## Important Deployment Note

This project uses an H2 in-memory database:

```properties
spring.datasource.url=jdbc:h2:mem:upimesh
```

So on deployment:

- The app works.
- The dashboard works.
- The APIs work.
- Demo accounts are created on startup.
- Data resets when the service restarts.

That is fine for a portfolio/demo project. For production, replace H2 with PostgreSQL or MySQL.

---

## Deploy On Render

### Step 1: Push Code To GitHub

Make sure your project is pushed to GitHub.

```bash
git add -A
git commit -m "Prepare Spring Boot UPI mesh demo for deployment"
git push origin main
```

### Step 2: Create A Render Account

Go to:

```text
https://render.com
```

Sign in with GitHub.

### Step 3: Create New Web Service

In Render:

1. Click `New`.
2. Select `Web Service`.
3. Connect your GitHub repository.
4. Select this repo.

Render should detect the Dockerfile.

Use:

```text
Environment: Docker
```

If Render asks for a branch, choose:

```text
main
```

### Step 4: Service Settings

Suggested settings:

```text
Name: upi-offline-mesh
Runtime: Docker
Plan: Free
Branch: main
```

Health check path:

```text
/api/mesh/state
```

### Step 5: Deploy

Click:

```text
Create Web Service
```

Render will:

1. Pull your GitHub repo.
2. Build the Docker image.
3. Run Maven inside Docker.
4. Create the Spring Boot JAR.
5. Start the app.
6. Expose a public URL.

Your final URL will look like:

```text
https://upi-offline-mesh.onrender.com
```

Open that URL to use the dashboard.

---

## Why `PORT` Matters

Locally, the app can run on:

```text
8080
```

But deployment platforms often assign a dynamic port through an environment variable named:

```text
PORT
```

So this project uses:

```properties
server.port=${PORT:8080}
```

Meaning:

- If `PORT` exists, use it.
- If `PORT` does not exist, use `8080`.

This keeps the same code working locally and in the cloud.

---

## Dockerfile Explanation

The Dockerfile uses two stages.

### Stage 1: Build

```dockerfile
FROM eclipse-temurin:25-jdk AS build
```

This image has Java 25 JDK, which is needed to compile the project.

It copies Maven wrapper files:

```dockerfile
COPY .mvn .mvn
COPY mvnw pom.xml ./
```

Then it downloads dependencies:

```dockerfile
RUN ./mvnw -B -DskipTests dependency:go-offline
```

Then it copies source code and builds the JAR:

```dockerfile
COPY src src
RUN ./mvnw -B -DskipTests package
```

### Stage 2: Runtime

```dockerfile
FROM eclipse-temurin:25-jre
```

This image has only the Java runtime, not the full compiler, so it is smaller.

It copies the built JAR:

```dockerfile
COPY --from=build /workspace/target/upi-offline-mesh-0.0.1-SNAPSHOT.jar app.jar
```

It starts the app:

```dockerfile
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

---

## Test Deployment Locally With Docker

If Docker is installed:

```bash
docker build -t upi-offline-mesh .
```

Run:

```bash
docker run --rm -p 8080:8080 upi-offline-mesh
```

Open:

```text
http://localhost:8080
```

To simulate a cloud platform port:

```bash
docker run --rm -e PORT=9090 -p 9090:9090 upi-offline-mesh
```

Open:

```text
http://localhost:9090
```

---

## Useful URLs After Deployment

Replace `<your-domain>` with your Render URL.

```text
https://<your-domain>/
https://<your-domain>/api/accounts
https://<your-domain>/api/transactions
https://<your-domain>/api/mesh/state
https://<your-domain>/api/server-key
```

H2 console may also be available:

```text
https://<your-domain>/h2-console
```

For a public demo, avoid sharing the H2 console link widely. In a production system it should be disabled.

---

## Common Deploy Problems

### Build Fails Because Java Version Is Wrong

This project is configured for Java 25:

```xml
<java.version>25</java.version>
```

The Dockerfile uses Java 25 images, so Docker deployment avoids most platform Java-version mismatch problems.

### App Starts But Render Says Port Not Detected

Confirm this exists:

```properties
server.port=${PORT:8080}
```

The app must bind to the platform-provided `PORT`.

### Data Disappears After Restart

Expected. H2 is in-memory.

For permanent data, add PostgreSQL and change:

```properties
spring.datasource.url=...
spring.datasource.username=...
spring.datasource.password=...
spring.jpa.hibernate.ddl-auto=update
```

### First Deployment Is Slow

Expected. Docker needs to download Java, Maven dependencies, and build the JAR.

---

## Deployment Checklist

- [ ] Code pushed to GitHub
- [ ] `Dockerfile` present
- [ ] `.dockerignore` present
- [ ] `server.port=${PORT:8080}` configured
- [ ] Render service uses Docker
- [ ] Health check path set to `/api/mesh/state`
- [ ] Dashboard opens
- [ ] `/api/accounts` returns demo accounts
- [ ] Inject -> Gossip -> Bridge flow works

