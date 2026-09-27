---
name: orch-pipeline
description: Shared orchestration engine for the orch-* skill family. Defines the gated Research-Plan-TDD-Review-Commit pipeline, the size classifier, the agent map, and the two human gates that the orch-* operation skills delegate to. Not usually invoked directly. Not usually invoked directly; it applies when an orch-* skill delegates its gated Research-Plan-TDD-Review-Commit pipeline.
metadata:
  origin: ECC
---

# Orchestrator Pipeline (shared engine)

The `orch-*` skills are thin wrappers. They do not re-implement any work — they
classify the request, choose which phases of *this* pipeline run, and delegate
each phase to an existing ECC agent or command. This file is that pipeline.

> Invoke an operation skill (`orch-add-feature`, `orch-fix-defect`, …) rather
> than this engine directly. This file is the reference they point at.

## When to Use

- Loaded indirectly whenever an `orch-*` operation skill runs.
- Read directly only when adding a new operation to the family or tuning the
  shared phases, gates, or agent map.

## Organization standard (MEP-VTES-001), always on

Every run of this pipeline is governed by the organization's Vendor Technology
and Engineering Standard. Load the `mep-vtes-standard`
skill at Intake and apply it in every phase. Where it conflicts with generic ECC
guidance (including `rules/common/*`), the standard wins. In short:

- **Stack gate.** Only approved technologies (C#/.NET 10 backends, Angular or
  React+Vite, Flutter, SQL Server/PostgreSQL, Redis, RabbitMQ, Hangfire, Azure
  DevOps, Azure Pipelines YAML, Docker/K8s/IIS). Anything else needs a
  Technology Exception Request **before** use. Stop and tell the user rather
  than proceeding.
- **Architecture gate.** Backends use exactly one style (Layered / Vertical Slice /
  Clean+DDD) per ADR-001. Frontends are feature-first with lint-enforced
  boundaries. SharePoint work picks Approach A or B by ADR first.
- **Any MUST violation is a blocking finding** at Review and at both gates.

## The operation family

| Skill | Operation | Trigger | First move |
|-------|-----------|---------|------------|
| `orch-add-feature` | feature | capability does not exist yet | research + plan a new slice |
| `orch-change-feature` | tweak | works, but desired behavior differs | amend existing behavior *and its tests* |
| `orch-fix-defect` | fix | broken; behavior is wrong | reproduce as a failing test, then fix |
| `orch-refine-code` | refactor | behavior stays, structure improves | restructure while keeping tests green |
| `orch-build-mvp` | mvp | bootstrap from a design/spec doc | ingest doc → vertical slices |

> These wrappers **compose** existing ECC commands rather than replace them:
> `/feature-dev`, `/plan`, `/code-review`, `/build-fix`, `/refactor-clean`, and
> `/gan-build`, plus the `tdd-workflow` skill. The orch-* family adds the shared
> size classifier and the two gates
> on top of them, so one umbrella covers all five operations consistently.

## Step 0 — Classify size (right-sizing)

Ceremony scales to blast radius. Score the request on three signals, take the
**highest** tier any signal reaches, and state the result in one line so the user
can override:

| Tier | Files touched | New dependency / contract | Design ambiguity | Phases that run |
|------|---------------|---------------------------|------------------|-----------------|
| trivial | 1, a few lines | none | none — the change is obvious | 4 → 5 → 6 |
| small | 1 file / 1 function | none | clear once you read the code | (1 light) → 4 → 5 → 6 |
| standard | 2–5 files | maybe a new internal module | one real choice to make | 1 → 2 → 4 → 5 → 6 |
| large | many / cross-cutting | new external dep, public API, or a spec doc | multiple open questions | 1 → 2 → (3) → 4 → 5 → 6 |

Phase 0 (Intake) always runs and is omitted from the mask column above. The
tie-breaker: anything touching a security trigger (below) or a public API /
contract is **at least** standard, regardless of file count.

## The phases

Each phase delegates — it does not do the work inline.

- **0. Intake** — restate the request. For `orch-build-mvp`, read the spec/design
  doc and extract scope, locked decisions, and a feature list.
- **1. Research & Reuse** — per `rules/common/development-workflow.md`: `gh search repos` /
  `gh search code`, then Context7 / vendor docs, then package registries, then
  Exa. Prefer adopting a proven implementation over net-new code. Only consider
  candidates in the MEP-VTES-001 approved stack. Before proposing any package, verify it
  exists on the official registry, is maintained (updated within 18 months), has no
  open High/Critical CVE, and carries a permitted licence (MIT/Apache-2.0/BSD/ISC/MS-PL,
  never GPL/AGPL/SSPL). Watch for the commercial-licence .NET libraries (MediatR 13+,
  AutoMapper 15+, MassTransit 9+, FluentAssertions 8+). Never paste org source, data
  or documents into public search or unapproved AI services.
