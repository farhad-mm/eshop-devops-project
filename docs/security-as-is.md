# Security Controls (As Implemented)

This document describes the security controls currently implemented for the eShop DevOps project.

## Purpose

The project implements a set of security controls across source control, CI/CD, container images, and runtime access. This document records what is in place and identifies known gaps relative to a production environment.

## Source Control and Secret Scanning

- **Gitleaks secret scanning**: A dedicated GitHub Actions workflow scans the repository and full Git history for accidentally committed secrets, including passwords, API keys, tokens, private keys, and connection strings.
- The scan runs on pushes to `dev` and `main`, on pull requests targeting `main`, and can be triggered manually.
- If a secret is detected, the workflow fails and requires remediation before the change can be accepted.

## CI/CD Security Controls

- **Trivy image vulnerability scanning**: After building and pushing container images, Trivy scans each API image for known vulnerabilities in operating-system packages and .NET dependencies.
- Severity filter: `HIGH,CRITICAL`.
- Current mode: reporting (does not block deployment).
- Images are scanned using immutable commit-SHA tags, making results traceable to the exact source revision.

## Container and Runtime Security

- **Private k3s cluster**: The cluster runs on VM103 with a private IP and no public exposure.
- **Private services**: API services are internal `ClusterIP` Services.
- **Private Ingress**: Traefik Ingress is reachable only via the private IP `10.10.10.198`.
- **Runner permissions**: The self-hosted runner has controlled access to k3s via group-readable kubeconfig permissions.

## Authentication and Access

- **SSH key authentication**: Access to VM103 uses key-based SSH with ProxyJump.
- **No public web exposure**: The environment is not presented as a public production website.
- **Grafana access**: Private, through SSH plus Kubernetes port-forwarding.

## Known Gaps and Future Improvements

- **Trivy not blocking**: Trivy currently reports findings but does not block deployment on high or critical vulnerabilities.
- **Dependency updates**: No automated dependency update workflow is implemented.
- **Public TLS/DNS**: No public DNS or TLS certificate is configured.
- **Secret management**: Development credentials are used in Kubernetes manifests; production-grade secret management (e.g., external secrets, sealed secrets, or a dedicated vault) is not implemented.
- **Image signing**: Container images are not signed or verified at deployment time.

## Trivy Finding Summary

The current Trivy scan identified:

- Zero critical vulnerabilities.
- Zero Ubuntu OS package vulnerabilities.
- One high-severity, fixable .NET dependency issue in `Microsoft.OpenApi` (CVE-2026-49451) across all three API images.

This finding is documented as a remediation item for a future dependency update.

## Related Documentation

- [CI/CD documentation](cicd-as-is.md)
- [Monitoring documentation](monitoring-as-is.md)
- [Disaster recovery overview](disaster-recovery/disaster-recovery-overview.md)
- [K3s deployment and demo access](k3s-deployment-and-demo-access.md)
