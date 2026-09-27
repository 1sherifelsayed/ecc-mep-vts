---
name: mep-vtes-standard
description: Organization engineering standard MEP-VTES-001 (Vendor Technology and Engineering Standard v1.0) - approved stack, mandatory backend/frontend/mobile architectures, security, JWT, LDAP, CLEF logging, Azure DevOps, CI/CD, Docker/Kubernetes/IIS, AI-assisted delivery and on-prem SharePoint rules. Use when planning, designing, implementing, reviewing or releasing any software, and whenever an orchestration skill picks technologies, architecture, or gate checks.
metadata:
  origin: organization
  document-id: MEP-VTES-001
  document-version: "1.0"
  issued: "2026-09-27"
  classification: Internal - shared with contracted vendors under NDA
---

# MEP-VTES-001 Engineering Standard

MEP-VTES-001 is the organization's mandatory standard for all software it
receives: new systems, rewrites, major enhancements, SharePoint solutions, and
work produced with AI tools or by non-technical builders. It is a contractual
annex to every SOW. **In this installation it overrides generic ECC guidance
wherever the two conflict.** For example, ECC skills that suggest a Node.js
backend, React Native, MongoDB, GitHub Actions or `localStorage` tokens are
superseded by the rules below.

> Classification: Internal, NDA. Do not copy this skill, or its references, into
> public repositories, issues, PRs, or unapproved AI services.

## When to Use

- Any planning, architecture, implementation, review, testing or release work.
- Choosing a language, framework, library, database, broker, scheduler, or host.
- Writing or reviewing ADRs, pipelines, Dockerfiles, Helm/K8s manifests, IIS config.
- Anything touching auth (JWT, AD/LDAP), logging, personal data, or SharePoint.
- Whenever an orchestration skill (`orch-pipeline`, `plan-orchestrate`,
  `team-agent-orchestration`, `dev-team`, and so on) plans or gates work.

## How It Works

### Normative language (Section 1.3)

- **MUST / MUST NOT**: absolute. A violation blocks the phase gate.
- **SHOULD / SHOULD NOT**: any deviation must be justified in an ADR.
- **MAY**: optional.

A deviation from the stack or from a MUST rule needs a written **Technology
Exception Request** (technology, version, licence, justification, alternatives,
supportability impact, exit plan). Enterprise Architecture and Cybersecurity
approve it *before* use, and it is recorded as an ADR. Retroactive exceptions are never
granted. So an agent MUST NOT introduce a non-approved technology on its own.
It stops and tells the user that an exception is required.

### Non-negotiables at a glance

| Area | Rule |
|------|------|
| Backend | C# / ASP.NET Core on **.NET 10 LTS** (.NET 8 not accepted for new work). No Java, Node, Python, PHP or Go services. |
| Web | **Angular** (current Active/LTS) or **React 19 + Vite**, TypeScript strict. No CRA, no Vue, Svelte or Blazor. Next.js only by exception. |
| Mobile | **Flutter** 3.x stable, one state approach (BLoC *or* Riverpod). No React Native, MAUI, Xamarin, Ionic or native-only. |
| Data | **SQL Server 2019+** (2022 preferred) or **PostgreSQL 16+** via EF Core code-first. Redis 7+ for cache only (TTL on every key). |
| Async | **RabbitMQ** 3.13+/4.x (RabbitMQ.Client or MassTransit v8). **Hangfire** Core (OSS). No Kafka or Service Bus, no Quartz. |
| Architecture | Exactly one backend style per deployable: Layered, Vertical Slice, or Clean + DDD, chosen via the matrix and recorded in **ADR-001**. |
| Frontend arch | Feature-first with lint-enforced boundaries. Defaults: Angular uses Nx-style modular libs, React uses Feature-Sliced Design, Flutter uses feature-first Clean. |
| Security | OWASP ASVS L2, OWASP Top 10 and Mobile Top 10, NCA ECC, PDPL. Data stays in KSA. TLS 1.2+ everywhere. Deny-by-default authorization. |
| Auth | JWT: RS256/ES256, access token 15 min or less, rotating hashed refresh tokens with reuse detection, clock skew 30 s or less. LDAPS/StartTLS only. |
| Logging | Serilog **CLEF**, mandatory properties, no PII or secrets, correlation ID propagated end to end. |
| DevOps | **Azure DevOps** is the system of record (Boards, Repos, Wiki, YAML Pipelines, Test Plans). No GitHub, GitLab or Bitbucket as source of truth. |
| Delivery | Same immutable artifact DEV to SIT to UAT to PROD; org-owned approvals on UAT/PROD; no manual deploys. |
| Licences | MIT, Apache-2.0, BSD, ISC, MS-PL only (MPL-2.0 and LGPL after review). No GPL, AGPL, SSPL. Pin everything. CycloneDX SBOM per release. |
| SharePoint | On-prem only. **Approach A** (OOTB config as scripts) or **Approach B** (own web app via REST API). No SPFx, farm solutions, add-ins, or M365/Graph/Power Platform. |
| AI use | Same gates as hand-written code. Verify every AI-suggested package. Commit agent config files. Keep `docs/ai-usage.md` current. |

