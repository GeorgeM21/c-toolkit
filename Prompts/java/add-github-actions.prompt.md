---
agent: 'Feature-Scaffolder'
description: 'Add CI/CD pipeline with GitHub Actions for Java/Spring Boot projects'
# Note: Uses Feature-Scaffolder agent if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Add GitHub Actions CI/CD Pipeline

## Task

Create GitHub Actions workflows for continuous integration and deployment of Java/Spring Boot applications, including build, test, security scanning, code quality, and deployment stages.

## Configuration

| Setting | Options | Default |
|---------|---------|---------|
| Java Version | 17, 21 | 21 |
| Build Tool | Maven, Gradle | Maven |
| Test Framework | JUnit 5 | JUnit 5 |
| Deploy Target | Azure Container Apps, AWS ECS, Docker Registry, None | Azure Container Apps |
| Include Security Scanning | Yes, No | Yes |
| Include Code Coverage | Yes, No | Yes |

## Workflow Files Structure

```
.github/
├── workflows/
│   ├── ci.yml                    # Main CI workflow
│   ├── cd.yml                    # Deployment workflow (optional)
│   └── codeql.yml                # Security scanning
├── dependabot.yml                # Dependency updates
└── CODEOWNERS                    # Code review assignments
```

## CI Workflow (ci.yml)

```yaml
name: CI

on:
  push:
    branches: [main, develop]
    paths-ignore: ['**.md', 'docs/**']
  pull_request:
    branches: [main, develop]
    paths-ignore: ['**.md', 'docs/**']

env:
  JAVA_VERSION: '21'
  JAVA_DISTRIBUTION: 'temurin'

jobs:
  build:
    name: Build and Test
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-java@v4
        with:
          distribution: ${{ env.JAVA_DISTRIBUTION }}
          java-version: ${{ env.JAVA_VERSION }}
          cache: maven

      - name: Build and run tests with coverage
        run: chmod +x mvnw && ./mvnw verify -B -Pcoverage

      - name: Upload JaCoCo coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: target/site/jacoco/jacoco.xml
          fail_ci_if_error: false

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: |
            target/surefire-reports/
            target/failsafe-reports/
          retention-days: 7

  security-scan:
    name: Security Scan (OWASP Dependency-Check)
    runs-on: ubuntu-latest
    needs: build
    permissions:
      security-events: write

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: ${{ env.JAVA_DISTRIBUTION }}
          java-version: ${{ env.JAVA_VERSION }}
          cache: maven

      - name: Run OWASP Dependency-Check
        run: chmod +x mvnw && ./mvnw dependency-check:check -B -DfailBuildOnCVSS=7 -Dformat=ALL
        continue-on-error: true

      - name: Upload Dependency-Check report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: dependency-check-report
          path: target/dependency-check-report.*
          retention-days: 14

  code-quality:
    name: Code Quality
    runs-on: ubuntu-latest
    needs: build

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: ${{ env.JAVA_DISTRIBUTION }}
          java-version: ${{ env.JAVA_VERSION }}
          cache: maven

      - name: Run Checkstyle
        run: chmod +x mvnw && ./mvnw checkstyle:check -B
        continue-on-error: true

      - name: Run SpotBugs
        run: ./mvnw spotbugs:check -B
        continue-on-error: true

  docker-build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [build, security-scan]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,prefix=
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

> **Gradle variant:** Change `cache: maven` to `cache: gradle`, replace `mvnw` with `gradlew`, use `./gradlew build jacocoTestReport --no-daemon`, and update coverage file path to `build/reports/jacoco/test/jacocoTestReport.xml`.

## CD Workflow (cd.yml)

```yaml
name: CD

on:
  workflow_run:
    workflows: [CI]
    types: [completed]
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' || github.event.inputs.environment == 'staging' }}
    environment:
      name: staging
      url: https://${PROJECT_NAME}-staging.azurecontainerapps.io

    steps:
      - uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Deploy to Azure Container Apps
        uses: azure/container-apps-deploy-action@v1
        with:
          resourceGroup: ${{ vars.AZURE_RESOURCE_GROUP }}-staging
          containerAppName: ${{ vars.AZURE_CONTAINER_APP }}
          imageToDeploy: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

      - name: Run smoke tests
        run: |
          sleep 30
          curl -f https://${PROJECT_NAME}-staging.azurecontainerapps.io/actuator/health || exit 1

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: ${{ github.event.inputs.environment == 'production' }}
    environment:
      name: production
      url: https://${PROJECT_NAME}.azurecontainerapps.io

    steps:
      - uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Deploy to Azure Container Apps
        uses: azure/container-apps-deploy-action@v1
        with:
          resourceGroup: ${{ vars.AZURE_RESOURCE_GROUP }}-prod
          containerAppName: ${{ vars.AZURE_CONTAINER_APP }}
          imageToDeploy: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

      - name: Run smoke tests
        run: |
          sleep 30
          curl -f https://${PROJECT_NAME}.azurecontainerapps.io/actuator/health || exit 1
