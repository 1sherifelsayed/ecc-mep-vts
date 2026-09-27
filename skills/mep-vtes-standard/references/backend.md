# MEP-VTES-001 - Backend Architecture Standards (Section 4)

Every backend MUST adopt **exactly one** of three approved styles. The choice is driven by domain complexity, not preference, and is recorded in **ADR-001** with explicit reference to the matrix. Mixing styles within one deployable is not permitted.

## Selection matrix (4.1)

| Criterion | Layered (N-Tier) | Vertical Slice | Clean Architecture + DDD |
|-----------|------------------|----------------|--------------------------|
| Typical use | CRUD, forms-over-data, internal tools, simple portals | Feature-rich apps with many independent use cases | Complex, long-lived domains with dense rules |
| Business rule density | Low | Low to medium | High |
| Number of use cases | < 30 | 30 to 200+ | Any; driven by domain complexity |
| Integrations | 0 to 2 simple | Several, feature-local | Many, needing anti-corruption layers |
| Expected lifetime | < 3 years | 3+ years | 5+ years, core business system |
| Team size | 1 to 3 devs | 2 to 8 devs | 4+ devs or multiple teams |
| Change pattern | Rare, localized | Frequent, per feature | Frequent, rule-driven |
| Mandatory ADR | ADR-001 with justification | ADR-001 with justification | ADR-001 + bounded-context map |

Rule of thumb: many business rules that change together, or multiple bounded contexts, means Clean + DDD. Many but largely independent use cases means Vertical Slice. Use Layered only for genuinely simple CRUD.

## Layered (4.2)

Presentation (API) -> Business -> Data. Each layer depends only on the layer below. Business logic MUST NOT live in controllers or the data layer. Repositories are optional, and DbContext may be used directly from Business.

```text
Org.ProjectName.sln
 src/
   Org.ProjectName.Api/        -> Controllers, filters, DI composition, Program.cs
   Org.ProjectName.Business/   -> Services, DTOs, validators, interfaces
   Org.ProjectName.Data/       -> DbContext, EF configurations, migrations, repositories
   Org.ProjectName.Common/     -> Constants, extensions, shared primitives
 tests/
   Org.ProjectName.Business.Tests/
   Org.ProjectName.Api.IntegrationTests/
```

## Vertical Slice (4.3)

Organized by feature/use case. Each slice holds its endpoint, request, handler, validator and response, and owns its data access. Slices MUST NOT call each other's handlers. Shared logic moves to Common or to domain entities. Cross-cutting behaviour (validation, logging, transactions) is implemented once as pipeline behaviours or endpoint filters.

```text
src/Org.ProjectName.Api/
  Features/
    Invoices/
      CreateInvoice/     -> Endpoint.cs, Command.cs, Handler.cs, Validator.cs, Response.cs
      GetInvoiceById/    -> Endpoint.cs, Query.cs, Handler.cs, Response.cs
      InvoiceMappings.cs
  Infrastructure/        -> DbContext, EF configurations, migrations, external clients
  Common/                -> Behaviours (validation, logging), ProblemDetails, auth policies
  Program.cs
tests/
  Org.ProjectName.Api.Tests/            -> mirrors Features/, one test class per slice
  Org.ProjectName.Api.IntegrationTests/
```

## Clean Architecture + DDD (4.4)

Dependencies point inward: Api -> Infrastructure -> Application -> Domain. Domain has **no** references to EF Core, ASP.NET Core or any infrastructure package. Aggregates enforce invariants. Application handlers orchestrate use cases through ports implemented in Infrastructure. Domain events and the **Transactional Outbox** provide cross-aggregate and cross-service consistency. Architecture tests verify dependency rules on every build.

```text
src/
  Org.ProjectName.Domain/          -> Entities, Aggregates, Value Objects, Domain Events,
                                      Domain Services, Specifications. NO framework refs.
  Org.ProjectName.Application/     -> Commands/Queries + Handlers, ports, DTOs, validators, behaviours
  Org.ProjectName.Infrastructure/  -> EF Core, migrations, repositories, Redis, RabbitMQ, Hangfire,
                                      LDAP, SharePoint REST client, external API clients
  Org.ProjectName.Api/             -> Endpoints/Controllers, auth, DI root, Program.cs
  Org.ProjectName.Worker/          -> (optional) Hangfire server / message consumers host
tests/
  Org.ProjectName.Domain.Tests/
  Org.ProjectName.Application.Tests/
  Org.ProjectName.ArchitectureTests/  -> NetArchTest / ArchUnitNET dependency rules
  Org.ProjectName.IntegrationTests/   -> Testcontainers (SQL Server/PostgreSQL/Redis/RabbitMQ)
```

## Mandatory backend standards (4.5)

| Area | Requirement |
|------|-------------|
| API design | RESTful resources; URL versioning `/api/v1/...`; OpenAPI 3.x generated at build; camelCase JSON; errors as **ProblemDetails (RFC 9457)**; server-side pagination, filtering and sorting on every list endpoint. |
| Validation | All input validated server-side (FluentValidation or DataAnnotations). Client-side validation is UX only. |
| Persistence | EF Core code-first, migrations committed. Idempotent SQL per release (`dotnet ef migrations script --idempotent`). **Auto-migration on start is forbidden in UAT/PROD.** Optimistic concurrency tokens on concurrently edited aggregates. |
| Configuration | appsettings.json + env overrides + env vars. Options pattern with `ValidateDataAnnotations().ValidateOnStart()`. No secrets, connection strings or keys in Git. |
| Health | `/health/live` and `/health/ready` (AspNetCore.HealthChecks.*) covering DB, Redis, RabbitMQ. |
| Resilience | `Microsoft.Extensions.Http.Resilience` (retry with jitter, timeout, circuit breaker) on every outbound HTTP call. Explicit timeouts on DB, Redis, LDAP, SharePoint and broker calls. |
| Date & time | Persist UTC (DateTimeOffset or UTC DateTime). Convert to Asia/Riyadh only at the presentation edge. Hijri display where the business requires it. Use `TimeProvider`. |
| Localization | ar-SA and en-US resources for all user-facing messages, including validation and errors. |
| Solution hygiene | `Directory.Build.props` (Nullable=enable, ImplicitUsings, TreatWarningsAsErrors in Release), `.editorconfig`, `Directory.Packages.props` (CPM), `global.json` pinning the SDK. |
| Architecture tests | Mandatory for Clean and Vertical Slice: tests that fail the build on forbidden dependencies (Domain -> Infrastructure, slice -> another slice's internals). |

## Messaging, jobs, caching (4.6)

| Component | Requirement |
|-----------|-------------|
| RabbitMQ | Durable exchanges/queues; publisher confirms; dead-letter exchange per consumer queue; idempotent consumers (message-id de-dup); versioned contracts in a dedicated Contracts project; Transactional Outbox wherever a DB change and a publish must be consistent. |
| Hangfire | Jobs idempotent and re-entrant; `AutomaticRetry` explicit per job; stable, documented recurring job IDs; dashboard behind an authorization filter (never anonymous in any environment); heavy work in a separate Worker host, not the API process. |
| Redis | Keys `{app}:{context}:{entity}:{id}`; every key has a TTL; cache-aside with explicit invalidation; app MUST stay functionally correct (just slower) if Redis is down. |
