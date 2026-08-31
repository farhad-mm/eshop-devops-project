# Monitoring and Observability (As Implemented)

This document describes the monitoring stack currently deployed for the eShop DevOps project.

## Purpose

The monitoring stack provides visibility into the health and performance of the k3s cluster, the eShop API services, and the underlying node.

## Deployment Details

- Tooling: `kube-prometheus-stack` Helm chart
- Namespace: `monitoring`
- Helm release name: `monitoring`
- Chart version: `kube-prometheus-stack-88.5.4`
- App version: `v0.93.1`

Verify the deployment on VM103:

```bash
sudo kubectl get pods -n monitoring
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm list -A
```

## Components

The monitoring stack includes:

- **Prometheus**: Metrics collection and storage.
- **Grafana**: Dashboards and visualization.
- **Alertmanager**: Alert routing and notification management.
- **Prometheus Operator**: Kubernetes-native management of Prometheus resources.
- **kube-state-metrics**: Kubernetes object-state metrics.
- **node-exporter**: Node-level metrics (CPU, memory, disk, network).

## Access Model

All monitoring components run inside the private k3s cluster.

- Grafana is exposed as a private `ClusterIP` Service.
- There is no public URL, public DNS, or TLS termination.
- Access is through SSH plus Kubernetes port-forwarding.

From the developer laptop:

```powershell
ssh eshop-k3s
```

Then on VM103:

```bash
sudo kubectl -n monitoring port-forward svc/monitoring-grafana 3000:80
```

Open in the browser:

```text
http://localhost:3000
```

Stop port-forwarding with `Ctrl+C` after use.

## Dashboards

The stack provides:

- Prebuilt Kubernetes cluster dashboards.
- Node resource dashboards.
- Application-oriented dashboards where configured.

For the committee demonstration, select a dashboard that clearly shows:

- Cluster health.
- Pod and node resource usage.
- API service availability.

## Current Limitations

- No custom eShop application metrics are currently instrumented.
- Alerting rules are not tuned for production-grade incident response.
- Grafana access remains private and manual (port-forward only).

## Related Documentation

- [CI/CD documentation](cicd-as-is.md)
- [Security documentation](security-as-is.md)
- [Disaster recovery overview](disaster-recovery/disaster-recovery-overview.md)
- [K3s deployment and demo access](k3s-deployment-and-demo-access.md)
