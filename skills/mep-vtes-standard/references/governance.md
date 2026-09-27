# MEP-VTES-001 - Delivery Model and Governance (Sections 1-2)

## Scope (1.2)

- All new systems, rewrites and major enhancements by external parties and subcontractors.
- Web apps, APIs, background services, integrations, mobile apps, SharePoint (OOTB and custom UI).
- Deliveries produced fully or partially with AI tools, coding agents or non-technical builders: identical standards (see cross-cutting.md, 6.10).
- Minor change requests apply the relevant sections proportionally, as agreed in writing with the organization's Technical Lead.

## Compliance (1.4) and exceptions (1.5)

- Compliance is verified at every phase gate (G1-G8). Payment milestones SHOULD be tied to gate approvals.
- A MUST violation blocks the gate unless the organization approves it with conditions and a dated remediation plan.
- Non-compliant work found at any time (including after handover and during warranty) is remediated at the vendor's cost.
- Deviations need a Technology Exception Request (technology, version, licence, justification, alternatives, supportability impact, exit plan), approved by Enterprise Architecture and Cybersecurity **before** use, and recorded as an ADR. No retroactive exceptions.

## Agile framework (2.1)

Scrum with fixed-length sprints (two weeks recommended). Sprint 0 (Inception) baselines analysis, design and architecture. Each later sprint refines them alongside working software.

| # | Phase | Cadence | Gate | Vendor owner |
|---|-------|---------|------|--------------|
| 1 | Initiation & Onboarding | Sprint 0 | G1 - Inception approved | Project Manager |
| 2 | Business Analysis & Requirements | Sprint 0 baseline, refined every sprint | G2 - BRD baselined | Business Analyst |
| 3 | UX / UI Design | Sprint 0 baseline, one sprint ahead of dev | G3 - Design approved | UX/UI Designer |
| 4 | Architecture & Technical Design | Sprint 0 baseline, ADRs every sprint | G4 - Architecture approved | Solution Architect / Tech Lead |
| 5 | Development | Sprints 1..N | Sprint reviews + G5 - Code complete | Tech Lead |
| 6 | Testing & QA | Continuous + hardening sprint | G6 - QA exit / UAT sign-off | QA Lead |
| 7 | Deployment & Release | Per release | G7 - Production readiness | DevOps Engineer |
| 8 | Handover, KT & Warranty | Final release + hypercare | G8 - Handover accepted | PM + Tech Lead |

G1-G4 check the baseline, G5-G7 check each release, and G8 checks the final transfer.

## Azure DevOps as system of record (2.2)

- The organization owns the ADO project. Vendor staff get named accounts, which are revoked at handover.
- **Boards**: Epic, Feature, User Story, Task, Bug, Test Case. Every deliverable, PR and defect is linked to a work item.
- **Repos**: the only source of truth for code, pipelines, infra definitions and technical docs (docs-as-code).
- **Wiki**: documentation index, meeting minutes, sprint review notes, gate submissions.
- **Pipelines**: all builds and deployments. **Test Plans**: test cases and execution evidence.

## Phase gate process (2.3)

1. Vendor prepares deliverables and gate evidence with verifiable references (ADO Wiki URLs, repo paths, work-item IDs, pipeline runs).
2. Vendor Technical Lead (and, for AI-assisted builders, the named technical reviewer) signs to confirm compliance.
3. Submitted as a **Gate Submission** work item in ADO.
4. The organization reviews within 5 business days and records Approved, Approved with conditions (dated), or Rejected.
5. Rejected submissions are resubmitted in full, and the review clock restarts.

## Definition of Ready (2.4)

- Clear title and description (As a / I want / So that), linked to a business requirement ID.
- Acceptance criteria in **Given / When / Then**, agreed by the Product Owner.
- UI design available for UI stories. API contract drafted for integration stories.
- Dependencies identified. Estimated and small enough to finish in one sprint.

## Definition of Done (2.4)

- Merged to main via an approved PR with a passing pipeline (build, lint, tests, scans).
- Unit tests written, coverage thresholds met, no new analyzer warnings.
- Acceptance criteria verified by QA, linked test cases passed, no open Critical/High defects on the story.
- Arabic and English texts and RTL layout verified for UI stories.
- Documentation updated where affected (API spec, ADRs, README, runbook).
- Deployed to SIT/UAT through the pipeline and demonstrated in the sprint review.

## RACI (2.5)

| Activity | Vendor | Org Product Owner | Org Tech Lead | Org Architecture / Security |
|----------|--------|-------------------|---------------|-----------------------------|
| Produce phase deliverables & gate submissions | R/A | I | C | I |
| Approve BRD & backlog | R | A | C | I |
| Approve architecture & ADRs | R | I | A | C |
| Approve technology exceptions | R | I | C | A |
| Approve UAT | C | A | C | I |
| Approve production release | R | C | A | C |
| Accept handover (G8) | R | C | A | C |
