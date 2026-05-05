---
agent: 'Feature-Scaffolder'
description: 'Add optimized multi-stage Dockerfile for Java/Spring Boot applications'
# Note: Uses Feature-Scaffolder agent if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Add Dockerfile for Java/Spring Boot Application

## Task

Create a production-ready, multi-stage Dockerfile and supporting Docker configuration files for Java/Spring Boot applications, following best practices for size, security, performance, and portability.

Before generating files, inspect the repository and derive the implementation from the existing project instead of applying template defaults blindly.

Required discovery steps:
- Detect the build tool from the repository (`pom.xml` / `build.gradle` / wrapper files).
- Detect the Java version from the project configuration and use a matching base image unless the user explicitly overrides it.
- Detect the Spring Boot major version and choose the correct layered-jar extraction mode.
- Inspect application configuration (`application.properties`, profiles, actuator settings) before inventing Spring profiles, datasource variables, or health check paths.
- Detect whether Docker assets already exist and preserve or extend them instead of replacing them blindly.
- **Ask the user for the deployment target** before generating files (see Configuration).

Output requirements:
- Prefer minimal, working assets over generic scaffolding.
- Do not hardcode a production profile in the image unless the repository clearly requires it.
- If compose files are generated, wire them to the app's actual configuration contract, not assumed environment variable names.
- Do not emit local machine absolute filesystem links in generated Markdown.

## Configuration

| Setting | Options | Default |
|---------|---------|---------|
| **Deployment Target** | kubernetes, docker-compose | **Ask the user** |
| Java Version | 17, 21 | 21 |
| Build Tool | Maven, Gradle | Maven |
| Base Image | eclipse-temurin, amazoncorretto, bellsoft-liberica, distroless | eclipse-temurin |
| Port | Custom port | 8080 |
| Include Health Check | Yes, No | Yes (compose), No (kubernetes) |
| Include docker-compose | Yes, No | Yes (compose), No (kubernetes) |

Configuration rules:
- Treat the table defaults as fallbacks only. Repository configuration wins.
- If the repository targets Java 17, do not emit Java 21 images unless explicitly asked.
- If the app exposes Actuator health, use it for health checks. If not, either omit the health check or document the chosen endpoint.
- Only create a `dev` or `docker` Spring profile if the repository already has one or the task explicitly asks you to create it.
- Validate that the chosen base image tag exists for the current target architecture before finalizing. If it does not, fall back within the same image family and document the fallback.

### Deployment target rules

The deployment target changes how the Dockerfile and supporting files are generated:

**Kubernetes-native:**
- Do NOT include a `HEALTHCHECK` directive in the Dockerfile — Kubernetes uses its own liveness/readiness probes.
- Do NOT install `curl` or any other tools in the runtime image — keep the attack surface minimal.
- Use an empty `ENV JAVA_OPTS=""` — the orchestrator (K8s manifests, Helm values) controls JVM tuning at deploy time.
- Use a shell entrypoint with `exec` to allow `JAVA_OPTS` expansion: `ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]`. The `exec` is critical — without it, `sh` is PID 1, `SIGTERM` never reaches Java, and graceful shutdown is broken.
- **Security note:** The `sh -c` entrypoint expands `$JAVA_OPTS` via the shell. In Kubernetes, `JAVA_OPTS` is set from pod spec env vars, which are controlled by cluster operators. Document that this variable must only be set by trusted sources (K8s manifests, Helm values, sealed secrets) — never from user-facing input.
- Add a Dockerfile comment or README section documenting the recommended K8s probe endpoint (e.g., `/actuator/health` if Actuator is present) and recommended JVM memory flags (e.g., `-XX:MaxRAMPercentage=75.0`) so operators know what to configure in their pod specs.
- docker-compose files are optional (for local development convenience only).

**Docker Compose-native:**
- Include a `HEALTHCHECK` directive using the actuator health endpoint.
- Install `curl` in the runtime image (required for the health check).
- Set `ENV JAVA_TOOL_OPTIONS` with sensible container-aware defaults in the Dockerfile.
- Use exec-form entrypoint: `ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]` — Java is PID 1 and receives signals directly.
- Always generate `docker-compose.yml` with restart policies and resource limits.

## Dockerfile Template

The template below uses BuildKit features for optimal caching. The `# syntax` directive is required.