- **2. Plan** — delegate to the `planner` agent (or `architect` /
  `code-architect` for structural decisions). Output a `task_list` ordered as
  thin vertical slices. → **GATE 1.** Per MEP-VTES-001 the plan must: name the
  backend architecture style and its ADR-001 justification (new backends); name the
  SharePoint approach (A/B) with an ADR (SharePoint work); tie each task to an Azure
  DevOps work-item ID with Given/When/Then acceptance criteria (Definition of Ready);
  and list the docs the change touches (OpenAPI, ADRs, README, runbook, CHANGELOG,
  `docs/ai-usage.md`).
- **3. Scaffold** — `orch-build-mvp` only: stand up the first end-to-end slice.
- **4. Implement (TDD)** — drive each task through the `tdd-guide` agent (or the `tdd-workflow` skill):
  red → green → refactor. Honor the operation's first-move rule.
- **5. Review** — `code-reviewer` agent / `/code-review`. Add `security-reviewer`
  whenever the diff touches a security trigger (below). Every review also runs the
  MEP-VTES-001 compliance checklist from `mep-vtes-standard`. Pass it to each
  reviewer agent in its prompt, and report violations as `HIGH - MEP-VTES-001 s<section>: ...`
  (string-concatenated SQL is `CRITICAL`).
- **6. Commit** — conventional commits (`feat:` / `fix:` / `refactor:` / …), one
  per logical chunk, each referencing its work item (`feat(invoices): add approval AB#1234`),
  on a `feature/<workItemId>-short-name` or `bugfix/<workItemId>-short-name` branch.
  Changes reach main only through an Azure DevOps PR (no direct or force pushes). → **GATE 2.**

## The two gates

This family is **gated, not autonomous**:

1. **GATE 1 — after Plan.** Present the `task_list`; do not write implementation
   code until the user approves.
2. **GATE 2 — before Commit.** Present the diff summary and proposed messages;
   do not commit until the user confirms.

Everything between the gates flows without stopping.

## Agent / command map

| Phase | Primary | Fallback / escalation |
|-------|---------|----------------------|
| Intake / understand | `code-explorer` | trace existing paths before a tweak, fix, or refactor |
| Plan | `planner` | `architect`, `code-architect` for structural calls |
| Implement | `tdd-guide` (or `tdd-workflow` skill) | `build-error-resolver` / `/build-fix` on build breaks |
| Review | `code-reviewer` / `/code-review` | language reviewer (`python-reviewer`, `typescript-reviewer`, …) |
| Security | `security-reviewer` | — |
| MVP inner loop | `/gan-build "<brief>" --skip-planner` | drives `gan-generator` → `gan-evaluator`; tune `--max-iterations` / `--pass-threshold` |

Match the language reviewer to the repo (see the repo's own `CLAUDE.md`). For
MEP-VTES-001 stacks this means `csharp-reviewer` (.NET), `typescript-reviewer` /
`react-reviewer` (Angular/React), `flutter-reviewer` (Flutter), and `database-reviewer`
for EF Core migrations and SQL.

## Security-review trigger

Pull in `security-reviewer` when the diff touches any of: authentication or
authorization, user-input handling, database queries, file-system paths,
external API calls, cryptography, or secrets / credentials. (Per `rules/common/security.md`.)
Under MEP-VTES-001, also include JWT issuance/validation, AD/LDAP binds or filters,
SharePoint REST access or service accounts, file uploads, logging that may carry
personal data, and CI/CD, Docker/K8s or IIS configuration.

## Handoff artifacts

The pipeline carries no hidden state — the planning docs *are* the handoff:

- `task_list` (from Plan) drives the Implement loop.
- Larger work may also emit PRD / architecture / system_design under the repo's
  `docs/` per `rules/common/development-workflow.md`.
- Review findings (CRITICAL / HIGH) must be resolved before Gate 2. This includes every
  MEP-VTES-001 MUST violation, unless the user holds an approved Technology Exception
  (recorded as an ADR).

## Verification

- size tier was stated and matched the work
- Gate 1 (plan) and Gate 2 (commit) were both honored
- `security-reviewer` ran iff a security trigger was touched
- commits are conventional and scoped to one logical change
- new / changed behavior has tests; coverage ≥ 80% per `rules/common/testing.md`
- MEP-VTES-001 compliance checklist passed, and the Definition of Done items that apply are
  met (pipeline green incl. lint/tests/scans, docs updated, ar/en + RTL verified for UI)
