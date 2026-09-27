# MEP-VTES-001 - Cross-Cutting Mandatory Standards (Section 6)

## 6.1 Security baseline

- OWASP ASVS (current) **Level 2**. Mitigate OWASP Top 10 (Web) and OWASP Mobile Top 10.
- NCA Essential Cybersecurity Controls (ECC) and the Personal Data Protection Law (PDPL) wherever personal data is processed. All data, backups and logs remain in the Kingdom of Saudi Arabia.
- TLS 1.2+ for all traffic, including internal service-to-service, DB, Redis, RabbitMQ, SharePoint and LDAP.
- Parameterized queries only (EF Core / Dapper parameters). **String-concatenated SQL is a Critical defect.**
- Security headers: HSTS, Content-Security-Policy, X-Content-Type-Options, Referrer-Policy, frame-ancestors. Server banner headers removed.
- Secrets via Kubernetes Secrets, ACL-protected IIS environment variables, or the org secret store. Never in Git, images, pipeline logs or documents.
- Authorization is policy-based, enforced server-side on every endpoint, and denies by default (`[Authorize]` globally, `[AllowAnonymous]` only where approved).
- Audit trail for all create/update/delete on business data and all security events (login, logout, failed login, permission change).

## 6.2 JWT token standards

| Aspect | Requirement |
|--------|-------------|
| Signing | RS256 or ES256 preferred. HS256 only for a single self-contained service with a key of 256+ bits from the secret store. `none` rejected. |
| Access token lifetime | 15 minutes or less. |
| Refresh token | Opaque, random, stored **hashed** server-side, **rotated on every use** with reuse detection (revoke the whole family on reuse). Lifetime per NFR, max 8 h for internal apps unless approved. |
| Validation | ValidateIssuer, ValidateAudience, ValidateLifetime, ValidateIssuerSigningKey all true. ClockSkew of 30 s or less. |
| Claims | Minimal: sub, name, roles/permissions, tenant. No passwords, national ID / Iqama numbers, salaries or other sensitive data. |
| Key rotation | `kid` header. Keys rotatable without downtime. Procedure documented. |
| Revocation | Logout and admin disable revoke refresh tokens immediately. Documented access-token revocation strategy (short lifetime or Redis deny-list). |

## 6.3 Active Directory over LDAP (when needed)

- LDAPS (636) or LDAP + StartTLS only. **Plain simple bind on 389 is forbidden.**
- Use `System.DirectoryServices.Protocols` (cross-platform). `System.DirectoryServices` (Windows-only) is not accepted for containerized services.
- A dedicated, read-only, least-privilege service account for searches, with credentials from the secret store.
- Verify user credentials by binding as the user. AD passwords are never stored, logged or cached.
- AD group to role mapping is configuration-driven and documented.
- LDAP filters built with escaped input (RFC 4515) to prevent injection.
- Connection and search timeouts set. Respect the lockout policy (no retry storms on failed binds).
- After AD authentication, the app issues its own JWT per 6.2.

## 6.4 Logging in CLEF

- Serilog with `CompactJsonFormatter`. Containers log to stdout. IIS writes rolling CLEF files **outside** the site folder, with retention. The ASP.NET Core Module stdout log MUST be disabled in production.
- Mandatory properties: `@t, @mt, @l, @x` (errors), `Application, Environment, CorrelationId, TraceId, UserId` (pseudonymous), `RequestPath` and `ElapsedMs` for request logs.
- Message templates with named properties. Never use string interpolation in log calls.
- Levels: Information for business events, Warning for recoverable anomalies, Error for failed operations, Fatal for host termination. Debug/Verbose off in UAT/PROD by default and switchable by config.
- MUST NOT log: passwords, tokens, API keys, connection strings, national ID / Iqama, full names with contact details, payment data, document contents, or full request/response bodies containing personal data.
- CorrelationId propagated across HTTP calls, RabbitMQ message headers and Hangfire jobs.

```json
{"@t":"2026-09-27T08:15:32.418Z","@mt":"Invoice {InvoiceId} approved by {UserId}","@l":"Information","InvoiceId":"INV-2026-00417","UserId":"u-5f2c","Application":"Org.Finance.Api","Environment":"UAT","CorrelationId":"0HN6T1R4B2Q9P","TraceId":"4bf92f3577b34da6a3ce929d0e0e4736","RequestPath":"/api/v1/invoices/417/approve","ElapsedMs":42.7}
```

## 6.5 Data management

- Naming fixed per engine: PostgreSQL snake_case, SQL Server PascalCase. Documented in the data model.
- Audit columns on every business table: `created_at, created_by, updated_at, updated_by` (UTC).
- Soft delete (`is_deleted` + global query filter) where the business requires retention.
- Every schema change is an EF Core migration + idempotent SQL script. No manual changes in any environment.
- Reference/seed data as versioned scripts or `HasData` migrations.
- Indexes justified by query patterns. FKs enforced. No nullable columns without business justification.
- Test/UAT use anonymized or synthetic data. Production data is never copied to vendor machines.
- Backup, restore and retention (RPO/RTO) documented, and restore tested before go-live.

## 6.6 Source control, branching, repository layout

- Trunk-based with short-lived branches (preferred) or GitFlow-lite (`main, develop, release/*, hotfix/*`). The choice is recorded in an ADR.
- Branch names: `feature/<workItemId>-short-name`, `bugfix/<workItemId>-short-name`, `hotfix/<version>`.
- Conventional Commits referencing the work item, e.g. `feat(invoices): add approval AB#1234`.
- Branch policies on main (and develop): PR required, at least one **org** reviewer for main, build validation, linked work item, all comments resolved, no direct or force pushes.
- Releases tagged SemVer `vMAJOR.MINOR.PATCH`.
- `.gitignore` and `.gitattributes` present. No binaries, build outputs, secrets or production data.
- Full history preserved. A single squashed "initial commit" at handover is rejected.

