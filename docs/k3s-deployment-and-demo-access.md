# K3s Deployment and Committee Demo Access

This document records the verified current deployment of the eShop DevOps project on VM103 and provides a safe, repeatable procedure for a committee demonstration.

## Purpose

The project uses a private single-node k3s environment on VM103 to deploy and operate the core eShop backend services.

The documented environment includes:

- Kubernetes deployment of Catalog API, Basket API, and Ordering API
- PostgreSQL databases for Catalog and Ordering
- Redis for Basket data
- RabbitMQ as the event bus
- Traefik private Ingress routing
- Prometheus, Grafana, and related monitoring components
- GitHub Actions continuous integration, security checks, and automated deployment through a self-hosted runner

The environment is intentionally private. It is not presented as a public production website.

## VM103 Access Model

VM103 is accessed from the developer laptop through the configured SSH alias:

```powershell
ssh eshop-k3s
```

Verified environment details:

| Item | Value |
|---|---|
| VM hostname | `eshopk3s` |
| VM role | Private k3s node and GitHub Actions self-hosted runner |
| VM private IP | `10.10.10.198` |
| Kubernetes distribution | k3s |
| Cluster topology | Single-node control plane |
| Kubernetes namespace for eShop APIs | `default` |
| Kubernetes namespace for monitoring | `monitoring` |
| Ingress controller | Traefik, included with k3s |
| Public IP / public DNS | Not configured |
| TLS certificate | Not configured |

The developer laptop connects through the configured SSH ProxyJump path. Access to the VM, Kubernetes API, Traefik routes, and Grafana remains private.

## Deployed Application Workloads

The current Kubernetes deployment in namespace `default` contains the following application and infrastructure workloads:

| Workload | Purpose |
|---|---|
| `catalog-api` | Product catalog REST API |
| `basket-api` | Basket API using gRPC and HTTP/2 |
| `ordering-api` | Ordering REST API |
| `catalog-db` | PostgreSQL / pgvector data store for Catalog |
| `ordering-db` | PostgreSQL / pgvector data store for Ordering |
| `redis` | Basket data store |
| `rabbitmq` | Event bus and asynchronous messaging |

To inspect the application workloads after connecting to VM103, run:

```bash
sudo kubectl get pods -n default -o wide
sudo kubectl get deployments -n default
sudo kubectl get svc -n default
```

A healthy expected state is that the application pods are `Running`, the API Deployments have available replicas, and the internal API Services are available.

## Automated CI/CD Deployment

The project uses GitHub Actions for build, test, security scanning, container image publishing, and deployment to k3s.

The deployment workflow uses a self-hosted GitHub Actions runner on VM103:

```text
vm103-k3s-runner
```

The deployment path is:

```text
Push to dev or main branch
        ↓
Run unit tests
        ↓
Build Catalog, Basket, and Ordering images
        ↓
Push commit-specific images to GitHub Container Registry
        ↓
Run Trivy image vulnerability scanning
        ↓
Deploy Kubernetes manifests through the VM103 self-hosted runner
        ↓
Wait for Kubernetes Deployment rollouts
        ↓
Run in-cluster health smoke tests
```

The deployment workflow verifies the core APIs using Kubernetes Service DNS names from temporary in-cluster curl pods:

- Catalog API: `http://catalog-api:8080/health`
- Basket API: `http://basket-api:8080/health` using HTTP/2
- Ordering API: `http://ordering-api:8080/health`

For detailed CI/CD information, see [CI/CD documentation](cicd-as-is.md) after Card 33 is completed.

## Private Traefik Ingress

k3s includes Traefik as the Ingress controller. The project has an Ingress named `eshop-ingress` in namespace `default`.

Verified Ingress details:

| Item | Value |
|---|---|
| Ingress name | `eshop-ingress` |
| Namespace | `default` |
| Ingress class | `traefik` |
| Private Ingress address | `10.10.10.198` |
| Entry point | HTTP port `80` |
| TLS | Not configured |
| Public DNS | Not configured |

The private routes are:

| Request path | Backend service |
|---|---|
| `/catalog` | `catalog-api:8080` |
| `/basket` | `basket-api:8080` |
| `/ordering` | `ordering-api:8080` |

Inspect the configured Ingress on VM103:

```bash
sudo kubectl get ingress -n default
sudo kubectl describe ingress eshop-ingress -n default
```

Do not expose VM103, Traefik, Grafana, or Kubernetes services to the public internet solely for the committee demonstration.

## API Demonstration Commands

After connecting to VM103 with `ssh eshop-k3s`, use the following safe read-only commands.