```dockerfile
# syntax=docker/dockerfile:1.7

FROM eclipse-temurin:21-jdk-noble AS build
WORKDIR /workspace

COPY .mvn/ .mvn/
COPY mvnw pom.xml ./

RUN --mount=type=cache,target=/root/.m2 \
    chmod +x mvnw && ./mvnw -B -q -DskipTests dependency:go-offline

COPY src/ src/

RUN --mount=type=cache,target=/root/.m2 \
    ./mvnw -B -q -DskipTests package && \
    java -Djarmode=tools -jar target/*.jar extract --layers --launcher --destination target/extracted

# --- Runtime stage ---
FROM eclipse-temurin:21-jre-noble AS runtime
WORKDIR /app

RUN groupadd --system spring && useradd --system --gid spring spring

COPY --link --from=build --chown=spring:spring /workspace/target/extracted/dependencies/ ./
COPY --link --from=build --chown=spring:spring /workspace/target/extracted/spring-boot-loader/ ./
COPY --link --from=build --chown=spring:spring /workspace/target/extracted/snapshot-dependencies/ ./
COPY --link --from=build --chown=spring:spring /workspace/target/extracted/application/ ./

USER spring:spring

EXPOSE 8080
```

After `EXPOSE 8080`, append the deployment-target-specific block:

**If Kubernetes-native**, append:
```dockerfile
ENV JAVA_OPTS=""

ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]
```

**If Docker Compose-native**, append:
```dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

ENV JAVA_TOOL_OPTIONS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -XX:InitialRAMPercentage=50.0"

HEALTHCHECK --interval=30s --timeout=3s --start-period=30s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

> **Important:** When using compose-native, the `apt-get` and `HEALTHCHECK` lines must come **before** the `USER` directive, or move `apt-get` before `USER` and keep `HEALTHCHECK` after. The template above places `curl` installation in the runtime stage before the user switch — adjust ordering so `apt-get` runs as root.

Notes:
- For Spring Boot 4+, prefer `-Djarmode=tools ... extract --layers --launcher --destination extracted`.
- For older Spring Boot versions that do not support `jarmode=tools`, fall back to `-Djarmode=layertools -jar app.jar extract` and adjust copy paths accordingly.
- Always use `--chown` on `COPY --from` directives so the non-root user owns the application files.
- Use `COPY --link` on runtime-stage `COPY --from` directives. The `--link` flag creates each layer independently of previous layers, improving cache reuse when only the base image changes. Requires BuildKit.
- The `# syntax=docker/dockerfile:1.7` directive enables BuildKit features like `--mount=type=cache` and `COPY --link`. Without it, mount directives silently fail on older Docker versions.
- Use `-q` (quiet) flag on Maven/Gradle builds to reduce log noise in CI.
- If the project produces multiple jars (e.g., sources, javadoc), configure Maven `finalName` in `pom.xml` for a predictable jar name instead of relying on glob patterns.
- When using the Maven wrapper together with cache mounts, keep the wrapper distribution path separate from any mounted repository cache if execute permissions or ownership would otherwise break wrapper startup.

> **Gradle variant:** Replace Stage 1 only. Copy `gradle/`, `gradlew`, `build.gradle`, `settings.gradle` instead of `.mvn/`, `mvnw`, `pom.xml`. Use `./gradlew bootJar --no-daemon -x test` for the build command, and normalize the produced jar to a stable name such as `build/libs/app.jar`.

## Base Image Options

| Image | Build Stage | Runtime Stage | Size | Notes |
|-------|-----------|--------------|------|-------|
| **Eclipse Temurin** (default) | `eclipse-temurin:21-jdk-noble` | `eclipse-temurin:21-jre-noble` | ~220MB | Best compatibility, Adoptium-backed |
| **Amazon Corretto** | `amazoncorretto:21` | `amazoncorretto:21-alpine` | ~200MB | Optimized for AWS workloads |
| **BellSoft Liberica** | `bellsoft/liberica-openjdk-alpine:21` | `bellsoft/liberica-openjre-alpine:21` | ~150MB | Smallest full JRE; musl libc may cause native library issues |
| **Distroless** | Use Temurin for build | `gcr.io/distroless/java21-debian12` | ~200MB | No shell/package manager; must use `java -jar` entry point; health checks via orchestrator only; forces kubernetes-native mode |

