---
mode: agent
model: GPT-4o
tools: ['codebase','terminal','tests','githubIssues','githubPullRequests']
description: 'K8s-first .NET microservice scaffold with ASP.NET Core, Kafka (Confluent for Kubernetes), EF Core (PostgreSQL/SQL Server), testing, logging, security, Helm charts, and FluxCD GitOps overlays.'
---

# .NET Microservice (Kubernetes-first) — Production-Ready Scaffold

> You are an expert .NET architect and senior ASP.NET Core developer.  
> You will **plan, scaffold, and iteratively implement** a production-grade microservice with HTTP APIs, Kafka producer/consumer components, and database access for **PostgreSQL and/or SQL Server**, including comprehensive **testing, logging, security, observability, Helm packaging, and FluxCD GitOps integration**.  
> Target runtime: **.NET 9** (fallback **.NET 8 LTS**).

---

## Inputs (ask or infer before starting)
Collect these up-front (ask concise questions only if missing), then **restate a plan**:

- **ServiceName**: e.g., `Orders`, `Payments`
- **API style**: `Minimal APIs` (default) or `Controllers`
- **Database(s)**: `PostgreSQL`, `SqlServer`, or `Both` (default `PostgreSQL`)
- **Messaging**: `Kafka` (default enabled) – topics, partitions, key format, headers
- **Auth**: `JWT` (default) or `OIDC` (issuer authority URL, audience, scopes/roles)
- **Observability**: `OpenTelemetry + HealthChecks` (default)
- **Packaging**: `Dockerfile` (default) and optional `docker-compose` for local infra
- **GitOps**: FluxCD with Helm (default); choose **HelmRepository** or **OCIRepository**
- **Kubernetes**: namespace, resource requests/limits, liveness/readiness probes
- **Traffic**: ClusterIP service (default); optionally Gateway API/Ingress, PodDisruptionBudget
- **Policies**: rate limiting, request validation, versioning, CORS

---

## Deliverables

### A. Solution & Projects (Clean Architecture)
```
/src
  ServiceName.Api                 # ASP.NET Core entrypoint (MinimalAPIs/Controllers)
  ServiceName.Application         # Use cases, DTOs, validators (FluentValidation)
  ServiceName.Domain              # Entities, value objects, domain events
  ServiceName.Infrastructure      # EF Core, repositories, migrations, DB connectivity
  ServiceName.Messaging           # Kafka producer/consumer, background services
  ServiceName.Shared              # Contracts, abstractions (optional)
/tests
  ServiceName.UnitTests           # xUnit + FluentAssertions
  ServiceName.IntegrationTests    # WebApplicationFactory + Testcontainers (Kafka, DB)
/deploy
  helm/ServiceName/               # Helm chart for the microservice
  flux/overlays/{dev,qa,prod}/    # Flux Kustomize overlays & HelmRelease YAML
  cfk/                            # Confluent for Kubernetes (operator/CRs) optional
```

### B. HTTP API
- Endpoints: RESTful CRUD sample; **API versioning**; **OpenAPI/Swagger** (grouped per version); examples.
- Validation: **FluentValidation** + automatic **ProblemDetails** responses; request size limits.
- Resilience: **Polly** for outbound calls (optional); consistent exception-to-ProblemDetails middleware.
- Health: `/health/live` (no deps) and `/health/ready` (DB & Kafka checks).
- Cross-cutting: correlation IDs, request logging, **rate limiting**, and **CORS**.

### C. Kafka Messaging (Producer & Consumer)
- **Confluent.Kafka** client with **keyed, JSON** messages.
- Producer: acks configuration, idempotence where appropriate, batch/linger tuning.
- Consumer: runs in **IHostedService** with **manual commit** post-success; **exponential backoff** retries; **DLQ** topic support; graceful shutdown that drains in-flight work.
- Contracts: versioned DTOs; optional Schema Registry integration later.
- Consistency: optional **outbox/inbox** strategy.

### D. Data Access (PostgreSQL & SQL Server)
- **EF Core** DbContext per service; provider-specific configuration.
- **Connection resiliency** (`EnableRetryOnFailure`) and optimistic concurrency.
- **Migrations** per provider, generated into the repo; **read models** (optional Dapper) for hot paths.
- Configuration via **IOptions**; credentials loaded from env/K8s Secret; **no secrets in Git**.

### E. Security
- **Authentication**: `AddAuthentication().AddJwtBearer(...)` with metadata discovery.
- **Authorization**: policy-based roles/scopes; endpoint-level requirements.
- **Headers**: HSTS (prod), X-Content-Type-Options, X-Frame-Options, Referrer-Policy, HTTPS-only.
- **Validation**: strict input validation; reject overlong payloads.
- **Threat modeling** checklist: injection, deserialization, SSRF, path traversal, secret leakage.

