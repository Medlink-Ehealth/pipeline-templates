# Pipeline Templates

Reusable GitHub Actions CI/CD workflow templates for Medlink-Ehealth projects.

## Available Templates

| Template | Path | Description |
|----------|------|-------------|
| **Shared CI** | `.github/workflows/shared-ci.yml` | Lightweight pipeline: secrets scan, Docker build + Trivy scan, ACR push |
| **Staging CD** | `.github/workflows/shared-cd-staging.yml` | Helm deploy to staging with smoke tests and auto-rollback |

## Shared CI Pipeline

A lightweight reusable `workflow_call` template that covers secrets scanning, Docker image building with vulnerability scanning, and pushing to Azure Container Registry. Use this when you don't need the full Node.js lint/test/CodeQL suite from `node-ci.yml`.

### Pipeline Overview

```
┌──────────────┐
│   Gitleaks   │
│ Secrets Scan │
└──────┬───────┘
       │
┌──────┴────────────┐
│  docker-build-scan │
│  Docker + Trivy    │
└──────┬────────────┘
       │
┌──────┴──────┐
│   ACR-Push  │
│ (main only) │
└─────────────┘
```

### Jobs

| Job | Description | Depends On | Runs When |
|-----|-------------|------------|-----------|
| **Gitleaks** | Scans full git history for leaked secrets | - | Always |
| **docker-build-scan** | Builds Docker image and runs Trivy scan (fails on CRITICAL/HIGH unfixed CVEs, uploads SARIF to GitHub Security) | Gitleaks | Always |
| **ACR-Push** | Authenticates to Azure via OIDC and pushes image to ACR with `sha` and `latest` tags | docker-build-scan | `main` branch only, when `acr-login-server` is set |

### Quick Start

```yaml
jobs:
  ci:
    uses: Medlink-Ehealth/pipeline-templates/.github/workflows/shared-ci.yml@main
    with:
      acr-login-server: medlink.azurecr.io
      image-name: patient-portal
      dockerfile-path: Dockerfile
      docker-context: .
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

### Inputs

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `working-directory` | string | No | `"."` | Working directory for the project |
| `coverage-threshold` | number | No | `70` | Minimum code coverage percentage (reserved for future use) |
| `dockerfile-path` | string | No | `""` | Path to the Dockerfile (defaults to repo root `Dockerfile`) |
| `docker-context` | string | No | `"."` | Docker build context directory |
| `acr-login-server` | string | No | `""` | ACR login server (e.g. `myregistry.azurecr.io`). If empty, the ACR push job is skipped |
| `image-name` | string | No | `""` | Docker image name |

### Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `AZURE_CLIENT_ID` | No | Azure AD app client ID for OIDC authentication |
| `AZURE_TENANT_ID` | No | Azure AD tenant ID |
| `AZURE_SUBSCRIPTION_ID` | No | Azure subscription ID |

> These secrets are only needed when `acr-login-server` is provided. Authentication uses [OIDC federated credentials](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation) — no stored client secrets required.

#### Full pipeline with ACR push



### Image Tagging

Images pushed to ACR are tagged with:
- `<acr-login-server>/<image-name>:<git-sha>` — immutable, traceable to the exact commit
- `<acr-login-server>/<image-name>:latest` — always points to the most recent `main` build

### Security Features

| Feature | Tool | What it catches |
|---------|------|-----------------|
| **Secrets detection** | Gitleaks | API keys, tokens, passwords, and other credentials across the full git history |
| **Container scanning** | Trivy | OS and library CVEs in the Docker image (fails on unfixed CRITICAL/HIGH); results uploaded to GitHub Security tab |
| **OIDC authentication** | Azure federated tokens | Eliminates stored credentials for ACR access |

---

## Staging CD Pipeline

A reusable `workflow_call` template for deploying to a staging AKS cluster via Helm with automatic smoke testing, rollback on failure, and optional Slack notifications.

### Pipeline Overview

```
┌─────────────────────┐
│       deploy         │
│  helm upgrade        │
│  --atomic --wait     │
└──────────┬──────────┘
           │
┌──────────┴──────────┐
│     smoke-test       │
│  curl /health        │
│  (retry loop)        │
└──────────┬──────────┘
           │
     ┌─────┴─────┐
     │            │
  success      failure
     │            │
     │    ┌───────┴───────┐
     │    │   rollback     │
     │    │  helm rollback │
     │    └───────┬───────┘
     │            │
     └─────┬──────┘
           │
┌──────────┴──────────┐
│      notify          │
│  Slack webhook       │
│  (optional)          │
└─────────────────────┘
```

### Jobs

| Job | Description | Depends On | Runs When |
|-----|-------------|------------|-----------|
| **deploy** | Authenticates to Azure via OIDC, gets AKS credentials, captures the current Helm revision, then runs `helm upgrade --atomic --wait` | - | Always |
| **smoke-test** | Curls each `/health` endpoint in a retry loop with configurable attempts and delay | deploy | deploy succeeds |
| **rollback** | Rolls back to the previous Helm revision, or uninstalls if this was the first deploy | deploy, smoke-test | smoke-test fails |
| **notify** | Sends a Slack message with deploy result (success, rolled back, or failed) | deploy, smoke-test, rollback | Always (if `slack-channel` is set) |

### Quick Start

```yaml
name: Deploy Staging

on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write

