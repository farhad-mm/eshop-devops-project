# Dockerization Guide

This document covers the Dockerization process for each containerized service in this project, in the order they were completed. Each service has its own section below, following the same structure: prerequisites, the correct step-by-step procedure, and a troubleshooting log of real issues encountered.

---

## Catalog.API

### Prerequisites
- Docker Desktop installed and running
- Repository cloned, with the `dev` branch checked out

### 1. Understand the project's dependency structure

Before writing a Dockerfile for any .NET microservice in this repo, inspect its `.csproj` file:

```bash
cat src/Catalog.API/Catalog.API.csproj
```

Look for two things:
- **`<ProjectReference>`** entries — other projects this service depends on and must be compiled alongside it
- **`<Compile Include="..\...">`** entries — individual shared source files pulled in from sibling folders

For Catalog.API, this revealed dependencies on:
- `EventBusRabbitMQ` (which itself depends on `EventBus`)
- `IntegrationEventLogEF`
- `eShop.ServiceDefaults`
- `Shared` (individual files, not a project reference)

**Why this matters:** Docker's build context must include every folder these references point to, or the build will fail. Skipping this step is the single most common cause of failed .NET Docker builds in a multi-project repository like this one.

### 2. Identify the repo's build configuration files

This repo uses **NuGet Central Package Management**, meaning package versions are defined once, centrally, rather than per-project. Confirm this by checking the actual repo root (not the nested `src/` folder containing the projects):

```bash
find . -maxdepth 1 -name "Directory.*.props"
```

This should show `Directory.Build.props` and `Directory.Packages.props`. **Both must be included in the Docker build context** — without them, `dotnet restore` cannot resolve package versions.

### 3. Set the correct build context

Because of the dependencies found above, the Docker build context must be the **repository root** (`src/`, the folder containing both the `.props` files and the nested `src/` folder with all service projects) — not the individual service's folder.

### 4. Write the Dockerfile

Create `src/Catalog.API/Dockerfile`:

```dockerfile
# Build stage: uses the full SDK to compile the app
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# Copy central config files first (needed for package version resolution)
COPY ["Directory.Build.props", "."]
COPY ["Directory.Packages.props", "."]

# Copy only .csproj files first, to leverage Docker layer caching for restore
COPY ["src/Catalog.API/Catalog.API.csproj", "src/Catalog.API/"]
COPY ["src/EventBus/EventBus.csproj", "src/EventBus/"]
COPY ["src/EventBusRabbitMQ/EventBusRabbitMQ.csproj", "src/EventBusRabbitMQ/"]
COPY ["src/IntegrationEventLogEF/IntegrationEventLogEF.csproj", "src/IntegrationEventLogEF/"]
COPY ["src/eShop.ServiceDefaults/eShop.ServiceDefaults.csproj", "src/eShop.ServiceDefaults/"]

RUN dotnet restore "src/Catalog.API/Catalog.API.csproj"

# Copy everything else and publish
COPY . .
WORKDIR /src/src/Catalog.API
RUN dotnet publish "Catalog.API.csproj" -c Release -o /app/publish

# Final stage: lightweight runtime image only, no SDK/build tools
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS final
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENTRYPOINT ["dotnet", "Catalog.API.dll"]
```

**Design decisions:**
- **Multi-stage build**: keeps the final image small and free of build tooling (smaller attack surface, faster deploys).
- **`.csproj` files copied before full source**: leverages Docker's layer caching so `dotnet restore` (slow) is skipped on rebuilds unless dependencies actually change.
- **Port 8080**: Microsoft's official ASP.NET container images default to this port.

### 5. Build the image

Run from the repository root (`src/`):

```bash
docker build -t eshop-catalog-api:local -f src/Catalog.API/Dockerfile .
```

### 6. Run with its dependencies via Docker Compose

Catalog.API requires PostgreSQL and RabbitMQ to function fully. Use Docker Compose to run the service with its dependencies together (see `docker-compose.yml` at the repo root).

```bash
docker compose up --build -d
```

### 7. Verify

```bash
curl "http://localhost:8080/api/catalog/items?api-version=1.0&pageSize=5"
```

A successful response returns JSON with real seeded product data, confirming the container, database connection, EF Core migrations, and data seeding all work correctly.

**Note the `api-version` query parameter is required** — this API uses explicit API versioning (via `Asp.Versioning.Http`), so requests without a version return HTTP 400.

### RabbitMQ (Event Bus)

Catalog.API publishes and subscribes to integration events via RabbitMQ, using the connection name `eventbus` (confirmed in `src/Catalog.API/Extensions/Extensions.cs`). This is provided in `docker-compose.yml` via:

```yaml
rabbitmq:
  image: rabbitmq:4-management
  environment:
    RABBITMQ_DEFAULT_USER: guest
    RABBITMQ_DEFAULT_PASS: guest
  ports:
    - "5672:5672"
    - "15672:15672"
```

The `catalog-api` service connects to it via `ConnectionStrings__EventBus: "amqp://guest:guest@rabbitmq:5672"`. The `-management` image variant also exposes a web dashboard at `http://localhost:15672` (login `guest`/`guest`), useful for confirming messages are actually flowing.

### Troubleshooting Log

Issues actually encountered while building this, in the order they occurred — kept here for reference, since a real DevOps workflow rarely goes perfectly on the first attempt.

