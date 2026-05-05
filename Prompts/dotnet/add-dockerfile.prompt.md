---
agent: 'Feature-Scaffolder'
description: 'Add optimized multi-stage Dockerfile for .NET applications'
# Note: Uses Feature-Scaffolder agent if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Add Dockerfile for .NET Application

## Task

Create a multi-stage Dockerfile and supporting Docker configuration files for .NET applications, following best practices for size, security, and performance.

## Configuration

| Setting | Options | Default |
|---------|---------|---------|
| .NET Version | 8.0, 9.0 | 9.0 |
| Project Type | Web API, Blazor Server, Worker Service, Console | Web API |
| Base Image | debian (bookworm), alpine, ubuntu (noble), chiseled, azurelinux | debian |
| Port | Custom port | 8080 |
| Include Health Check | Yes, No | Yes |
| Include docker-compose | Yes, No | Yes |

## Dockerfile Template

```dockerfile
# ============================================================
# Stage 1: Build
# ============================================================
FROM mcr.microsoft.com/dotnet/sdk:9.0-bookworm-slim AS build

WORKDIR /src

# Copy project files first (for better layer caching)
COPY ["src/${PROJECT_NAME}/${PROJECT_NAME}.csproj", "src/${PROJECT_NAME}/"]
COPY ["src/${PROJECT_NAME}.Core/${PROJECT_NAME}.Core.csproj", "src/${PROJECT_NAME}.Core/"]
COPY ["src/${PROJECT_NAME}.Application/${PROJECT_NAME}.Application.csproj", "src/${PROJECT_NAME}.Application/"]
COPY ["src/${PROJECT_NAME}.Infrastructure/${PROJECT_NAME}.Infrastructure.csproj", "src/${PROJECT_NAME}.Infrastructure/"]

# Copy Directory.Build.props and Directory.Packages.props if they exist
COPY ["Directory.Build.props", "./"]
COPY ["Directory.Packages.props", "./"]

# Restore packages
RUN dotnet restore "src/${PROJECT_NAME}/${PROJECT_NAME}.csproj"

# Copy all source code
COPY . .

# Build and publish
WORKDIR "/src/src/${PROJECT_NAME}"
RUN dotnet publish "${PROJECT_NAME}.csproj" \
    -c Release \
    -o /app/publish \
    --no-restore \
    /p:UseAppHost=false \
    /p:PublishTrimmed=false

# ============================================================
# Stage 2: Runtime
# ============================================================
FROM mcr.microsoft.com/dotnet/aspnet:9.0-bookworm-slim AS runtime

# Install curl for health checks (remove if using chiseled images)
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy published application
COPY --from=build /app/publish .

# Create non-root user (already exists in base image as $APP_UID)
USER $APP_UID

# Configure environment
ENV ASPNETCORE_URLS=http://+:8080
ENV ASPNETCORE_ENVIRONMENT=Production
ENV DOTNET_RUNNING_IN_CONTAINER=true
ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# Entry point
ENTRYPOINT ["dotnet", "${PROJECT_NAME}.dll"]
```

## Base Image Options

### Debian (Default - Best Compatibility)
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0-bookworm-slim AS runtime
# Size: ~220MB, Full glibc support, most compatible
```

### Alpine (Smallest Size)
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0-alpine AS runtime
# Size: ~110MB, Uses musl libc
# Note: May have compatibility issues with some native libraries

# For Alpine, use apk instead of apt-get
RUN apk add --no-cache curl
```

### Ubuntu Chiseled (Most Secure)
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0-noble-chiseled AS runtime
# Size: ~110MB, No shell, no package manager, minimal attack surface
# Cannot install additional packages or run shell commands

# Health check must use built-in .NET health check mechanism
# Remove curl-based HEALTHCHECK and use TCP or custom implementation
```

### Azure Linux (Azure Optimized)
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0-azurelinux3.0 AS runtime
# Optimized for Azure, uses tdnf package manager

RUN tdnf update -y && tdnf install -y curl && tdnf clean all
```

## docker-compose.yml

```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: ${PROJECT_NAME}-api
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=db;Database=${PROJECT_NAME};User Id=sa;Password=${DB_PASSWORD};TrustServerCertificate=true
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - app-network

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: ${PROJECT_NAME}-db
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=${DB_PASSWORD}
      - MSSQL_PID=Developer
    ports:
      - "1433:1433"
    volumes:
      - sqlserver-data:/var/opt/mssql
    healthcheck:
      test: /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "$${SA_PASSWORD}" -C -Q "SELECT 1" || exit 1
      interval: 10s
      timeout: 3s
      retries: 10
      start_period: 10s
    networks:
      - app-network

  # Optional: PostgreSQL alternative
  # postgres:
  #   image: postgres:16-alpine
  #   container_name: ${PROJECT_NAME}-postgres
  #   environment:
  #     - POSTGRES_USER=postgres
  #     - POSTGRES_PASSWORD=${DB_PASSWORD}
  #     - POSTGRES_DB=${PROJECT_NAME}
  #   ports:
  #     - "5432:5432"
  #   volumes:
  #     - postgres-data:/var/lib/postgresql/data
  #   healthcheck:
  #     test: ["CMD-SHELL", "pg_isready -U postgres"]
  #     interval: 5s
  #     timeout: 5s
  #     retries: 5
  #   networks:
  #     - app-network

volumes:
  sqlserver-data:
  # postgres-data:

networks:
  app-network:
    driver: bridge
```

## .dockerignore

```
# Build results
**/bin/
**/obj/
**/out/

# IDE and editor files
.vs/
.vscode/
.idea/
*.suo
*.user
*.userosscache
*.sln.docstates

# Version control
.git/
.github/
.gitignore
.gitattributes

# Docker files (avoid recursive copy)
**/Dockerfile*
**/.dockerignore
docker-compose*.yml

# Documentation
*.md
LICENSE
docs/

# Test results
**/TestResults/
**/coverage/

# Other
**/.DS_Store
**/Thumbs.db
*.log
```

## Environment-Specific Dockerfiles

### Development Dockerfile (with hot reload)

```dockerfile
# Dockerfile.dev
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS dev

WORKDIR /app

# Install tools
RUN dotnet tool install --global dotnet-ef
ENV PATH="$PATH:/root/.dotnet/tools"

# Expose ports
EXPOSE 5000
EXPOSE 5001

# Entry point for development with hot reload
ENTRYPOINT ["dotnet", "watch", "run", "--project", "src/${PROJECT_NAME}/${PROJECT_NAME}.csproj"]
```

### Development docker-compose

```yaml
# docker-compose.override.yml (automatically merged in development)
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - ~/.nuget/packages:/root/.nuget/packages:ro
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - DOTNET_USE_POLLING_FILE_WATCHER=1
    ports:
      - "5000:5000"
      - "5001:5001"
```

## GitHub Actions Integration

```yaml
# .github/workflows/docker-build.yml
name: Docker Build and Push

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## Best Practices

- ✅ Use multi-stage builds to minimize image size
- ✅ Copy project files before source for better layer caching
- ✅ Run as non-root user ($APP_UID)
- ✅ Use specific image tags, not `latest`
- ✅ Include health checks
- ✅ Set appropriate environment variables
- ✅ Use .dockerignore to exclude unnecessary files
- ✅ Consider chiseled images for production security
- ✅ Use build arguments for flexible builds
- ❌ Don't include development tools in runtime image
- ❌ Don't hardcode secrets in Dockerfile
- ❌ Don't use `COPY . .` before restoring packages

