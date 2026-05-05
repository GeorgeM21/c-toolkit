---
agent: 'Feature-Scaffolder'
description: 'Add CI/CD pipeline with GitHub Actions for .NET projects'
# Note: Uses Feature-Scaffolder agent if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Add GitHub Actions CI/CD Pipeline

## Task

Create GitHub Actions workflows for continuous integration and deployment of .NET applications, including build, test, security scanning, and deployment stages.

## Configuration

| Setting | Options | Default |
|---------|---------|---------|
| .NET Version | 8.0, 9.0 | 9.0 |
| Test Framework | xUnit, NUnit, MSTest | xUnit |
| Deploy Target | Azure App Service, Azure Container Apps, Docker Registry, None | Azure Container Apps |
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
    paths-ignore:
      - '**.md'
      - 'docs/**'
  pull_request:
    branches: [main, develop]
    paths-ignore:
      - '**.md'
      - 'docs/**'

env:
  DOTNET_VERSION: '9.0.x'
  DOTNET_SKIP_FIRST_TIME_EXPERIENCE: true
  DOTNET_CLI_TELEMETRY_OPTOUT: true

jobs:
  build:
    name: Build and Test
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for versioning

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}
          
      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj', '**/Directory.Packages.props') }}
          restore-keys: |
            ${{ runner.os }}-nuget-

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore --configuration Release

      - name: Test
        run: dotnet test --no-build --configuration Release --verbosity normal --collect:"XPlat Code Coverage" --results-directory ./coverage

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          directory: ./coverage
          fail_ci_if_error: false

      - name: Publish
        run: dotnet publish src/${PROJECT_NAME}/${PROJECT_NAME}.csproj --no-build --configuration Release --output ./publish

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: app
          path: ./publish
          retention-days: 7

  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: build
    permissions:
      security-events: write
      
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Restore dependencies
        run: dotnet restore

      - name: Run security scan
        run: |
          dotnet list package --vulnerable --include-transitive 2>&1 | tee security-scan.txt
          
          if grep -q "has the following vulnerable packages" security-scan.txt; then
            echo "::warning::Vulnerable packages detected"
          fi

  code-quality:
    name: Code Quality
    runs-on: ubuntu-latest
    needs: build
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Restore dependencies
        run: dotnet restore

      - name: Check formatting
        run: dotnet format --verify-no-changes --verbosity diagnostic

  docker-build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [build, security-scan]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

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
        options:
          - staging
          - production

env:
  AZURE_CONTAINER_APP: ${PROJECT_NAME}-app
  AZURE_RESOURCE_GROUP: ${PROJECT_NAME}-rg
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
      - name: Checkout
        uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Deploy to Azure Container Apps
        uses: azure/container-apps-deploy-action@v1
        with:
          resourceGroup: ${{ env.AZURE_RESOURCE_GROUP }}-staging
          containerAppName: ${{ env.AZURE_CONTAINER_APP }}
          imageToDeploy: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

      - name: Run smoke tests
        run: |
          sleep 30  # Wait for deployment
          curl -f https://${PROJECT_NAME}-staging.azurecontainerapps.io/health || exit 1

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: ${{ github.event.inputs.environment == 'production' }}
    environment:
      name: production
      url: https://${PROJECT_NAME}.azurecontainerapps.io
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Deploy to Azure Container Apps
        uses: azure/container-apps-deploy-action@v1
        with:
          resourceGroup: ${{ env.AZURE_RESOURCE_GROUP }}-prod
          containerAppName: ${{ env.AZURE_CONTAINER_APP }}
          imageToDeploy: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

      - name: Run smoke tests
        run: |
          sleep 30
          curl -f https://${PROJECT_NAME}.azurecontainerapps.io/health || exit 1
```

## CodeQL Security Scanning (codeql.yml)

```yaml
name: CodeQL

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '30 1 * * 0'  # Weekly on Sunday at 1:30 AM

jobs:
  analyze:
    name: Analyze
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      actions: read
      contents: read

    strategy:
      fail-fast: false
      matrix:
        language: [csharp]

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'

      - name: Build
        run: dotnet build --configuration Release

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: "/language:${{ matrix.language }}"
```

## Dependabot Configuration

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "nuget"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 10
    groups:
      dotnet-minor:
        patterns:
          - "Microsoft.*"
          - "System.*"
        update-types:
          - "minor"
          - "patch"
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

## Pull Request Template

```markdown
<!-- .github/pull_request_template.md -->
## Description

<!-- Describe your changes -->

## Type of Change

- [ ] Bug fix (non-breaking change fixing an issue)
- [ ] New feature (non-breaking change adding functionality)
- [ ] Breaking change (fix or feature causing existing functionality to change)
- [ ] Documentation update

## Checklist

- [ ] My code follows the project's style guidelines
- [ ] I have performed a self-review of my code
- [ ] I have added tests that prove my fix/feature works
- [ ] New and existing unit tests pass locally
- [ ] I have updated the documentation accordingly

## Related Issues

<!-- Link any related issues using: Fixes #123, Closes #456 -->
```

## Required Secrets

Configure these secrets in your GitHub repository:

| Secret | Description |
|--------|-------------|
| `AZURE_CREDENTIALS` | Azure service principal JSON for deployments |
| `CODECOV_TOKEN` | Codecov token for coverage reports |
| `NUGET_API_KEY` | NuGet API key (for package publishing) |

## Best Practices

- ✅ Use caching for NuGet packages and Docker layers
- ✅ Run security scans on every build
- ✅ Use environments for deployment approvals
- ✅ Fail fast on security issues
- ✅ Include smoke tests after deployment
- ✅ Use matrix builds for multiple .NET versions if needed
- ✅ Pin action versions for reproducibility
- ✅ Use `paths-ignore` to skip unnecessary builds
- ❌ Don't store secrets in workflow files
- ❌ Don't skip tests for "quick fixes"
- ❌ Don't deploy directly to production without staging