jobs:
  deploy-staging:
    uses: Medlink-Ehealth/pipeline-templates/.github/workflows/shared-cd-staging.yml@main
    with:
      cluster-name: aks-staging-cluster
      resource-group: rg-staging
      namespace: staging
      release-name: patient-portal
      chart-path: ./charts/patient-portal
      values-file: ./charts/patient-portal/values-staging.yaml
      image-repository: medlink.azurecr.io/patient-portal
      image-tag: ${{ github.sha }}
      health-endpoints: '["https://api.staging.medlink.com/health"]'
      slack-channel: "#deployments"
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Inputs

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `cluster-name` | string | Yes | - | AKS cluster name |
| `resource-group` | string | Yes | - | Azure resource group containing the AKS cluster |
| `namespace` | string | No | `"staging"` | Kubernetes namespace to deploy into |
| `release-name` | string | Yes | - | Helm release name |
| `chart-path` | string | Yes | - | Path to the Helm chart (local or repo/chart) |
| `values-file` | string | No | `""` | Path to the Helm values file for staging |
| `image-tag` | string | Yes | - | Docker image tag to deploy (typically the git SHA) |
| `image-repository` | string | Yes | - | Full image repository (e.g. `myregistry.azurecr.io/my-app`) |
| `helm-timeout` | string | No | `"5m"` | Timeout for `helm upgrade --atomic` |
| `health-endpoints` | string | Yes | - | JSON array of health check URLs |
| `health-retries` | number | No | `10` | Number of retry attempts per health endpoint |
| `health-retry-delay` | number | No | `15` | Seconds between health check retries |
| `slack-channel` | string | No | `""` | Slack channel (e.g. `#deployments`). If empty, notifications are skipped |
| `helm-set-values` | string | No | `""` | Additional `--set` overrides (comma-separated, e.g. `replicas=2,resources.cpu=250m`) |

### Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `AZURE_CLIENT_ID` | Yes | Azure AD app client ID for OIDC |
| `AZURE_TENANT_ID` | Yes | Azure AD tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Yes | Azure subscription ID |
| `SLACK_WEBHOOK_URL` | No | Slack incoming webhook URL. Only required when `slack-channel` is set |

### Prerequisites

- An **AKS cluster** accessible via the provided Azure credentials
- A **Helm chart** with `image.repository` and `image.tag` values
- The Docker image must already be pushed to the registry (use the [Node.js CI](#nodejs-ci-pipeline) template to build and push first)
- Azure OIDC federated credentials configured (see [Setting Up OIDC for ACR](#setting-up-oidc-for-acr))
- For Slack notifications: an [incoming webhook](https://api.slack.com/messaging/webhooks) stored as `SLACK_WEBHOOK_URL`

### How Rollback Works

1. Before deploying, the pipeline captures the **current Helm revision number**
2. `helm upgrade --atomic` is used, so Helm itself will roll back if the upgrade or readiness probes fail
3. If the upgrade succeeds but **smoke tests fail**, the `rollback` job triggers and runs `helm rollback` to the captured revision
4. If this was the **first deployment** (no prior revision), the failed release is uninstalled with `helm uninstall`
5. Slack notifications report the final outcome: `success`, `rolled_back`, `deploy_failed`, or `rollback_failed`

### Slack Notification States

| State | Color | Meaning |
|-------|-------|---------|
| `success` | Green | Deploy succeeded and all health checks passed |
| `rolled_back` | Orange | Smoke tests failed, rollback completed successfully |
| `deploy_failed` | Red | Helm upgrade itself failed (atomic rollback handled by Helm) |
| `rollback_failed` | Red | Smoke tests and rollback both failed — manual intervention needed |

### Usage Examples

#### Chain CI and CD together

```yaml
name: CI/CD

on:
  push:
    branches: [main]

permissions:
  contents: read
  security-events: write
  id-token: write

jobs:
  ci:
    uses: Medlink-Ehealth/pipeline-templates/.github/workflows/node-ci.yml@main
    with:
      acr-login-server: medlink.azurecr.io
      image-name: patient-portal
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  deploy-staging:
    needs: ci
    uses: Medlink-Ehealth/pipeline-templates/.github/workflows/shared-cd-staging.yml@main
    with:
      cluster-name: aks-staging
      resource-group: rg-staging
      release-name: patient-portal
      chart-path: ./charts/patient-portal
      image-repository: medlink.azurecr.io/patient-portal
      image-tag: ${{ github.sha }}
      health-endpoints: '["https://api.staging.medlink.com/health","https://web.staging.medlink.com/health"]'
      slack-channel: "#deployments"
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

#### Multiple services with custom Helm overrides

```yaml
jobs:
  deploy-api:
    uses: Medlink-Ehealth/pipeline-templates/.github/workflows/shared-cd-staging.yml@main
    with:
      cluster-name: aks-staging
      resource-group: rg-staging
      release-name: api-service
      chart-path: ./charts/api
      image-repository: medlink.azurecr.io/api
      image-tag: ${{ github.sha }}
      health-endpoints: '["https://api.staging.medlink.com/health"]'
      helm-set-values: "replicaCount=2,ingress.enabled=true"
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

## Contributing

1. Create a feature branch from `main`
2. Make your changes
3. Open a pull request — CODEOWNERS will be assigned for review automatically

## License

See [LICENSE](LICENSE) for details.