### F. Logging & Observability
- **Serilog** (structured JSON to stdout).
- **OpenTelemetry**: traces (ASP.NET, HttpClient, EF Core; custom ActivitySource for Kafka), metrics (request latency, error rate, consumer lag), logs; **OTLP** export.
- **HealthChecks** wired to Kubernetes probes and `/health` endpoints.

### G. Testing
- **Unit**: xUnit + FluentAssertions + Moq/NSubstitute.
- **Integration**: **Testcontainers** for Kafka + Postgres/SQL Server; API via `WebApplicationFactory<Program>`.
- Seed test data and run EF migrations on start; assert producer emits and consumer commits.
- **Contract tests** (optional) for Kafka payloads & HTTP APIs.

### H. Kubernetes-first (Helm + FluxCD)
Generate:
- **Helm chart**: `Chart.yaml`, `values.yaml`, templates for Deployment, Service, ServiceAccount, Role/RoleBinding (if needed), ConfigMap, Secret, HPA, PDB, probes, resource limits, optional Gateway/Ingress.
- **Flux overlays** (dev/qa/prod): `HelmRelease` referencing a `HelmRepository` or `OCIRepository`; values overrides; Kustomization entry that applies namespace + release.
- **Secrets handling**: mount from `Secret` (default). Provide hooks for External Secrets Operator if present (optional).
- **Config**: image repo/tag, env vars, connection strings, Kafka bootstrap, auth parameters.

### I. Confluent for Kubernetes (CFK) Option
- **Install** CFK via Helm (operator + CRDs) and (optionally) Confluent Platform components for **KRaft** mode.
- Provide **CR examples**: `Kafka` (cluster), `KafkaTopic`, `KafkaUser` (for SASL/SCRAM or mTLS), and network policies.
- Generate **Flux** definitions to manage CFK installation (HelmRelease) and Confluent resources declaratively.
- Surface **bootstrap** addresses and credentials to the app via `Secret` + `ConfigMap` → env vars.

---

## Architectural & Coding Standards
- Enforce **Clean Architecture** boundaries.
- **Async** with **CancellationToken**; avoid blocking calls.
- **Options pattern** for configuration; only compose in `Program.cs`/composition root.
- Enable **nullable reference types**, `WarningsAsErrors` where feasible, and analyzers (`Microsoft.CodeAnalysis.NetAnalyzers`).
- Prefer **CQRS-style** application services when helpful.
- **Idempotency** for consumers (dedupe key or outbox pattern) with **at-least-once** semantics.

---

## Scaffolding Commands (create a new solution)
```bash
dotnet new sln -n ServiceName
dotnet new webapi        -n ServiceName.Api             --framework net9.0 --use-minimal-apis
dotnet new classlib      -n ServiceName.Application     --framework net9.0
dotnet new classlib      -n ServiceName.Domain          --framework net9.0
dotnet new classlib      -n ServiceName.Infrastructure  --framework net9.0
dotnet new classlib      -n ServiceName.Messaging       --framework net9.0
dotnet new xunit         -n ServiceName.UnitTests
dotnet new xunit         -n ServiceName.IntegrationTests
dotnet sln add **/*.csproj
```

### NuGet packages (add only what you need)
- **API/Validation**: `Swashbuckle.AspNetCore`, `FluentValidation`, `FluentValidation.DependencyInjectionExtensions`, `Asp.Versioning.Http`
- **EF Core**: `Microsoft.EntityFrameworkCore`, `Npgsql.EntityFrameworkCore.PostgreSQL`, `Microsoft.EntityFrameworkCore.SqlServer`, `Microsoft.EntityFrameworkCore.Design`, `EFCore.NamingConventions`
- **Kafka**: `Confluent.Kafka` (optionally `Confluent.SchemaRegistry`)
- **Logging/Obs**: `Serilog.AspNetCore`, `Serilog.Sinks.Console`, `OpenTelemetry.Extensions.Hosting`, `OpenTelemetry.Instrumentation.AspNetCore`, `OpenTelemetry.Instrumentation.Http`, `OpenTelemetry.Exporter.Otlp`
- **Security**: `Microsoft.AspNetCore.Authentication.JwtBearer`
- **Testing**: `FluentAssertions`, `Moq` (or `NSubstitute`), `Microsoft.AspNetCore.Mvc.Testing`, `Testcontainers`, `Testcontainers.Kafka`, `Testcontainers.PostgreSql`, `Testcontainers.MsSql`, `Respawn` (optional)

---