```

> **AWS ECS variant:** Replace Azure steps with `aws-actions/configure-aws-credentials@v4`, `aws-actions/amazon-ecr-login@v2`, `aws-actions/amazon-ecs-render-task-definition@v1`, and `aws-actions/amazon-ecs-deploy-task-definition@v2`. Use `wait-for-service-stability: true` for health verification.

## CodeQL Security Scanning (codeql.yml)

```yaml
name: CodeQL

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '30 1 * * 0'

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      actions: read
      contents: read
    strategy:
      matrix:
        language: [java]

    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '21'
          cache: maven
      - run: chmod +x mvnw && ./mvnw package -DskipTests -B
      - uses: github/codeql-action/analyze@v3
```

## Dependabot Configuration

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "maven"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 10
    groups:
      spring-boot:
        patterns: ["org.springframework.boot:*", "org.springframework:*", "org.springframework.cloud:*"]
        update-types: ["minor", "patch"]
      testing:
        patterns: ["org.junit*", "org.mockito*", "org.assertj*"]
        update-types: ["minor", "patch"]
    ignore:
      - dependency-name: "*"
        update-types: ["version-update:semver-major"]

  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

## Maven Plugins

### JaCoCo (coverage profile)

Add under `<profiles>` — activated with `./mvnw verify -Pcoverage`:

```xml
<profile>
    <id>coverage</id>
    <build><plugins>
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>0.8.12</version>
            <executions>
                <execution><id>prepare-agent</id><goals><goal>prepare-agent</goal></goals></execution>
                <execution><id>report</id><phase>verify</phase><goals><goal>report</goal></goals></execution>
            </executions>
        </plugin>
    </plugins></build>
</profile>
```

### OWASP Dependency-Check, Checkstyle, SpotBugs

Add under `<build><plugins>`:

| Plugin | GAV | Key Config |
|--------|-----|------------|
| **OWASP Dependency-Check** | `org.owasp:dependency-check-maven:10.0.3` | `<failBuildOnCVSS>7</failBuildOnCVSS>`, formats: HTML + JSON |
| **Checkstyle** | `org.apache.maven.plugins:maven-checkstyle-plugin:3.4.0` | `<configLocation>google_checks.xml</configLocation>`, uses `com.puppycrawl.tools:checkstyle:10.18.1` |
| **SpotBugs** | `com.github.spotbugs:spotbugs-maven-plugin:4.8.6.4` | `<effort>Max</effort>`, `<threshold>Medium</threshold>` |

## Required Secrets & Variables

| Secret / Variable | Description |
|-------------------|-------------|
| `AZURE_CREDENTIALS` | Azure service principal JSON for deployments |
| `CODECOV_TOKEN` | Codecov token for coverage reports |
| `AZURE_RESOURCE_GROUP` | Azure resource group name (set per environment) |
| `AZURE_CONTAINER_APP` | Azure Container App name |

> For AWS ECS deployments, add: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `ECS_CLUSTER`, `ECS_SERVICE`, `ECS_TASK_DEFINITION`, `ECR_REPOSITORY`, `ECS_CONTAINER_NAME`.

## Best Practices

- Use caching for Maven/Gradle dependencies and Docker layers to speed up builds
- Fail fast on high/critical security vulnerabilities (CVSS >= 7)
- Use GitHub environments with approval gates for production deployments
- Pin action versions (e.g., `actions/checkout@v4`) for reproducibility
- Use `paths-ignore` to skip builds when only documentation changes
- Activate JaCoCo via Maven profile (`-Pcoverage`) to avoid overhead in local builds
- Never store secrets in workflow files — use GitHub Secrets and Variables
- Never deploy directly to production without staging validation
