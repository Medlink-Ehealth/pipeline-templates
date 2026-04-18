# Pipeline Templates

Reusable GitHub Actions CI/CD workflow templates for Medlink-Ehealth projects.

## Available Templates

| Template | Path | Description |
|----------|------|-------------|
| **Node.js CI** | `.github/workflows/node-ci.yml` | Full CI/CD pipeline for Node.js/TypeScript projects |

---

## Node.js CI Pipeline

A reusable `workflow_call` template that provides a complete CI/CD pipeline for Node.js and TypeScript applications. It runs code quality checks, tests, security scans, container builds, and deployment to Azure Container Registry.

### Pipeline Overview

```
┌─────────────────────┐   ┌──────────────────┐   ┌─────────────────┐
│  lint-and-typecheck  │   │    codeql         │   │    gitleaks      │
│  ESLint + Prettier   │   │    SAST Analysis  │   │    Secrets Scan  │
│  tsc --noEmit        │   └────────┬─────────┘   └────────┬────────┘
└──────────┬──────────┘            │                       │
           │                       │                       │
┌──────────┴──────────┐            │                       │
│       test           │            │                       │
│  Vitest + Coverage   │            │                       │
└──────────┬──────────┘            │                       │
           │                       │                       │
┌──────────┴──────────┐            │                       │
│  docker-build-scan   │            │                       │
│  Docker + Trivy      │            │                       │
└──────────┬──────────┘            │                       │
           │                       │                       │
           └───────────┬───────────┘───────────────────────┘
                       │
              ┌────────┴────────┐
              │    acr-push      │
              │  Push to ACR     │
              │  (main only)     │
              └─────────────────┘
```

### Jobs

| Job | Description | Depends On |
|-----|-------------|------------|
| **lint-and-typecheck** | Runs ESLint (zero warnings policy), Prettier format check, and `tsc --noEmit` | - |
| **test** | Runs Vitest with coverage enforcement (default 70% threshold for lines, functions, branches, statements) | - |
| **codeql** | GitHub CodeQL static analysis for JavaScript/TypeScript | - |
| **gitleaks** | Scans full git history for leaked secrets | - |
| **docker-build-scan** | Builds Docker image and runs Trivy vulnerability scan (fails on CRITICAL/HIGH) | lint-and-typecheck, test |
| **acr-push** | Authenticates to Azure via OIDC and pushes image to ACR | docker-build-scan, codeql, gitleaks |

### Quick Start

Create a workflow file in your repository (e.g. `.github/workflows/ci.yml`):

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  security-events: write
  id-token: write

jobs:
  ci:
    uses: Medlink-Ehealth/pipeline-templates/.github/workflows/node-ci.yml@main
    with:
      node-version: "20"
      coverage-threshold: 70
      acr-login-server: myregistry.azurecr.io
      image-name: my-app
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

### Inputs

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `node-version` | string | No | `"20"` | Node.js version for setup-node |
| `working-directory` | string | No | `"."` | Working directory for npm commands |
| `coverage-threshold` | number | No | `70` | Minimum coverage percentage (applied to lines, functions, branches, and statements) |
| `dockerfile-path` | string | No | `"./Dockerfile"` | Path to Dockerfile |
| `docker-context` | string | No | `"."` | Docker build context directory |
| `acr-login-server` | string | No | `""` | ACR login server (e.g. `myregistry.azurecr.io`). If empty, the ACR push job is skipped |
| `image-name` | string | No | `""` | Docker image name for ACR |

### Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `AZURE_CLIENT_ID` | No | Azure AD app client ID for OIDC authentication |
| `AZURE_TENANT_ID` | No | Azure AD tenant ID |
| `AZURE_SUBSCRIPTION_ID` | No | Azure subscription ID |

> These secrets are only required when `acr-login-server` is provided. Authentication uses [OIDC federated credentials](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation) — no client secrets are stored in GitHub.

### Prerequisites

#### For all projects

- **package-lock.json** must be committed (required for `npm ci` and dependency caching)
- **ESLint** and **Prettier** must be configured in the project (via config files or `package.json`)
- **TypeScript** must be configured with a `tsconfig.json`
- **Vitest** must be installed with a coverage provider (e.g. `@vitest/coverage-v8`)

#### For Docker and ACR push

- A **Dockerfile** must exist at the configured path
- An **Azure AD app registration** with federated credentials configured for the GitHub repository
- The app registration must have **AcrPush** role on the target ACR

### Setting Up OIDC for ACR

1. Create an Azure AD app registration
2. Add a federated credential for your GitHub repository:
   - Organization: `Medlink-Ehealth`
   - Repository: `<your-repo>`
   - Entity type: `Branch` (for `main`) or `Pull Request`
3. Assign the **AcrPush** role to the app on your ACR:
   ```bash
   az role assignment create \
     --assignee <APP_CLIENT_ID> \
     --role AcrPush \
     --scope /subscriptions/<SUB_ID>/resourceGroups/<RG>/providers/Microsoft.ContainerRegistry/registries/<ACR_NAME>
   ```
4. Add the following secrets to your GitHub repository:
   - `AZURE_CLIENT_ID`
   - `AZURE_TENANT_ID`
   - `AZURE_SUBSCRIPTION_ID`

### Usage Examples

#### Minimal (lint, test, and security scans only)

```yaml
jobs:
  ci:
    uses: Medlink-Ehealth/pipeline-templates/.github/workflows/node-ci.yml@main
```

#### Custom coverage threshold and Node version

```yaml
jobs:
  ci:
    uses: Medlink-Ehealth/pipeline-templates/.github/workflows/node-ci.yml@main
    with:
      node-version: "22"
      coverage-threshold: 80
```

#### Monorepo (subdirectory)

```yaml
jobs:
  ci:
    uses: Medlink-Ehealth/pipeline-templates/.github/workflows/node-ci.yml@main
    with:
      working-directory: packages/api
      dockerfile-path: packages/api/Dockerfile
      docker-context: .
```

#### Full pipeline with ACR deployment

```yaml
jobs:
  ci:
    uses: Medlink-Ehealth/pipeline-templates/.github/workflows/node-ci.yml@main
    with:
      node-version: "20"
      coverage-threshold: 70
      acr-login-server: medlink.azurecr.io
      image-name: patient-portal
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

### Image Tagging

Images pushed to ACR are tagged with:
- `<acr-login-server>/<image-name>:<git-sha>` — immutable, traceable to the exact commit
- `<acr-login-server>/<image-name>:latest` — convenience tag, always points to the most recent main build

### Security Features

| Feature | Tool | What it catches |
|---------|------|-----------------|
| **SAST** | CodeQL | SQL injection, XSS, prototype pollution, insecure randomness, and other vulnerability patterns |
| **Secrets detection** | Gitleaks | API keys, tokens, passwords, and other credentials in the full git history |
| **Container scanning** | Trivy | OS and library vulnerabilities in the Docker image (fails on CRITICAL/HIGH) |
| **OIDC authentication** | Azure federated tokens | Eliminates stored credentials for ACR access |

## Contributing

1. Create a feature branch from `main`
2. Make your changes
3. Open a pull request — CODEOWNERS will be assigned for review automatically

## License

See [LICENSE](LICENSE) for details.