## Web & API Layer — Implementation Notes
- Wire **API versioning**; expose **Swagger** per version with examples.
- Add **global exception handling** → ProblemDetails; **validation** via FluentValidation pipeline.
- **Rate limiting** (`AddRateLimiter` fixed-window/token-bucket) and **CORS**.
- Health endpoints: `/health/live` (always green) and `/health/ready` (DB/Kafka checks).

---

## Kafka — Implementation Notes
- Producer wrapper with `IProducer<TKey, TValue>` and JSON serialization; configure `acks=all`, enable idempotence when desired.
- Consumer as `BackgroundService`: disable auto-commit; on success → `Commit`; on transient errors → **retry with jittered exponential backoff**; on poison → publish to **DLQ** with headers (`error`, `stacktrace`, `correlationId`).
- Use **CancellationToken** for graceful stop; seek/commit logic guarded.
- **Contract versioning** (suffix topic with version or embed version in payload/header).

---

## EF Core — Implementation Notes
- One `DbContext` per service; separate Provider configurations for Postgres vs SQL Server.
- `EnableRetryOnFailure`, command timeout, and sensible pool size.
- Migrations per provider: e.g., `Migrations/Postgres` and `Migrations/SqlServer` folders.
- Keep Domain persistence-agnostic; Infrastructure contains EF entity configs and repositories.

---

## Security — Implementation Notes
- `AddAuthentication().AddJwtBearer` with authority & audience; handle clock skew.
- Policy-based authorization with scope/role requirements.
- Add standard **security headers**; enforce HTTPS redirection in prod.
- Secrets via env or mounted K8s Secret; **never** commit secrets.

---

## Observability — Implementation Notes
- **Serilog** JSON to stdout; include correlation/trace ids, user, request id, business keys.
- **OpenTelemetry**: set up `ResourceBuilder` with service name/version; instrument ASP.NET, HttpClient, EF Core; create **ActivitySource** for Kafka operations; export OTLP.
- Health checks wired to probes; ready checks include DB connectivity and Kafka broker metadata.

---

## Testing Strategy (in detail)
- **Unit**: cover application services, validators, domain logic.
- **Integration**:
  - Start **Testcontainers** for Postgres/SQL Server & Kafka.
  - Use `WebApplicationFactory<Program>` to test API endpoints with in-memory host.
  - Run EF migrations at test startup; seed data.
  - Verify **producer** emits expected records; **consumer** processes and **commits** offsets; validate DLQ behavior for poison messages.
- **Contract tests** (optional): ensure Kafka and HTTP DTOs remain compatible.

---

## Kubernetes (Helm) Artifacts to Generate

### Chart structure
```
/deploy/helm/ServiceName/
  Chart.yaml
  values.yaml
  templates/
    deployment.yaml
    service.yaml
    hpa.yaml
    pdb.yaml
    serviceaccount.yaml
    configmap.yaml
    secret.yaml                # ref existing Secret; do not inline secrets
    gateway.yaml               # optional Gateway/HTTPRoute (Gateway API) or Ingress
    networkpolicy.yaml         # optional
```

### values.yaml (keys to support)
```yaml
image:
  repository: <REQUIRED>        # e.g., ghcr.io/org/service
  tag: <REQUIRED>
  pullPolicy: IfNotPresent

resources:
  requests: { cpu: "100m", memory: "256Mi" }
  limits:   { cpu: "500m", memory: "512Mi" }

env:
  ASPNETCORE_URLS: http://+:8080
  DOTNET_ENVIRONMENT: Production

config:
  dbProvider: postgresql        # or sqlserver
  connectionStringSecretRef: { name: service-db, key: connectionString }
  kafka:
    bootstrapServers: kafka:9092
    securityProtocol: PLAINTEXT # TLS/SASL options configurable
    topicPrefix: service

service:
  type: ClusterIP
  port: 8080

otel:
  enabled: true
  otlpEndpoint: http://otel-collector:4317

ingress:
  enabled: false
gatewayApi:
  enabled: false
hpa:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 75
```

---

## FluxCD GitOps Artifacts to Generate

### Option 1: HelmRepository
```
/deploy/flux/overlays/dev/
  namespace.yaml
  helmrepository.yaml
  helmrelease.yaml
  kustomization.yaml
```
- `HelmRepository`: points at your chart registry (e.g., Git/Helm repo).
- `HelmRelease`: references the chart `ServiceName`, sets values overrides (image tag, env, secrets, resource limits).

### Option 2: OCIRepository
```
/deploy/flux/overlays/dev/
  namespace.yaml
  ocirepository.yaml
  helmrelease.yaml
  kustomization.yaml
```
- `OCIRepository`: points at container registry (e.g., ACR/GHCR) where the **Helm chart** is pushed as **OCI artifact**.
- `HelmRelease`: consumes chart from the `OCIRepository` and applies `values` overlays per environment.

