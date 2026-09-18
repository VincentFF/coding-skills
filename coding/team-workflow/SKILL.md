---
name: team-workflow
description: Governed multi-subagent execution workflow (recon → plan → worker → review loop). Use when the user explicitly requests team workflow, multi-agent coordination ("走多 agent 流程", "team workflow", "用 subagent 团队"), or approves governed execution for high-risk, architectural, cross-module tasks.
---

# Team Workflow

Governed pipeline for high-risk or architectural changes. The parent keeps user alignment, planning, routing, and final acceptance; subagents execute. Dispatch mechanics (async launch, steer/resume, evidence, isolation) follow the `pi-subagents` skill — do not restate them here.

**Entry condition**: delegation is authorized (explicit user request or an approved proposal). Not authorized → work directly, do not enter this pipeline.

## Roster

- `scout` — read-only recon: affected files, dependencies, blast radius. Read-only holds by instruction; restrict tools at dispatch when the guarantee must be hard.
- `worker` — the single writer; all edits go through one `worker` session per worktree.
- `reviewer` — fresh-context adversarial review of the worker's diff.

## Pipeline

### 1. Recon (conditional)
Skip when the parent already knows the affected area. Otherwise dispatch `scout` read-only; use its summary to size the blast radius and shape the plan.

### 2. Plan agreement
Present a bounded plan — objective, target files/seams, interface contracts, verification commands — and get user approval **before any write**. Unapproved decisions surfaced later are relayed to the user, never decided by the parent.

### 3. Execute
Dispatch one `worker` with the approved spec: objective, target files, constraints/non-goals, acceptance criteria, verification commands. For cross-module work with separable seams, fan out one `worker` per lane/worktree — see the `pi-subagents` multi-lane orchestration reference.

### 4. Review–fix loop (quality gate)
Dispatch `reviewer` in fresh context with the spec + diff. On actionable findings: resume the **same** `worker` run (preserved context), which fixes and re-verifies; then resume the **same** `reviewer` so it can confirm its findings were addressed. If the same dispute survives two rounds, stop and present both positions to the user for arbitration. Deliver only when the reviewer passes — or the user explicitly accepts residual issues — **and** the worker's verification output is on record.

## Escalations (optional, on demand)

- Irreversible or architectural fork with multiple defensible branches → consult `oracle` (bounded read-only critique) before finalizing the plan.
- Lane infrastructure failure (launch/tooling/prompt runtime) → stop, report run/worktree state; no silent fallback to direct execution.
