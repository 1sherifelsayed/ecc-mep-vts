# MEP-VTES-001 - SharePoint Standards, On-Premises (Section 7)

SharePoint Server is on-premises only. There is no SharePoint Online, M365 or Power Platform tenancy, no cloud service may be introduced, and SPFx is not used. Every requirement MUST be met by exactly one of two approaches.

## 7.1 The two approved approaches

| | Approach A - OOTB configuration | Approach B - custom UI internal web app |
|---|---|---|
| What | Standard sites, lists, libraries, content types, managed metadata, views, permissions, OOTB web parts: **configured only, no code** | Ordinary internal web app (Angular/React + ASP.NET Core on IIS, built to Sections 3-6) using SharePoint as a document/content store via the **on-prem REST API**. The UI is the app's own, not a SharePoint page. |
| Use when | Collaboration, document management, simple record-keeping: team sites, libraries with metadata/versioning, simple lists, content approval, alerts, search | Tailored UX, business rules, validation, calculations, dashboards, multi-step workflow, or integration with other systems |
| Rules | 7.3 | 7.4-7.5 |

- The approach is chosen at design time, recorded in an ADR, and approved before development. Use A whenever it satisfies the requirement, otherwise B.
- Never blend the two by customizing SharePoint. B extends SharePoint from outside via REST and never deploys code into the farm.
- SharePoint is not a relational database. In B, structured business data (transactions, master data, relational integrity, complex queries, reporting, high write volume) goes in SQL Server/PostgreSQL.
- A list expected to exceed the list view threshold (5,000 items by default) in active use means the data belongs in the app database.
- Business rules, approvals and scheduled processing in B live in the ASP.NET Core app (Hangfire, RabbitMQ), never in SharePoint.
- Reporting/dashboards are built in the B app over its database, not by querying SharePoint lists at run time.

Rule: if configuring standard SharePoint meets the need, use A. The moment a screen, rule or data set goes beyond it, use B. There is no third option.

## 7.2 Not permitted

| Practice | Replacement |
|----------|-------------|
| SPFx web parts and extensions | Approach B |
| Full-trust farm solutions (.wsp) with server code | Approach B |
| Sandboxed code solutions | Approach B |
| SharePoint Add-ins (SharePoint- or provider-hosted), ACS app-only principals | Approach B with a Windows service account (7.5) |
| SharePoint Designer workflows/customizations | Untracked and not reproducible, so not allowed |
| InfoPath forms | OOTB list form (A) or B app |
| SharePoint 2010/2013 declarative workflows | Workflow in the B app |
| Script Editor / Content Editor with JS, JSLink, custom master pages, page layouts with code, DOM manipulation | A configuration or B |
| Direct reads/writes to content, config or search databases | SharePoint REST API |
| Third-party web parts, add-ins, farm-level products | Approved exception before procurement |
| Manual click-configuration of UAT/production | Provisioning scripts (7.3) |
| Any SharePoint Online, M365, Graph, Power Apps, Power Automate dependency | Not allowed; content stays on-prem |

### Farm governance (both approaches)

- Org SharePoint admins own the farm. Vendors never get farm admin rights and never install software, solutions, features, certificates or scheduled tasks on SharePoint servers.
- Farm-level changes (service application, quota, managed path, web app setting, search config, Kerberos SPN) are raised as change requests and performed by org admins.
- Farm version/build, web app URLs and in-scope site collections are confirmed in writing before design and recorded in README and `docs/architecture`.
- Vendors get a dedicated dev site collection plus a UAT site collection that mirrors production. Never develop or test on production sites. Prefer a separate dev farm when one is provided.
- Solutions survive farm patching: no dependency on undocumented behaviour, internal URLs, page markup or DB schema.

## 7.3 Approach A standards