> Provide overlays for `dev`, `qa`, `prod` (separate image tags, resource sizing, endpoints).

---

## Confluent for Kubernetes (CFK) — GitOps Pattern

### CFK install via Flux + Helm
Generate Flux `HelmRepository` (Confluent Helm repo) and a `HelmRelease` to install:
- **CFK operator** (CRDs/controllers)
- (Optional) **Confluent Platform** components (Kafka in **KRaft**)

### Example CFK CRs (managed by Flux Kustomization)
- `Kafka` cluster (replicas, storage class, listeners, TLS/SASL conf)
- `KafkaTopic` resources for each topic the service uses (with partitions/replication)
- `KafkaUser` with credentials and ACLs (SASL/SCRAM or mTLS)
- Export **bootstrap servers** and **secrets** to the application namespace via `Secret` references.

> The microservice reads bootstrap servers and credentials from env vars backed by K8s `Secret` references in the Helm chart `values.yaml`.

---

## Example App Configuration Snippets to Generate

### `Program.cs` (high level)
- Add **Serilog** and **OpenTelemetry**.
- `AddAuthentication().AddJwtBearer` and policy-based authorization.
- `MapGroup("/api/v{version}")` with versioning + Swagger setup.
- Register **DbContext** (Postgres/SQL Server) with pooled factory and retries.
- Register **Kafka** producer + consumer as DI services with hosted consumers.
- Health checks: `AddCheck("self", ...)`, `AddNpgSql/MsSql`, custom `KafkaHealthCheck`.

### `appsettings.json`
- Connection strings via env (placeholders).
- Kafka config (bootstrap servers, SASL/TLS switch).
- Serilog minimum levels, enrichment properties.
- OTLP exporter endpoint.

### `Dockerfile`
- Multi-stage build; publish trimmed self-contained image (optional); run as non-root user.

### `docker-compose.yml` (local dev)
- Service + Postgres + Kafka (KRaft) + optional Schema Registry & OTEL collector.

---

## Step-by-Step Plan (how you will work)
1. Confirm **Inputs**, propose a short plan, await confirmation.
2. Scaffold **solution & projects**; add NuGet packages; wire DI and configuration.
3. Implement **API** (health + sample CRUD) with validation, OpenAPI, versioning.
4. Add **EF Core** (Postgres/SQL Server), migrations, repositories, unit tests.
5. Implement **Kafka** producer + consumer; end-to-end test with Testcontainers.
6. Add **Serilog** + **OpenTelemetry**; validate traces/metrics/logs locally.
7. Add **security** (JWT/OIDC) + policies; validate 401/403 paths.
8. Create **Helm chart** and **Flux overlays** (dev/qa/prod).
9. (Optional) Install **CFK** via Flux, declare Kafka cluster/topics/users.
10. Produce a **README.md** with run/test/build/deploy steps and env var tables.

---

## Acceptance Criteria
- Solution builds and tests (`dotnet build`, `dotnet test`) pass.
- API: Swagger UI available; validation errors return **ProblemDetails**; health endpoints green.
- DB: migrations apply cleanly; CRUD works against chosen provider(s).
- Kafka: producer sends; consumer processes with retries and **manual commits**; DLQ path testable.
- Logs: structured JSON with correlation IDs; traces/metrics exported when collector available.
- Security: endpoints protected per policy; unauthorized/forbidden paths behave correctly.
- Kubernetes: Helm chart renders (`helm template`) with sensible defaults; Flux `HelmRelease` applies cleanly with environment overrides.
- GitOps: app versioning via image tag; config via values; secrets via K8s `Secret`; no secrets in Git.
- CFK: Kafka topics/users declared via CRs; app picks up bootstrap/creds from `Secret`/env.

---

## Guardrails
- Prefer latest stable frameworks compatible with chosen .NET version.
- No deprecated APIs; use **async I/O** with cancellation where sensible.
- Keep endpoints thin; business logic lives in Application; persistence in Infrastructure; messaging in Messaging.
- Comment non-obvious code; document public API via XML docs/OpenAPI examples.
- Keep PR-sized changes small; commit with clear messages when evolving an existing repo.

---

## (Optional) Files/Boilerplates to Emit on Request
- `README.md` with run/test/deploy instructions.
- `Helm` chart boilerplate and default `values.yaml`.
- `Flux` overlays (`HelmRepository`/`OCIRepository`, `HelmRelease`, `Kustomization`, `Namespace`).
- `CFK` CR examples for Kafka cluster/topic/user.
- `docker-compose.yml` for local dev (Postgres + Kafka + app).
- Sample `Program.cs`, `DbContext`, `KafkaProducerService`, `KafkaConsumerService`.