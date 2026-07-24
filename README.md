# eShop DevOps Project

A DevOps engineering project built around Microsoft's [dotnet/eShop](https://github.com/dotnet/eShop) reference application, demonstrating containerization, orchestration, CI/CD automation, infrastructure as code, monitoring, and security practices for a real-world microservices system.

This project was built as a graduation project to demonstrate production-grade software delivery practices used by real DevOps and cloud engineering teams.

## Overview

This project takes Microsoft's eShop reference microservices application and wraps it with a full DevOps delivery pipeline. The core services included in scope are:

- **Catalog.API** — product catalog management
- **Basket.API** — shopping cart, backed by Redis
- **Ordering.API** — order placement and tracking

Bonus services included where time allows: Identity.API (authentication), WebApp (storefront UI), OrderProcessor/PaymentProcessor (event-driven background services).

The application uses PostgreSQL with the pgvector extension (one database per service), Redis (basket cache), and RabbitMQ (event bus for inter-service communication).

## Architecture
*(to be completed — Sprint 3)*

## Tech Stack
*(to be completed)*

## Setup — Running Locally (without Docker)

The application can be run locally using .NET Aspire, which orchestrates all services and their dependencies automatically.

**Prerequisites:** .NET 10 SDK, Docker Desktop (Aspire uses it to run Postgres/Redis/RabbitMQ as containers automatically)

```bash
cd src/eShop.AppHost
dotnet run
```

This opens the Aspire Dashboard, showing every service's status and logs, plus a link to the live storefront.

See `docs/screenshots/` for evidence of a successful local run (Resources view and dependency graph).

## Deployment — Dev & Prod

Containerization work is in progress. Each service is documented individually as it's completed:

- [Catalog.API — Dockerization Guide](docs/dockerization-catalog-api.md)

*(Additional services and full dev/prod environment configuration to be added as Sprint 2 progresses.)*

## CI/CD Pipeline
*(to be completed — Sprint 2)*

## Monitoring
*(to be completed — Sprint 3)*

## Security
*(to be completed — Sprint 3)*

## Disaster Recovery
*(to be completed — Sprint 3)*

## Acknowledgements

This project is built on top of Microsoft's [dotnet/eShop](https://github.com/dotnet/eShop) reference application. Sample catalog data (product names, descriptions, brand names) is fictional, originally generated using GPT-3.5-Turbo, with product images generated using DALL·E 3.