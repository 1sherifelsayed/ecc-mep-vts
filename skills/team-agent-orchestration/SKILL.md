---
name: team-agent-orchestration
description: "Run team-based orchestration for agent squads using work items, ownership, agent Kanban, merge gates, and control pane handoffs. Use when coordinating an agent squad with work items, ownership, Kanban, and merge gates."
metadata:
  origin: ECC
---

# Team Agent Orchestration

Use this skill when agents are being managed like a team rather than a single assistant. The purpose is to make team-based orchestration reliable: clear work items, explicit ownership, agent Kanban state, branch isolation, control pane visibility, and merge gates.

## When To Activate

- The task spans multiple agents, tools, harnesses, branches, or worktrees.
- The user mentions team orchestration, agent Kanban, squad, conductor, control pane, manager, desktop app, Zellij, tmux, Hermes, Devin, Codex, Claude Code, or multi-agent work.
- A project needs shared workflow state across people and agents.
- Existing agent fan-out is producing output but not mergeable product.

## Organization Standard (MEP-VTES-001)

All team work is governed by the organization's Vendor Technology and Engineering
Standard. Load `mep-vtes-standard` before shaping the
board and pass its relevant rules into every agent's card prompt:

- **Work items**: cards mirror Azure DevOps work items (Epic / Feature / User Story /
  Task / Bug). A card is Ready only when it meets the Definition of Ready (linked
  requirement ID, Given/When/Then acceptance criteria, UI design or API contract).
- **Branches**: `feature/<workItemId>-short-name` / `bugfix/<workItemId>-short-name`;
  commits are Conventional Commits with `AB#<id>`; integration only via an Azure DevOps
  PR with build validation and an org reviewer on main.
- **Merge gate = Definition of Done**: pipeline green (build, lint, unit + architecture
  tests, coverage, SAST, SCA, secret and image scans), MEP-VTES-001 compliance checklist
  clean, docs updated (OpenAPI, ADRs, README, runbook, `docs/ai-usage.md`), ar/en + RTL
  verified for UI.
- **Scope guard**: no agent introduces a technology outside the approved stack. That
  needs a Technology Exception Request, so the card moves to Blocked with the owner set to the user.
- **AI accountability**: agent output is held to the same gates as human code, every
  suggested package is verified to exist, be maintained and be licence-compliant, and the
  agent config files that shaped the code are committed.

## Operating Model

Treat every agent as a teammate with a narrow contract:

- **Owner**: the person or agent accountable for the work item.
- **Scope**: files, branch, tool surface, and forbidden areas.
- **State**: backlog, ready, running, review, blocked, merged, or archived.
- **Evidence**: tests, screenshots, logs, review notes, or eval reports.
- **Merge gate**: the exact condition that allows integration.

## Agent Kanban

Use agent Kanban when work must be visible across sessions.

| Column | Meaning | Exit Criteria |
| --- | --- | --- |
| Backlog | Candidate work item, not yet shaped | Acceptance criteria written |
| Ready | Shaped and assignable | Owner and branch/worktree assigned |
| Running | Agent is actively working | Handoff artifact and changed files exist |
| Review | Work is complete but not merged | Tests, diff review, and risk check pass |
| Blocked | Needs external input or failed gate | Blocker has owner and next action |
| Merged | Integrated into mainline | PR merged or local main updated |
| Archived | No longer relevant | Reason recorded |

Each card should fit this schema:

```json
{
  "id": "agent-card-001",
  "title": "Build dynamic workflow skill",
  "owner": "codex",
  "state": "running",
  "branch": "product/dynamic-workflow-team-orchestration",
  "worktree": ".",
  "acceptance": [
    "Skill exists",
    "Tests cover required concepts",
    "Content artifact contains video and article angles"
  ],
  "merge_gate": "lint, focused tests, and catalog check pass",
  "handoff": "path/to/handoff.md"
}
```

## Team-Based Orchestration Flow

1. **Shape the board**: convert fuzzy ambition into work items with owners and merge gates.
2. **Pick execution mode**: single-agent, dynamic workflow mode, dmux/tmux, worktree fan-out, or external desktop orchestrator.
3. **Assign boundaries**: one owner per card, clear file scope, and no overlapping writes without an integrator.
4. **Run agents**: each agent writes evidence and handoff notes, not just code.
5. **Review in sequence**: tests first, then diff review, then security/risk checks, then content/product polish.
6. **Merge deliberately**: one integrator resolves conflicts and updates the control pane or status artifact.
7. **Extract reusable skill**: if the card pattern repeats, promote it into `skills/`.

## Control Pane Requirements

A useful control pane for team orchestration should show:

- Active work items and their agent Kanban state.
- Owner, harness, branch, worktree, and last heartbeat.
- Links to handoff artifacts, tests, screenshots, and PRs.
- Blockers grouped by owner and unblock action.
- Merge readiness by gate, not vibes.
- Reusable workflow candidates that should become shared skills.

Do not add more automation until the operator can answer: who owns this, what changed, what gate failed, and what can safely merge?

## Dynamic Workflow Compatibility

When a card needs dynamic workflow mode:

- Put the task-local harness under the card owner.
- Store inputs and outputs on the card.
- Require an eval before moving from Running to Review.
- Promote the harness to a shared skill only after repeat use.

## Failure Modes To Watch

- **Agent soup**: many agents running, no owner or merge gate.
- **Invisible work**: useful output exists only in a chat transcript.
- **Board theater**: a Kanban board exists but cards have no acceptance criteria.
- **Overlapping writes**: parallel agents edit the same files without worktrees.
- **No product artifact**: the process produces docs but no runnable or publishable surface.
- **Standard drift**: agents pick non-approved tech or skip ADR-001, CLEF logging, or ar/en + RTL because the MEP-VTES-001 rules never reached their card prompt.

## Output Standard

Finish each orchestration pass with:

- Board/card changes.
- Merged or pending branches.
- Tests and eval evidence.
- Blockers with owner and next action.
- New shared skill candidates.