> **Base image freshness:** Prefer the latest Ubuntu LTS as the base suffix (e.g., `noble` for 24.04 over `jammy` for 22.04) to get current security patches. Before finalizing, verify the tag exists on Docker Hub for the target architecture. If it does not, fall back to the previous LTS and document the reason.

## docker-compose.yml

Generate only when the deployment target is **docker-compose** or the user explicitly requests it.

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: ${PROJECT_NAME}-app
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=postgres
      - POSTGRES_URL=jdbc:postgresql://db:5432/${PROJECT_NAME}
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASS=${DB_PASSWORD}
      - JAVA_TOOL_OPTIONS=-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -XX:InitialRAMPercentage=50.0
    depends_on:
      db:
        condition: service_healthy
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1.0'
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - app-network

  db:
    image: postgres:16-alpine
    container_name: ${PROJECT_NAME}-db
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=${PROJECT_NAME}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${PROJECT_NAME}"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - app-network

# For MySQL: replace the db service image with mysql:8.0 and update SPRING_DATASOURCE_URL accordingly.

volumes:
  postgres-data:

networks:
  app-network:
    driver: bridge
```

Compose rules:
- Use the app's real environment contract. If the repository expects `SPRING_DATASOURCE_URL`, use that. If it expects custom variables like `POSTGRES_URL`, use those instead.
- Do not expose the database port to the host unless the task explicitly asks for it.
- Prefer localhost-only bindings for debug-only ports in development compose files.
- Do not assume a `docker` profile exists.
- Always include `restart: unless-stopped` on production services so containers recover from crashes and host reboots.
- Always include `deploy.resources.limits` with sensible memory/CPU defaults for the app service. Adjust based on the application's actual requirements. Omit from dev compose files.
- Check the PostgreSQL Docker image version for the correct volume mount path. PostgreSQL 18+ uses `/var/lib/postgresql` (PGDATA moved to `/var/lib/postgresql/<version>/docker`). PostgreSQL 17 and below use `/var/lib/postgresql/data`.

## .dockerignore

Generate a `.dockerignore` that excludes: build output (`target/`, `build/`, `out/`, `*.jar`, `*.war` — except `.mvn/wrapper/*.jar`), IDE files (`.idea/`, `.vscode/`, `*.iml`), `.git/`, `.github/`, Docker files (`Dockerfile*`, `.dockerignore`, `docker-compose*.yml`), documentation (`*.md`, `LICENSE`, `docs/`), test reports (`surefire-reports/`, `jacoco/`, `coverage/`), logs (`*.log`, `logs/`), and environment files (`.env`, `.env.*`).

Do not over-trim the build context if the selected build tool needs additional files such as `settings.gradle`, `gradle.properties`, Maven wrapper jars, or generated resource inputs.

> **Kaniko / remote builder caveat:** The `.dockerignore` pattern `Dockerfile*` excludes Dockerfiles from the build context. This is fine for standard `docker build`, but some CI builders (Kaniko, certain BuildKit frontends) read the Dockerfile from the context. If the project uses such a builder, remove the `Dockerfile*` exclusion or use a more targeted pattern.

## Development Dockerfile

Create a `Dockerfile.dev` that differs from production as follows:

```dockerfile
# syntax=docker/dockerfile:1.7
# Dockerfile.dev — single stage, JDK for local development with optional remote debugging
FROM eclipse-temurin:21-jdk-noble

WORKDIR /app

RUN apt-get update \
    && apt-get install -y --no-install-recommends curl git \
    && rm -rf /var/lib/apt/lists/*

RUN groupadd --system spring && useradd --system --gid spring --create-home --home-dir /home/spring spring

ENV MAVEN_CONFIG=/home/spring/.m2
ENV MAVEN_USER_HOME=/home/spring/.m2

# BuildKit mount cache for Maven — use spring user's home so the
# prefetched repository is accessible at runtime when running as spring.
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN --mount=type=cache,target=/home/spring/.m2/repository \
    chmod +x mvnw && ./mvnw -B -q -Dmaven.repo.local=/home/spring/.m2/repository dependency:go-offline
COPY src/ src/

RUN chown -R spring:spring /app /home/spring

USER spring:spring

EXPOSE 8080 5005

ENV SPRING_BOOT_RUN_JVM_ARGUMENTS=""

ENTRYPOINT ["sh", "-c", "exec ./mvnw -Dmaven.repo.local=/home/spring/.m2/repository spring-boot:run -Dspring-boot.run.jvmArguments=\"$SPRING_BOOT_RUN_JVM_ARGUMENTS\""]
```

Pair with a `docker-compose.dev.yml` that mounts `./src` and `./pom.xml` as volumes, adds a `maven-cache` named volume at `/home/spring/.m2` (matching the dev image's `MAVEN_USER_HOME`), exposes ports `8080`, `5005` (debug), and `35729` (LiveReload), and sets `SPRING_DEVTOOLS_RESTART_ENABLED=true`.

Development rules:
- If no `dev` profile exists in the repository, do not force one. Use the app's default config or create the profile only if the task explicitly requests it.
- Be explicit that mounting source code alone does not guarantee Java recompilation; if the workflow relies on DevTools restarts, document the limitation or add the missing compile/watch behavior.
- Bind debug ports to `127.0.0.1` unless broader access is explicitly requested.
- Do not include `restart` policies or resource limits in dev compose files.
- Run the dev container as a non-root user unless a required toolchain step makes that impossible and the reason is documented.
- Do not enable JDWP by default. Remote debugging must be opt-in via environment variables or an explicit dev-only command.
- Install only OS packages required by the documented dev workflow or container entrypoint. Omit optional tools by default.
- If dependency prefetching is added to the dev image, ensure the runtime command uses the same warmed repository path.
- Do not describe the dev image as hot-reload, editable, or live-sync unless bind mounts and recompilation behavior are actually configured and documented.

## GitHub Actions Integration

For Docker build and push workflows in CI/CD, see the dedicated prompt: **`/add-github-actions`**.

## Best Practices

- Use multi-stage builds: JDK for build, JRE for runtime — never ship the JDK in production
- Use BuildKit `--mount=type=cache` for Maven/Gradle caches — this is significantly faster than `dependency:go-offline` because the cache persists across builds even when layers are invalidated
- Declare `# syntax=docker/dockerfile:1.7` at the top of every Dockerfile to enable BuildKit features
- Use Spring Boot layered JARs with a version-appropriate extraction command for optimal Docker layer caching
- Copy dependency descriptors (`pom.xml` / `build.gradle`) before source code so dependency layers cache independently
- Always run as non-root user (`USER spring`) in production containers
- Use `--chown` on `COPY --from` directives so application files are owned by the non-root user
- Use `COPY --link` on runtime-stage `COPY --from` directives for better layer cache reuse
- Use specific image tags (e.g., `eclipse-temurin:21-jre-noble`), never `latest`; prefer the latest Ubuntu LTS suffix
- Set JVM container-aware flags (`-XX:+UseContainerSupport`, `-XX:MaxRAMPercentage`)
- Never hardcode secrets in Dockerfile — use environment variables or secret management
- Always use exec-form entrypoint (`ENTRYPOINT ["java", ...]`) when no env var expansion is needed. If env var expansion is needed (e.g., `$JAVA_OPTS`), use `sh -c` with `exec` so Java replaces the shell and becomes PID 1 for proper signal handling
- Do not install unnecessary tools (curl, wget) in production images unless required for health checks
- In development images, avoid installing OS packages that are not required by the container entrypoint or explicitly documented workflow
- Always set `restart: unless-stopped` on production compose services
- Always set `deploy.resources.limits` (memory/CPU) on production compose services to prevent runaway processes from exhausting host resources
- Do not duplicate JVM config in both the Dockerfile `ENV` and compose `environment` — pick one location to avoid maintenance drift
- Do not enable remote debugging by default in generated development assets
- Do not set environment variables (`ENV`) that are not consumed by subsequent instructions or the runtime entrypoint — unused env vars add confusion and bloat the image config
- When the Dockerfile switches to a non-root user, ensure any cache or repository paths used during prefetch (build-time) are also accessible at runtime. Mismatched paths (e.g., prefetching into `/root/.m2` but running as `spring`) silently negate the cache

## Validation

After generating the files, validate them when tooling is available:
- Run the project's package step with tests skipped if needed.
- Run `docker compose config` for each generated compose file.
- Run `docker build` for the production image.
- If `Dockerfile.dev` is generated, run `docker build -f Dockerfile.dev` as well.
- If the app and database stack can be started locally, verify the health endpoint or landing page responds.
- If the dev image prefetches dependencies, verify the runtime command uses the prefetched repository rather than re-resolving into a different location.

If validation reveals project-specific incompatibilities, update the generated files instead of leaving the prompt output in a generic state.