#### Issue 1: `NU1015` — package versions not specified
**Symptom:** `dotnet restore` failed with errors like "The following PackageReference item(s) do not have a version specified."
**Cause:** The repo uses NuGet Central Package Management (`Directory.Packages.props`), which wasn't yet included in the Docker build context.
**Fix:** Added explicit `COPY` instructions for `Directory.Build.props` and `Directory.Packages.props`, and moved the build context from `src/src` to the true repository root (`src/`).

#### Issue 2: Missing `EventBus` project
**Symptom:** `Skipping project ".../EventBus/EventBus.csproj" because it was not found.`
**Cause:** `EventBusRabbitMQ.csproj` has its own `<ProjectReference>` to `EventBus`, which wasn't copied into the build context.
**Fix:** Added a `COPY` line for `EventBus/EventBus.csproj` alongside the other dependency projects.

#### Issue 3: Build context path mismatch after fixing Issues 1 & 2
**Symptom:** All `COPY` instructions failed with "not found" errors, despite the paths being correct in the Dockerfile.
**Cause:** The `docker build` command was still being run from `src/src`, while the Dockerfile had been updated to expect a build context of `src/` (the actual repo root).
**Fix:** Ran the build command from the repository root instead, with an updated `-f` flag pointing to the Dockerfile's new relative location: `docker build -t eshop-catalog-api:local -f src/Catalog.API/Dockerfile .`

#### Issue 4: Missing database connection string (expected, standalone test)
**Symptom:** `ConnectionString is missing. It should be provided in 'ConnectionStrings:catalogdb'...`
**Cause:** Ran the container standalone (`docker run`) without a database, to confirm the container itself starts correctly.
**Resolution:** Not a bug — confirmed the container works correctly in isolation. Moved to Docker Compose to provide the required database dependency.

#### Issue 5: `.gitignore` accidentally overwritten
**Symptom:** New ignore rules were added, but the repository's original, more complete `.gitignore` content disappeared.
**Cause:** Pasting new content into an already-open file in an editor replaced the existing text instead of appending to it.
**Fix:** Restored a complete, standard .NET `.gitignore` covering build output, IDE files, test results, and secrets, then verified with `git status` before committing.

---

## Basket.API

### 1. Dependency inspection

`Basket.API.csproj` was inspected the same way as Catalog.API. It references `eShop.ServiceDefaults` and `EventBusRabbitMQ` (which transitively needs `EventBus`), but — unlike Catalog.API — has no dependency on `IntegrationEventLogEF` or `Shared`, since it uses Redis rather than Entity Framework/Postgres.

Two dependencies not seen in Catalog.API:
- **`Aspire.StackExchange.Redis`** — the Redis client
- **`Grpc.AspNetCore`** with a `.proto` file (`GrpcServices="Server"`) — Basket.API exposes a gRPC service in addition to any HTTP endpoints

### 2. Confirming the Redis connection name

```bash
grep -rn "AddRedis" src/Basket.API
# -> builder.AddRedisClient("redis");
```

This confirmed the connection string environment variable needed is `ConnectionStrings__redis`.

### 3. Dockerfile

Same proven multi-stage pattern as Catalog.API, with dependency paths adjusted:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

COPY ["Directory.Build.props", "."]
COPY ["Directory.Packages.props", "."]

COPY ["src/Basket.API/Basket.API.csproj", "src/Basket.API/"]
COPY ["src/EventBus/EventBus.csproj", "src/EventBus/"]
COPY ["src/EventBusRabbitMQ/EventBusRabbitMQ.csproj", "src/EventBusRabbitMQ/"]
COPY ["src/eShop.ServiceDefaults/eShop.ServiceDefaults.csproj", "src/eShop.ServiceDefaults/"]

RUN dotnet restore "src/Basket.API/Basket.API.csproj"

COPY . .
WORKDIR /src/src/Basket.API
RUN dotnet publish "Basket.API.csproj" -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS final
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENTRYPOINT ["dotnet", "Basket.API.dll"]
```

Built successfully on the first attempt, with no rework needed — confirming that checking dependencies and the Redis connection name before writing the Dockerfile (rather than discovering them via failed builds, as happened with Catalog.API) avoids the earlier round of trial and error entirely.

### 4. Docker Compose integration

Added to `docker-compose.yml`: a `redis:7` service (no configuration required out of the box), and a `basket-api` service depending on both `redis` and `rabbitmq`, with:

```yaml
ConnectionStrings__redis: "redis:6379"
ConnectionStrings__EventBus: "amqp://guest:guest@rabbitmq:5672"
```

Basket.API is mapped to host port `8081` (rather than `8080`) since both it and Catalog.API listen on `8080` inside their own containers and cannot both claim the same host port simultaneously.

### 5. Verification

Basket.API exposes a gRPC-only endpoint, so a plain `curl` request correctly receives an HTTP 400 with the message "An HTTP/1.x request was sent to an HTTP/2 only endpoint" — this confirms the server is running and correctly enforcing gRPC's HTTP/2 requirement, rather than indicating a failure. Full gRPC-level testing (e.g., via `grpcurl`) is deferred to the testing pillar of this project.

Startup logs confirmed a clean connection to RabbitMQ with no errors, matching the pattern proven with Catalog.API.

### Notes

Two non-fatal startup warnings appear in the logs, both expected for a local/dev container and worth revisiting under the project's Security (DevSecOps) pillar rather than fixing now:
- **DataProtection keys** stored inside the container's filesystem are lost on container recreation — irrelevant for local testing, but relevant for persistent auth scenarios in production.