### Catalog health

```bash
curl -i http://127.0.0.1/catalog/health
```

Expected result:

```text
HTTP/1.1 200 OK
Healthy
```

### Ordering health

```bash
curl -i http://127.0.0.1/ordering/health
```

Expected result:

```text
HTTP/1.1 200 OK
Healthy
```

### Catalog API data through private Ingress

```bash
curl -i "http://127.0.0.1/catalog/api/catalog/items?api-version=1.0&pageSize=2"
```

Expected result: HTTP `200 OK` and JSON containing catalog products.

### Basket API note

Basket API is a gRPC service that uses HTTP/2. A browser-style request such as `curl http://127.0.0.1/basket/` may return `404` or an HTTP/2-related response. This does not by itself indicate that the Basket API pod is unhealthy.

Verify Basket API deployment status using:

```bash
sudo kubectl get pods -n default -l app=basket-api
sudo kubectl get deployment basket-api -n default
```

The automated CI/CD pipeline verifies Basket API internally using an HTTP/2 health request.

## Monitoring Access

Monitoring is deployed through the `kube-prometheus-stack` Helm chart in namespace `monitoring`.

The monitoring stack includes:

- Prometheus
- Grafana
- Alertmanager
- Prometheus Operator
- kube-state-metrics
- node-exporter

Verify the monitoring deployment on VM103:

```bash
sudo kubectl get pods -n monitoring
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm list -A
```

Grafana remains private. For a committee demonstration, access it through Kubernetes port-forwarding.

First connect to VM103:

```powershell
ssh eshop-k3s
```

Then run this command on VM103:

```bash
sudo kubectl -n monitoring port-forward svc/monitoring-grafana 3000:80
```

Open the following address in the browser on the developer laptop:

```text
http://localhost:3000
```

Stop port-forwarding using `Ctrl+C` after the demonstration.

## Committee Demonstration Sequence

Use the following sequence during the live committee demonstration.

1. Show the architecture diagram and explain that VM103 is a private environment.
2. Show the GitHub Actions pipeline, including tests, GHCR image publishing, Trivy scanning, deployment, rollout checks, and smoke tests.
3. Show the separate Gitleaks Secret Scan workflow.
4. Connect to VM103:

   ```powershell
   ssh eshop-k3s
   ```

5. Show the application workloads:

   ```bash
   sudo kubectl get pods -n default -o wide
   ```

6. Show the private Traefik Ingress:

   ```bash
   sudo kubectl get ingress -n default
   ```

7. Demonstrate Catalog and Ordering health:

   ```bash
   curl -i http://127.0.0.1/catalog/health
   curl -i http://127.0.0.1/ordering/health
   ```

8. Show live Catalog API data:

   ```bash
   curl -s "http://127.0.0.1/catalog/api/catalog/items?api-version=1.0&pageSize=2"
   ```

9. Show monitoring workloads:

   ```bash
   sudo kubectl get pods -n monitoring
   ```

10. Start Grafana port-forwarding and show the selected dashboard.
11. Show the rollback runbook and recovery documentation.
12. Explain current limitations and planned future improvements honestly.

## Current Scope and Limitations

The following statements define the current verified scope:

- The core Catalog, Basket, and Ordering backend services are deployed to VM103 k3s.
- GitHub Actions deploys the Kubernetes manifests through the VM103 self-hosted runner.
- The project uses private Traefik HTTP routing, not public internet exposure.
- Public DNS and a public TLS certificate are not configured.
- Grafana is private and accessed through SSH plus Kubernetes port-forwarding.
- The WebApp/storefront is demonstrated locally through .NET Aspire and Docker Compose; it is not currently deployed as a Kubernetes workload on VM103.
- The k3s cluster has one node, so it does not provide multi-node control-plane high availability.
- Recovery, backup, and rollback are documented separately under `docs/disaster-recovery/`.

## Related Documentation

- [Architecture diagram](architecture/eshop-devops-architecture.png)
- [Editable architecture diagram](architecture/eshop-devops-architecture.drawio)
- [Kubernetes rollback procedure](disaster-recovery/card-29-rollback-procedure.md)
- [Proxmox infrastructure documentation](proxmox-infrastructure.md)
- [Container and node recovery procedure](disaster-recovery/card-32-container-node-recovery.md)
- [CI/CD documentation](cicd-as-is.md)
- [Monitoring documentation](monitoring-as-is.md)
- [Security documentation](security-as-is.md)
- [Disaster recovery overview](disaster-recovery/disaster-recovery-overview.md)
