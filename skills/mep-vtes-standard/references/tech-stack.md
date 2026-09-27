# MEP-VTES-001 - Approved Technology Stack (Section 3)

Only these technologies may be used. Versions MUST be within the vendor's official support window at handover and SHOULD NOT reach end of support within 12 months after go-live.

| Layer | Technology | Version rule | Mandatory usage notes |
|-------|------------|--------------|-----------------------|
| Web frontend | Angular | Current Active or LTS major only | TypeScript strict. Standalone components, lazy-loaded feature routes, Signals for local state. Angular CLI or Nx workspace. |
| Web frontend | React | React 19 (or current supported major), TypeScript strict, built with **Vite** | Create React App MUST NOT be used. Next.js only with a written exception. |
| Mobile | Flutter | Latest stable 3.x, Dart sound null safety | Single codebase Android + iOS. One state-management approach per project (BLoC **or** Riverpod). |
| Backend / API | C# on ASP.NET Core / .NET | New projects: **.NET 10 (LTS)**. .NET 8 (EOS 10 Nov 2026) NOT accepted for new work | ASP.NET Core Web API (Controllers or Minimal APIs). Nullable reference types enabled. Central Package Management. |
| Relational DB | Microsoft SQL Server | 2019 or later (2022 preferred) | EF Core code-first migrations. Dapper permitted for read-heavy queries. No business logic in stored procedures. |
| Relational DB | PostgreSQL | 16 or later | EF Core + Npgsql. snake_case naming (EFCore.NamingConventions). |
| Distributed cache | Redis | 7.x or later | Cache and short-lived state only, never the system of record. TTL on every key. |
| Messaging | RabbitMQ | 3.13+ or 4.x | RabbitMQ.Client or MassTransit v8 (Apache-2.0). Commercial versions need an exception. |
| Background jobs | Hangfire | Core (OSS edition), SQL Server or PostgreSQL storage | Pro/Ace only with an exception. Dashboard MUST be behind authorization. |
| Logging | Structured logs in CLEF | Serilog + Serilog.Formatting.Compact | stdout in containers, rolling files on IIS. Mandatory properties in cross-cutting.md 6.4. |
| Authentication | JWT Bearer | Microsoft.AspNetCore.Authentication.JwtBearer | Rules in cross-cutting.md 6.2. |
| Directory | Active Directory over LDAP (when needed) | LDAPS (636) or LDAP + StartTLS only | System.DirectoryServices.Protocols. Rules in cross-cutting.md 6.3. |
| Collaboration | SharePoint Server on-prem - OOTB config | Org farm; version/build confirmed per project. No SharePoint Online / M365 | Approach A: configuration only, delivered as re-runnable provisioning scripts. |
| Collaboration | SharePoint Server on-prem - custom UI app | Angular or React + ASP.NET Core on IIS via on-prem REST API | Approach B: the UI is an ordinary internal web app. SPFx, farm solutions, add-ins, SharePoint Designer, InfoPath NOT permitted. |
| Source control | Git on Azure DevOps Repos | Org-owned ADO project | No GitHub/GitLab/Bitbucket as primary source of truth. |
| Work management | Azure DevOps Boards | Org-owned ADO project | Epics, Features, User Stories, Tasks, Bugs, Test Cases. All deliverables linked. |
| CI/CD | Azure Pipelines (YAML) | YAML stored in the repo | Classic (UI-defined) pipelines not accepted. |
| Containers | Docker | OCI images, multi-stage builds | Official `mcr.microsoft.com/dotnet/*` base images pinned by tag or digest. |
| Orchestration | Kubernetes | Supported upstream version as run by the org | Helm charts or Kustomize overlays per environment. Probes and resource limits mandatory. |
| Web hosting (Windows) | IIS | IIS 10 on Windows Server + ASP.NET Core Hosting Bundle matching the runtime | In-process hosting, dedicated "No Managed Code" app pool per application. |

## Not permitted without an approved exception (3.1)

- Backend languages/runtimes other than C#/.NET (Java, Node.js services, Python services, PHP, Go). Node.js is allowed **only as frontend build tooling**.
- Frontend frameworks other than Angular or React (Vue, Svelte, jQuery-based new UIs, Blazor).
- Mobile frameworks other than Flutter (React Native, .NET MAUI, Xamarin, Ionic, native-only apps).
- Systems of record other than SQL Server or PostgreSQL (MongoDB, MySQL, Oracle for new builds, Firebase). Oracle EBS/HR is reached only through approved integration interfaces.
- Brokers other than RabbitMQ (Kafka, Azure Service Bus, AWS SQS). Schedulers other than Hangfire (Quartz.NET, Windows Task Scheduler for business jobs).
- SharePoint Online, Microsoft 365, Microsoft Graph, Power Apps, Power Automate, or any cloud-hosted collaboration service.
- All SharePoint code customization: SPFx, full-trust farm solutions, sandboxed solutions, add-ins, SharePoint Designer, InfoPath, legacy workflows.
- Hosting on vendor-owned infrastructure, public SaaS back-ends, or anywhere outside the org's approved data centres.
- Low-code/no-code platforms or AI app builders whose output cannot be exported as maintainable source in the approved stack.
- Unmaintained or deprecated packages, packages with known High/Critical CVEs, copyleft licences (GPL, AGPL, SSPL) in distributed code.

## Third-party libraries and licensing (3.2)

- Permitted: MIT, Apache-2.0, BSD-2/3-Clause, ISC, MS-PL. MPL-2.0 and LGPL after review. GPL, AGPL, SSPL and "source-available" are forbidden in delivered code.
- Pin all versions: `Directory.Packages.props` (.NET), `package-lock.json` / `pnpm-lock.yaml` (web), `pubspec.lock` (Flutter).
- No dependency with an unresolved High/Critical CVE. No package unmaintained for more than 18 months.
- The pipeline produces a **CycloneDX SBOM** for every release.

### Commercial licence alert

Several .NET libraries moved to commercial licensing in 2025: **MediatR 13+, AutoMapper 15+, MassTransit 9+, FluentAssertions 8+**. Either pin to the last open-source major, use a licence-free alternative (hand-written mappers or **Mapperly**, direct handlers, **Shouldly**), or get written approval with the licence procured in the organization's name. Licences bought in the vendor's name are rejected at handover.