### Reference files (load the one you need)

| Topic | File |
|-------|------|
| Agile phases, gates G1-G8, Azure DevOps, DoR/DoD, RACI | [references/governance.md](references/governance.md) |
| Approved stack, forbidden technologies, licensing | [references/tech-stack.md](references/tech-stack.md) |
| Backend architecture matrix, solution layouts, API, persistence, messaging | [references/backend.md](references/backend.md) |
| Frontend and mobile architectures, i18n/RTL, a11y, token handling | [references/frontend-mobile.md](references/frontend-mobile.md) |
| Security, JWT, LDAP, CLEF, data, Git, repo layout, CI/CD, Docker/K8s, IIS, AI use | [references/cross-cutting.md](references/cross-cutting.md) |
| SharePoint Approach A/B, forbidden practices, farm governance, ALM | [references/sharepoint.md](references/sharepoint.md) |

### How orchestrators apply it

1. **Intake / plan.** Confirm the work fits the approved stack. For a new backend,
   pick the architecture style with the matrix in `references/backend.md` and plan
   ADR-001. For SharePoint work, decide Approach A or B first (ADR). Put DoR items
   (Given/When/Then acceptance criteria, work-item ID, UI design or API contract)
   into the plan.
2. **Implement.** Follow the mandatory standards for the layer being touched.
   Branch `feature/<workItemId>-short-name`. Commits are Conventional Commits that
   reference the work item (`feat(invoices): add approval AB#1234`).
3. **Review.** Add an MEP-VTES compliance pass to every code review using the
   checklist below. Any MUST violation counts as a **blocking (HIGH/CRITICAL)** finding.
   String-concatenated SQL is always **CRITICAL**.
4. **Release / gate.** Check the Definition of Done and the gate evidence (ADO work items,
   pipeline runs, repo paths) before declaring work complete.

### Compliance review checklist

- [ ] Only approved technologies and versions; no new dependency without licence,
      maintenance (<18 months stale) and CVE check; versions pinned; package
      really exists on the official registry.
- [ ] Backend style matches ADR-001; no mixing styles; architecture tests present
      (Vertical Slice / Clean).
- [ ] API: `/api/v1/...`, OpenAPI 3.x, camelCase JSON, ProblemDetails (RFC 9457),
      server-side paging/filtering/sorting, server-side validation.
- [ ] EF Core migrations committed plus idempotent script; no auto-migrate in UAT/PROD;
      audit columns; UTC storage; naming (PG snake_case / SQL Server PascalCase).
- [ ] Options validated on start; no secrets in Git, images, logs or docs.
- [ ] `/health/live` and `/health/ready`; resilience (retry with jitter, timeout,
      circuit breaker) on outbound HTTP; explicit timeouts on DB, Redis, LDAP,
      SharePoint and broker.
- [ ] Authorization global and policy-based; `[AllowAnonymous]` only where approved;
      audit trail on CUD and security events; security headers set.
- [ ] JWT and LDAP rules (Section 6.2 / 6.3) met.
- [ ] CLEF logging with mandatory properties, message templates, no forbidden data.
- [ ] Frontend: feature-first, public API per feature, lint-enforced boundaries,
      generated API clients, lazy routes, ar-SA/en-US plus RTL, WCAG 2.1 AA, tokens
      never in localStorage/sessionStorage.
- [ ] Tests: unit + integration (Testcontainers) + E2E (Playwright /
      integration_test); coverage gate met; zero new analyzer warnings.
- [ ] Docs updated where affected: README (setup in 30 minutes or less), ADRs (MADR), OpenAPI,
      runbook, CHANGELOG (Keep a Changelog), `docs/ai-usage.md`.
- [ ] Azure Pipelines YAML: build, lint, tests, arch tests, coverage, SAST, SCA, secret
      scan, image scan, SBOM.

## Examples

**Plan request: "Add a Node/Express microservice for notifications."**
Flag it as non-compliant (Section 3.1: only C#/.NET backends). Propose an ASP.NET
Core Worker host that consumes RabbitMQ and uses Hangfire for retries, placed in
the existing architecture style. Offer a Technology Exception Request only if the
user insists on Node.

**Review finding.**
`HIGH - MEP-VTES-001 s6.2: access token lifetime is 60 min (MUST be 15 min or less); ClockSkew
defaults to 5 min (MUST be 30 s or less). Fix: set ExpiresIn=15m and TokenValidationParameters.ClockSkew = TimeSpan.FromSeconds(30).`

**Library choice.**
Mapping DTOs in .NET: do not add AutoMapper 15+ or MediatR 13+ (commercial
licences). Use Mapperly or hand-written mappers, and direct handlers, or pin the last
open-source major.

**SharePoint request: "Approval dashboard over a document library."**
This needs business rules and dashboards, so it is Approach B: an Angular/React + ASP.NET Core
app on IIS. Structured data goes in SQL Server/PostgreSQL, documents go in SharePoint through
REST behind `IDocumentStore`, and there is no SPFx and no list-based reporting.
