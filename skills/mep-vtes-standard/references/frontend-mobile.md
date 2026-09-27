# MEP-VTES-001 - Frontend and Mobile Architecture Standards (Section 5)

Frontend and mobile projects MUST be organized by business feature with enforced boundaries. Each technology has a default architecture. An approved alternative needs an ADR.

## Common principles (5.1)

- **Feature-first**: organized by business feature (invoices, employees, leave-requests), never by technical type at the top level (no app-wide `components/`, `services/`, `models/`).
- **Public API per feature**: one entry point (`index.ts` / barrel / routes file). Deep imports into another feature's internals are forbidden.
- **Unidirectional dependencies**: shared/core never import from features. Features never import from the app shell.
- **Tool-enforced boundaries**: lint rules in CI, not convention alone.
- **Generated API clients**: HTTP clients and DTOs generated from the backend OpenAPI document (NSwag, openapi-generator, orval), committed or generated in the pipeline.
- **Lazy loading**: every feature lazy-loaded (route-level code splitting) unless justified in an ADR.

## Approved architecture per technology (5.2)

| Technology | Default | Allowed alternative (with ADR) |
|------------|---------|-------------------------------|
| Angular | B. Domain-Driven Modular Libraries | A. FSD (adapted) |
| React | A. Feature-Sliced Design | B. Modular Libraries (Nx) |
| Flutter | C. Feature-First Clean Architecture | Feature-first with Riverpod (lighter data layer) |

## A. Feature-Sliced Design - React default (5.3)

Layers `app, pages, widgets, features, entities, shared`, each split into slices (business features) and segments (`ui, model, api, lib`). Enforce with eslint-plugin-boundaries or Steiger in CI, path aliases (`@/features`, `@/entities`), and a public `index.ts` per slice.

```text
src/
  app/        -> providers, router, global styles, bootstrapping
  pages/      -> route-level compositions (invoice-list/, invoice-details/)
  widgets/    -> large self-contained UI blocks (invoice-summary-panel/)
  features/   -> user interactions delivering business value
    create-invoice/  ui/ model/ api/ lib/ index.ts   <- public API
  entities/   -> business entities: types, API calls, entity UI
    invoice/  ui/ model/ api/ index.ts
  shared/     -> framework-agnostic: ui-kit, api client, config, lib, i18n
```

Import rule: `app > pages > widgets > features > entities > shared` (downward only).

## B. Domain-Driven Modular Libraries (Nx-style) - Angular default (5.4)

Domains (bounded contexts) are split into typed libraries: `feature` (smart, routed), `data-access` (state + API), `ui` (presentational), `util`. Enforce with `@nx/enforce-module-boundaries` using scope/type tags, and use Nx affected builds in CI.

```text
apps/finance-portal/          -> thin shell: routing, layout, bootstrapping
libs/
  invoices/
    feature-invoice-list/     -> smart components + routes (lazy)
    feature-invoice-editor/
    data-access/              -> API services, state (NgRx SignalStore / signals), facades
    ui/                       -> presentational components
    util/                     -> pure functions, pipes, validators
  shared/  ui/ data-access-auth/ util-i18n/ util-date/
```

Tag rules: `type:feature` -> data-access, ui, util. `type:ui` -> ui, util only. `type:data-access` -> data-access, util only. `scope:invoices` -> scope:invoices and scope:shared only.
Single-app variant (no Nx): `src/app/{core, shared, features/<feature>/{pages, components, data-access, models, <feature>.routes.ts}}` with eslint-plugin-boundaries.

## C. Feature-First Clean Architecture - Flutter default (5.5)

Each feature has its own `data`, `domain` and `presentation` layers. Domain is pure Dart. Enforce with flutter_lints / very_good_analysis + custom_lint import rules. Use get_it/injectable or Riverpod for DI, with one state approach per project recorded in an ADR.

```text
lib/
  main_dev.dart | main_uat.dart | main_prod.dart  -> flavor entry points
  app/     -> MaterialApp, go_router, theme, DI bootstrap
  core/    -> dio + interceptors, error types, secure storage, ARB localization, constants
  features/
    leave_requests/
      data/          datasources/ models/ (json_serializable/freezed) repositories/
      domain/        entities/ repositories/ (abstract) usecases/ (one class each)
      presentation/  bloc/ (or providers/) pages/ widgets/
test/ (mirrors lib/)   integration_test/
```

Rule: `presentation -> domain <- data`. Domain has no Flutter or package imports.

## Mandatory frontend/mobile standards (5.6)

| Area | Requirement |
|------|-------------|
| Language & lint | TypeScript strict; ESLint + Prettier (Angular/React); Dart analyzer with zero warnings. Lint runs in CI and blocks merge. |
| Localization & RTL | Arabic + English with full RTL. Angular: @angular/localize or Transloco. React: i18next. Flutter: flutter_localizations + ARB. No hard-coded user-facing strings. |
| Accessibility | WCAG 2.1 AA: semantic markup, keyboard navigation, contrast, labels, screen-reader support (Semantics widgets in Flutter). |
| Token handling (web) | Access token **in memory only**. Refresh token in an HttpOnly, Secure, SameSite=Strict cookie (BFF preferred). **Never** in localStorage/sessionStorage. |
| Token handling (mobile) | flutter_secure_storage (Keychain / Keystore). Release builds obfuscated (`--obfuscate --split-debug-info`) with symbols handed over. |
| Configuration | Env files / Flutter flavors per environment. No secrets in any client bundle. |
| Error handling | Global HTTP interceptor mapping ProblemDetails to user messages. Global error boundary / ErrorHandler / FlutterError.onError with CLEF-compatible remote logging where required. |
| Performance | Angular build budgets. Lighthouse performance and accessibility of 80 or more on key pages. Flutter release builds profiled, no jank on target devices. |
| Testing | Unit: Vitest or Jest (Angular/React), flutter_test. E2E: Playwright (web), integration_test (Flutter). |
| Mobile release assets | Android keystore, Apple certs/profiles and store listings under the **organization's** accounts, never the vendor's. |