Mandatory repository layout:

```text
<repository-root>/
  README.md            -> purpose, prerequisites, local setup (30 min or less), run, test
  CONTRIBUTING.md      -> branching, commit, PR, coding conventions
  CHANGELOG.md         -> per release (Keep a Changelog)
  global.json | Directory.Build.props | Directory.Packages.props | .editorconfig
  src/                 -> backend, frontend, mobile as separate folders/repos
  tests/               -> unit, integration, architecture, e2e
  docs/
    adr/               -> ADR-001-architecture-style.md, ADR-002-... (MADR template)
    architecture/      -> C4 diagrams (.drawio / Structurizr / Mermaid source)
    api/               -> openapi.json (exported per release)
    runbooks/          -> operations runbook, incident playbooks
    ai-usage.md        -> AI Usage Declaration (6.10)
  pipelines/           -> azure-pipelines-ci.yml, azure-pipelines-cd.yml, templates/
  deploy/
    docker/            -> Dockerfile(s), .dockerignore
    k8s/ or helm/      -> manifests/charts with values-dev|sit|uat|prod.yaml
    iis/               -> web.config templates, deployment scripts
    sharepoint/        -> provisioning scripts, permission matrix
  db/                  -> idempotent migration scripts per release, seed data
```

## 6.7 CI/CD with Azure Pipelines

- CI on every PR and merge: restore, build, lint, unit tests, architecture tests, coverage gate, **SAST**, dependency (**SCA**) scan, **secret scan**, container image build and **image scan** (e.g. Trivy). Plus a CycloneDX SBOM per release (3.2).
- CD: the **same immutable artifact/image** goes through DEV, SIT, UAT, PROD. Environment values are injected at deploy time. Manual approval gates on UAT and PROD are owned by the organization.
- Images are pushed only to the org's container registry, tagged with SemVer and commit SHA.
- No manual deployments to UAT/PROD. Every prod change traces to a pipeline run and a release tag.

## 6.8 Docker and Kubernetes

- Non-root, read-only root filesystem where feasible, no privileged containers.
- Liveness/readiness (and startup where needed) probes mapped to `/health/live` and `/health/ready`.
- CPU/memory requests and limits. Replica count and PodDisruptionBudget for PROD.
- Config via ConfigMaps. Secrets via K8s Secrets or the org secret provider. Nothing baked into images.
- One namespace per environment/application as agreed with the platform team. Ingress with TLS.
- Graceful shutdown (SIGTERM, `HostOptions.ShutdownTimeout`) so in-flight requests and jobs complete.

Reference Dockerfile (ASP.NET Core, .NET 10):

```dockerfile
# syntax=docker/dockerfile:1
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY global.json Directory.Build.props Directory.Packages.props ./
COPY src/Org.ProjectName.Api/*.csproj src/Org.ProjectName.Api/
RUN dotnet restore src/Org.ProjectName.Api/Org.ProjectName.Api.csproj
COPY . .
RUN dotnet publish src/Org.ProjectName.Api/Org.ProjectName.Api.csproj -c Release -o /app --no-restore
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app
COPY --from=build /app .
USER $APP_UID
ENV ASPNETCORE_HTTP_PORTS=8080
EXPOSE 8080
ENTRYPOINT ["dotnet", "Org.ProjectName.Api.dll"]
```

## 6.9 IIS hosting

- ASP.NET Core Hosting Bundle matching the runtime, installed and documented.
- In-process hosting. One dedicated app pool per application with .NET CLR version = **No Managed Code**.
- Pool identity: ApplicationPoolIdentity or a least-privilege domain service account (documented). A domain account is required for Windows auth to SharePoint or SQL Server.
- If Hangfire servers or message consumers run in-process, the pool MUST use `startMode=AlwaysRunning`, `idleTimeout=0`, `preloadEnabled=true`, or the workload moves to a separate Windows Service or container. Otherwise jobs stop silently.
- HTTPS binding with an org-issued certificate. HTTP redirected. `Server` and `X-Powered-By` removed.
- `ASPNETCORE_ENVIRONMENT` and secret references set via web.config transforms or host config, never hard-coded in appsettings for UAT/PROD.
- Deploy with `app_offline.htm` (Web Deploy or scripted). Scripts go in `deploy/iis/`.

## 6.10 AI-assisted development and non-technical builders

Accountability does not transfer to the tool. A system that the org's engineers cannot read, build, test and deploy is rejected, even if it appears to work.

- AI-assisted development is permitted, but the vendor stays fully accountable. It gets the same standards and gates as hand-written code.
- A non-technical builder MUST be paired with a named, qualified technical reviewer (5+ years .NET or relevant stack) who reviews and co-signs every PR and release.
- Only org-approved AI tools. No org source, data, credentials, personal data or internal documents pasted into public or unapproved AI services.
- Every AI-suggested package MUST be verified to exist on the official registry, be actively maintained and be licence-compliant (anti "slopsquatting").
- Agent and prompt config files that shaped the codebase (`CLAUDE.md`, `AGENTS.md`, `.cursorrules`, `copilot-instructions.md`) are committed to the repo.
- An **AI Usage Declaration** (tools used, scope, review process) is kept current at `docs/ai-usage.md`.
- AI app builder output must be exported to the approved stack and repository. Platform-locked projects are rejected.