| Area | Requirement |
|------|-------------|
| Information architecture | Approved IA definition (site hierarchy, lists/libraries, content types, site columns, term sets, views, navigation, search settings) before provisioning. |
| Content types & columns | Reusable site columns and content types at site-collection or content type hub level, never ad-hoc list columns. Internal names in English without spaces. Display names in Arabic and English. |
| Permissions | Grant to AD security groups inside SharePoint groups, never to individuals. Break inheritance only at site or library level, and record every break in a delivered permission matrix. Item-level unique permissions at scale are forbidden. |
| Scale | Design within the 5,000-item threshold: indexed columns, filtered views, metadata navigation, folders. Agree library sizes, item counts and file sizes with admins. |
| Versioning & retention | Versioning with explicit major/minor limits. Content approval, retention and records per business requirements and the records policy. |
| Forms & formatting | OOTB list forms and supported code-free column/view formatting only. If that is not enough, move to Approach B. Never inject script. |
| Branding & language | Arabic + English via platform language features, RTL verified. Supported theming only. No custom master pages or page layouts. |
| Provisioning as code | Idempotent, re-runnable **PnP PowerShell / SharePoint Management Shell** scripts (or farm-supported provisioning templates) in Git under `deploy/sharepoint/`, run by the org in DEV, UAT, PROD. A click-by-click guide is not acceptable. |

## 7.4 Approach B standards

- It is an ordinary internal web app and follows Sections 3-6 in full. Nothing here relaxes them.
- The UI is served by the app on IIS. SharePoint pages are not modified; at most a link or nav entry points to the app.
- SharePoint is reached **only** via the on-prem REST API (`/_api`) over HTTPS **from the ASP.NET Core backend**. The browser never calls SharePoint directly.
- SharePoint access sits behind an app-defined interface (e.g. `IDocumentStore`, `ISharePointListClient`) in Infrastructure (Clean) or the feature's data-access folder (Vertical Slice). No SharePoint types in Domain or Application.
- Site URLs, list titles and IDs are configuration. Items/files are referenced by identifier (UniqueId / item ID), never by display URL.
- CSOM assemblies target .NET Framework and are not used from .NET 10 services. A CSOM-only operation needs an exception and is isolated in a small, separately hosted component.
- Large files are chunked and streamed, never fully buffered. Timeouts, retry with jitter and circuit breaker per 4.5.
- Uploads are validated (size, extension, content type) and malware-scanned before writing to SharePoint.
- Every SharePoint call is logged in CLEF with CorrelationId, operation and duration, and never file contents, personal data or credentials.
- Degrades predictably when SharePoint is down: clear message, no data loss, queued retry (RabbitMQ/Hangfire) for deferrable operations.
- SharePoint is included in the readiness health check and in integration tests (against the dev site collection, never production).

## 7.5 Approach B authentication and authorization

- End users authenticate with Windows/AD (Kerberos preferred, NTLM fallback) or the approved directory integration (6.3). The app then issues its own JWT per 6.2.
- The app calls SharePoint with a dedicated, least-privilege AD service account scoped to specific sites, lists and libraries. It is never a farm or domain admin, and a site collection admin only if formally approved.
- Service-account credentials come from the org secret store. Rotation is documented and possible without code change or redeploy.
- The service account MUST NOT bypass SharePoint security. The app enforces its own policy-based, deny-by-default authorization mapped to the same AD groups, so users can't reach through the app what they can't reach directly.
- Per-user SharePoint access requires **Kerberos constrained delegation**, agreed with the org before the design is finalized.
- Every SharePoint create/update/delete on a user's behalf is audited (acting user, target item, timestamp).

## 7.6 SharePoint ALM and deployment

- App source, provisioning scripts, IA definition and permission matrix live in the org's ADO repo and follow 6.6.
- Azure Pipelines build/test/package like any internal web app (6.7) and deploy to IIS (6.9).
- SharePoint config changes are released like code: a versioned, idempotent script run DEV, then UAT, then PROD, with org approval per promotion.
- Every release records the SharePoint config state it expects and a documented way to revert it.
- UAT runs on a site collection mirroring production with anonymized or synthetic content. Production content is never copied to vendor machines or dev.
- Production execution of provisioning scripts and IIS deploys is done by the org, or by a pipeline with org-owned credentials behind an approval gate. It is never done by a vendor with a personal or shared admin account.
