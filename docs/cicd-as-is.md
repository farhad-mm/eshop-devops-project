# CI/CD Pipeline (As Implemented)

This document describes the GitHub Actions CI/CD pipeline as it is currently implemented for the eShop DevOps project.

## Purpose

The pipeline automates build, test, security scanning, container image publishing, and deployment of the eShop backend services to the private k3s cluster on VM103.

## Repository and Workflow Location

- Repository: `farhad-mm/eshop-devops-project`
- Primary workflow: `.github/workflows/ci-cd.yml`
- Branches: `dev` and `main`
- Pull requests: targeted at `main`

## Triggers

The CI/CD workflow runs in the following situations:

- Push to `dev` branch
- Push to `main` branch
- Pull request targeting `main`

## Pipeline Stages

The pipeline executes the following stages in order:

### 1. Restore and Unit Tests

- Restores .NET dependencies.
- Runs unit tests for Basket and Ordering services.
- Fails the pipeline if tests do not pass.

### 2. Build Container Images

Builds Docker images for the three core API services:

- Catalog API
- Basket API
- Ordering API

Images are tagged with:

- `latest`
- Git commit SHA (immutable, traceable tag)

### 3. Push Images to GHCR

Images are pushed to GitHub Container Registry:

```text
ghcr.io/farhad-mm/eshop-devops-project/catalog-api:<tag>
ghcr.io/farhad-mm/eshop-devops-project/basket-api:<tag>
ghcr.io/farhad-mm/eshop-devops-project/ordering-api:<tag>
```

Authentication uses the built-in `GITHUB_TOKEN`.

### 4. Trivy Image Vulnerability Scanning

After pushing images, the pipeline installs Trivy and scans each API image for known vulnerabilities:

- Severity filter: `HIGH,CRITICAL`
- Unfixed findings: ignored
- Exit code: `0` (reporting mode, does not block deployment)

Trivy scans commit-specific image tags, making results traceable to the exact source revision.

Findings are visible in GitHub Actions logs. The current scan identified a fixable high-severity `Microsoft.OpenApi` dependency issue, with no critical or Ubuntu OS package vulnerabilities.

### 5. Deploy to k3s on VM103

The deployment stage runs on the self-hosted GitHub Actions runner:

```text
vm103-k3s-runner
```

It applies the Kubernetes manifests in the `k8s/` directory:

```bash
kubectl apply -f k8s/
```

This deploys or updates:

- Catalog API Deployment and Service
- Basket API Deployment and Service
- Ordering API Deployment and Service
- Supporting infrastructure (databases, Redis, RabbitMQ) as defined

### 6. Wait for Kubernetes Rollouts

The pipeline waits for each API Deployment to complete its rollout:

```bash
kubectl rollout status deployment/catalog-api -n default --timeout=120s
kubectl rollout status deployment/basket-api -n default --timeout=120s
kubectl rollout status deployment/ordering-api -n default --timeout=120s
```

This ensures that Kubernetes has recreated pods to match the new Deployment spec before proceeding.

### 7. In-Cluster Smoke Tests

After rollout, the pipeline runs health smoke tests from temporary in-cluster curl pods:

- Catalog API:

  ```bash
  kubectl run smoke-catalog-api `
    --namespace=default `
    --image=curlimages/curl:8.12.1 `
    --restart=Never `
    --rm `
    --attach `
    --command -- `
    sh -c 'for i in $(seq 1 12); do curl --fail --silent --show-error --max-time 5 http://catalog-api:8080/health && exit 0; echo "Catalog not ready; retrying..."; sleep 5; done; exit 1'
  ```

- Basket API (HTTP/2):

  ```bash
  kubectl run smoke-basket-api `
    --namespace=default `
    --image=curlimages/curl:8.12.1 `
    --restart=Never `
    --rm `
    --attach `
    --command -- `
    sh -c 'for i in $(seq 1 12); do curl --http2-prior-knowledge --fail --silent --show-error --max-time 5 http://basket-api:8080/health && exit 0; echo "Basket not ready; retrying..."; sleep 5; done; exit 1'
  ```

- Ordering API:

  ```bash
  kubectl run smoke-ordering-api `
    --namespace=default `
    --image=curlimages/curl:8.12.1 `
    --restart=Never `
    --rm `
    --attach `
    --command -- `
    sh -c 'for i in $(seq 1 12); do curl --fail --silent --show-error --max-time 5 http://ordering-api:8080/health && exit 0; echo "Ordering not ready; retrying..."; sleep 5; done; exit 1'
  ```

These tests verify:

- Pods are running and healthy.
- Kubernetes Services correctly route internal traffic.
- The API health endpoints respond successfully.
- Basket API is validated using HTTP/2, matching its service configuration.

The pipeline fails if any core API does not become reachable and healthy.

## Security Controls in CI/CD

- **Trivy image scanning**: Identifies known vulnerabilities in container images before deployment.
- **Gitleaks secret scanning**: Separate workflow that scans repository files and full Git history for accidentally committed secrets.
- **Immutable image tags**: Commit-SHA tags make each deployment traceable to a specific source revision.
- **Private deployment target**: The runner deploys to a private k3s cluster, not a public endpoint.

## Current Limitations

- Trivy is configured in reporting mode and does not block deployment on high or critical findings.
- No automated dependency update workflow is implemented.
- No approval gate or manual deployment stage is configured.
- The WebApp/storefront is not deployed to k3s through this pipeline.

## Related Documentation

- [Monitoring documentation](monitoring-as-is.md)
- [Security documentation](security-as-is.md)
- [Disaster recovery overview](disaster-recovery/disaster-recovery-overview.md)
- [K3s deployment and demo access](k3s-deployment-and-demo-access.md)
