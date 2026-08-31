# Container and Node Recovery Procedure

This document describes how to diagnose and recover from failures affecting application containers, pods, deployments, and the VM103 k3s node. It complements the backup strategy (Card 28) and the Kubernetes rollback procedure (Card 29).

## Purpose

The eShop application runs on a single-node k3s cluster on VM103. The deployed core services are:

- Catalog API
- Basket API
- Ordering API
- Catalog PostgreSQL database
- Ordering PostgreSQL database
- Redis
- RabbitMQ

This document provides:

- A practical approach to diagnosing container and pod failures.
- Safe recovery commands for API deployments.
- Post-recovery validation steps.
- Guidance for VM103/node-level failures.
- Clear boundaries between current capabilities and future improvements.

## Kubernetes Self-Healing for Pod Failures

Kubernetes Deployments provide automatic self-healing for transient container failures.

If a container crashes or exits unexpectedly:

1. The kubelet on VM103 detects the failure.
2. The Deployment controller recreates the pod to match the desired replica count.
3. The new pod is scheduled on the same node (eshopk3s) and starts using the existing Deployment spec.

For the eShop APIs, this means that a single failing pod is typically recreated automatically without manual intervention.

## Diagnosing a Failed or Unhealthy Pod

Use the following commands on VM103 to diagnose a failing or unhealthy workload.

Connect to VM103:

```powershell
ssh eshop-k3s
```

Then run:

```bash
# List all pods and their status
sudo kubectl get pods -n default -o wide

# Describe a specific failing pod
sudo kubectl describe pod <pod-name> -n default

# View recent events for the pod
sudo kubectl get events -n default --sort-by='.lastTimestamp'

# Inspect container logs
sudo kubectl logs <pod-name> -n default
```

Key fields to check in `kubectl describe pod`:

- `State`: Running, Waiting, or Terminated
- `Last State`: previous container state if it crashed
- `Reason`: e.g. `Error`, `CrashLoopBackOff`, `ImagePullBackOff`
- `Restart Count`: number of container restarts
- `Events`: scheduling, pulling, starting, and failure messages

Common failure patterns:

- `CrashLoopBackOff`: container repeatedly exits; check application logs.
- `ImagePullBackOff`: image cannot be pulled; check image name, tag, and GHCR access.
- `Pending`: insufficient resources or scheduling issues.
- `Error` in `Last State`: application-level failure; inspect logs.

## Recovering a Failed API Deployment

If an API pod is failing and self-healing is not sufficient, use the following safe recovery approach.

### 1. Confirm the affected service

Identify which API is affected:

```bash
sudo kubectl get pods -n default -l app=catalog-api
sudo kubectl get pods -n default -l app=basket-api
sudo kubectl get pods -n default -l app=ordering-api
```

Or check deployments directly:

```bash
sudo kubectl get deployment catalog-api basket-api ordering-api -n default
```

### 2. Inspect logs and events

For the affected pod:

```bash
sudo kubectl describe pod <pod-name> -n default
sudo kubectl logs <pod-name> -n default
```

Look for:

- Application exceptions or startup failures.
- Database connection errors.
- Missing configuration or secrets.
- Resource constraints (CPU/memory).

### 3. Trigger a safe deployment restart

If the issue appears to be a transient runtime problem, restart the Deployment:

```bash
sudo kubectl rollout restart deployment/<deployment-name> -n default
```

Examples:

```bash
sudo kubectl rollout restart deployment/catalog-api -n default
sudo kubectl rollout restart deployment/basket-api -n default
sudo kubectl rollout restart deployment/ordering-api -n default
```

This causes Kubernetes to recreate the pods using the current Deployment spec.

### 4. Verify rollout and pod health

After restarting:

```bash
sudo kubectl rollout status deployment/<deployment-name> -n default --timeout=120s

sudo kubectl get pods -n default -l app=<api-name>

sudo kubectl get deployment <deployment-name> -n default
```

Expected healthy state:

- All pods are `Running`.
- `READY` shows `1/1` for each API pod.
- `AVAILABLE` matches the desired replica count.

### 5. Validate API health endpoints

After the pods are healthy, verify the API health endpoints through the private Ingress:

```bash
curl -i --max-time 10 http://127.0.0.1/catalog/health
curl -i --max-time 10 http://127.0.0.1/ordering/health
```

