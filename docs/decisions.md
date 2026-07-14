# Architecture & DevOps Decisions

This document records key decisions made during the project, and the reasoning behind them.

## Decision: Base repository
Used `dotnet/eShop` (not the older `eShopOnContainers`), since it is Microsoft's actively maintained successor and uses .NET Aspire for local orchestration.

## Decision: Containerization scope
Not all 20 projects in the repo will be containerized. Scope was chosen to balance meeting core requirements with realistic time constraints:

**Core (required):**
- Catalog.API
- Basket.API
- Ordering.API

These three demonstrate different service types: a straightforward API (Catalog), a service with a cache dependency (Basket + Redis), and a service with business logic + database (Ordering).

**Bonus (added if time allows):**
- Identity.API (authentication — adds a security dimension)
- WebApp (storefront UI — gives a demoable, visual result)
- OrderProcessor / PaymentProcessor (event-driven background services)

**Out of scope:**
- ClientApp / HybridApp (.NET MAUI mobile apps — not relevant to server-side container/K8s work)
- mobile-bff, webhooksclient (supporting/demo services for the mobile app, not core to the DevOps pipeline goals)

## Decision: Actual tech stack (confirmed via running the app)
Initial assumptions were revised after actually running the app locally via .NET Aspire:
- Database: **PostgreSQL with pgvector** (not SQL Server, as originally assumed)
- Cache: **Redis** (basket)
- Event bus: **RabbitMQ**
- Local orchestration: **.NET Aspire**

## Decision: Docker build approach
- Multi-stage builds: SDK image for build stage, ASP.NET runtime image for final stage, to minimize final image size and reduce attack surface (no build tooling shipped to production).
- One Dockerfile per service (matches the microservices architecture; each service becomes its own Kubernetes Deployment later).

## Decision: Container registry
GitHub Container Registry (GHCR) — chosen over Docker Hub because it authenticates directly with the `GITHUB_TOKEN` already available in GitHub Actions, avoiding an extra secret to manage.

## Decision: Branching strategy
`main` (protected, PR-only) and `dev` (day-to-day work). This supports a future CI/CD pipeline with separate dev/prod deployment triggers and an approval gate before production deploys.