Expected result for each:

```text
HTTP/1.1 200 OK
Healthy
```

## Recovery Considerations for Stateful Dependencies

The eShop platform depends on several stateful services.

### PostgreSQL databases

- `catalog-db` and `ordering-db` run as Kubernetes Deployments with persistent storage.
- If a database pod fails, Kubernetes will recreate it.
- If the underlying volume or data is corrupted, recovery requires the backup procedure documented in Card 28.

### Redis

- Used by Basket API for session and basket data.
- If the Redis pod fails, Kubernetes recreates it.
- Data persistence depends on the configured storage; loss of data requires restoration from backup.

### RabbitMQ

- Provides asynchronous messaging between services.
- If the RabbitMQ pod fails, Kubernetes recreates it.
- Persistent messages depend on configured storage; loss requires backup-based recovery.

For all stateful services, the current approach relies on Kubernetes self-healing for transient failures and on the backup strategy for data-loss scenarios.

## VM103 or Node-Level Failure

VM103 is a single-node k3s cluster. This has important implications:

- If VM103 becomes unavailable, all workloads become unavailable.
- There is no second node to fail over to.
- Recovery depends on restoring the VM itself.

### VM103 recovery approach

1. Use the Proxmox infrastructure to restart or restore VM103.
2. Verify that the k3s service is running after the VM restarts.
3. Confirm that all Deployments and pods return to a healthy state.

Relevant documentation (Card 28 backup strategy to be added):\n\n- [Proxmox infrastructure documentation](../proxmox-infrastructure.md)

### k3s service verification

After a VM restart, on VM103 run:

```bash
sudo systemctl status k3s
sudo kubectl get nodes
sudo kubectl get pods -A
```

If k3s is not running:

```bash
sudo systemctl start k3s
```

## Relationship to Backup and Rollback Procedures

This recovery procedure works alongside:

- **Card 28 â€” Backup Strategy and Verification**  
  Defines backup scope for k3s control-plane state, PostgreSQL databases, repository/configuration, and VM-level recovery.

- **Card 29 â€” Kubernetes Deployment Rollback Procedure**  
  Defines how to roll back a bad application release using immutable image tags and Git as the source of truth.

Use Card 29 when:

- A new deployment introduces a breaking change or bug.
- You need to return to a known-good application version.

Use this Card 32 procedure when:

- A pod or container is failing or unhealthy.
- A deployment needs to be restarted.
- VM103 or k3s needs to be recovered after an infrastructure issue.

Use Card 28 when:

- Data loss or corruption is suspected.
- A full environment rebuild is required.

## Current Limitations and Future Improvements

### Current limitations

- The k3s cluster has only one node; there is no control-plane or worker-node high availability.
- Automated backup scheduling and restore testing are documented but not continuously enforced.
- Recovery from data loss requires manual execution of backup/restore procedures.

### Future improvements

- Implement automated, scheduled backups for PostgreSQL and k3s state.
- Periodically test restore procedures in a non-production environment.
- Consider multi-node or externalized control-plane options for higher availability in a production-like environment.

## Summary of Key Commands

Diagnosis:

```bash
sudo kubectl get pods -n default -o wide
sudo kubectl describe pod <pod-name> -n default
sudo kubectl logs <pod-name> -n default
sudo kubectl get events -n default --sort-by='.lastTimestamp'
```

Deployment restart:

```bash
sudo kubectl rollout restart deployment/<deployment-name> -n default
```

Rollout and health verification:

```bash
sudo kubectl rollout status deployment/<deployment-name> -n default --timeout=120s
sudo kubectl get pods -n default -l app=<api-name>
curl -i http://127.0.0.1/catalog/health
curl -i http://127.0.0.1/ordering/health
```

k3s service check:

```bash
sudo systemctl status k3s
sudo kubectl get nodes
sudo kubectl get pods -A
```

## Related Documentation

- [Backup Strategy and Verification](../disaster-recovery/card-28-backup-strategy.md)
- [Kubernetes Rollback Procedure](../disaster-recovery/card-29-rollback-procedure.md)
- [Proxmox Infrastructure Documentation](../proxmox-infrastructure.md)
- [K3s Deployment and Committee Demo Access](../k3s-deployment-and-demo-access.md)