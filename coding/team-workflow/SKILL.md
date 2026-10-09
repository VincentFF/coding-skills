---
name: team-workflow
description: Execute pi-profile-switch OpenSpec changes with a worker and an independent reviewer. Use when the user explicitly requests team workflow or multi-agent execution for this project. OpenSpec supplies the plan; the reviewer also completes opsx-verify; the parent owns final acceptance.
---

# Team Workflow

## Scope and sources

Use this skill for **pi-profile-switch OpenSpec changes** after team execution has been explicitly authorized by the user or mandated by applicable instructions. Risk alone does not authorize delegation. Outside this project, stop using this skill and select the applicable workflow; do not impose these project rules elsewhere.

The selected change's artifacts are the execution contract. Follow the project's `AGENTS.md` and `openspec/config.yaml`, the applicable `openspec-*` skills, and `pi-subagents` for delegation mechanics. Use OpenSpec's resolved paths and edit boundaries, including a selected store when applicable. Do not create a second plan, task ledger, evidence document, or workflow script merely to coordinate two agents.

## Owners

| Owner | Responsibility |
| --- | --- |
| Parent | User intent, change selection, blocking plan conflicts, dispatch, final mechanical checks, acceptance, and publication authority. |
| Worker | Implementation, task verification, and implementation checkboxes in the authoritative tasks artifact. |
| Reviewer | Read-only independent review, `/opsx-verify`, and targeted confirmation of fixes. |

Default to **one retained worker and one fresh-context reviewer per change**. Reuse their sessions for fixes and targeted re-review. Children do not delegate further. Keep one writer per working directory; the parent edits only after the worker has relinquished it.

Add workers only for independent tasks with exclusive file ownership or isolated worktrees, settled interfaces, and a concrete elapsed-time benefit. Shared activation, generation, and rollback changes are not independent merely because they have different task numbers. Follow `pi-subagents` for partitioning and integration. Do not add scout, tester, or oracle stages by default.

## 1. Start

1. Follow `/opsx-apply` to select the change, check outstanding completed changes that require archive, resolve status and scope, and obtain apply instructions. Announce the selected change. Preserve CLI-controlled blocked states; `all_done` means task progress, not acceptance.
2. Read the returned `contextFiles`, required `context`, and applicable `operationGuidance`. Obtain instructions once per change and artifact in the parent session, then reuse them in handoffs. Refresh status when needed and reread changed artifacts; cached task counts or prior file contents are not current evidence.
3. Before dispatch, reconcile required outputs, scenarios, design interfaces, Doc Impact, and task `Verification:` clauses. Check for missing acceptance criteria, checks that reject required content, and unresolved product choices. A material conflict pauses execution for `/opsx-update` or the user. A coherent apply request does not require another plan approval.
4. Assign the worker the remaining implementation tasks. Point to original sources rather than pasting artifacts or rewriting their requirements.

Do not rerun planning analysis at every task boundary. If implementation exposes a new contract conflict, return to `/opsx-update`; neither worker nor reviewer may silently narrow the contract.

## 2. Implement

The worker reads the authoritative inputs, implements within the assigned boundary, runs each task's verification, and updates its checkbox only when the complete requirement and verification pass. Follow task-specific fact sources and project documentation ownership. Report mismatches; do not duplicate existing source checklists into a separate fact table.

Use tests that expose the required behavior:
- For a bug fix, prefer a reproducing regression test before the fix.
- For new behavior, choose test-first execution when useful or explicitly required by the task.
- For preserved behavior, valid existing regression coverage is sufficient.
- For documentation or refactoring, use the task's mechanical or fact checks.

Do not require historical red logs for every task. When a task explicitly requires red evidence, preserve it and distinguish a missing-behavior failure from a broken fixture or harness. Never change assertions merely to accommodate the implementation.

Deduplicate shared verification commands when a successful run on the current tree fully covers multiple tasks. Rerun checks after relevant edits. Follow the project's required source checks, but do not run the full suite after every checkbox. Normal execution continues without mandatory slices, time-based checkpoints, or repeated user approval.

Return a compact handoff: changed files, completed/open tasks, verification commands and outcomes, and blockers. Include a failure excerpt only when it explains a blocker or required evidence. On interruption, also identify partial changes and the next check. Stop for scope changes, unresolved requirements, debugging without a new actionable hypothesis, or delegation infrastructure failure; report state rather than silently falling back to another execution mode.

## 3. Review and verify together

After the worker finishes and relinquishes the tree, dispatch a fresh-context reviewer. Supply the original contract paths, complete changed-file inventory including untracked additions, diff inspection instructions, worker verification claims, and relevant high-risk boundaries.

The reviewer independently reads the original artifacts and current implementation. **Perform independent review and `/opsx-verify` in this same pass**, following that skill's completeness, correctness, and coherence checks. Reuse the parent's apply-instruction output; do not request the same instruction template again. Reread current artifacts rather than trusting a worker summary.

Check required behavior, scope, design interfaces, valid scenario coverage, and project hard constraints. Prioritize compatibility with uncontrolled Pi behavior, launcher sequencing, instance generation/cleanup, resource filtering, and failure recovery where affected. A unit fake does not prove a promised process or external-interaction guarantee. Flag missing required boundary evidence even when the suite passes.

Inspect code and tests first, including whether scenario assertions detect the intended failure and the runner actually executes them. Run focused checks when needed to investigate a concrete concern; audit historical red evidence only when the task requires it, and do not routinely rerun the full suite. Skip generic smell catalogs and tooling-enforced style checks.

Produce **one review/verification result** using `/opsx-verify`'s dimensions and severity format. Each actionable finding cites a file/line or scenario and is classified as:
- **Contract blocker:** required behavior or artifacts conflict; parent routes to `/opsx-update` or the user.
- **Implementation defect:** implementation, documentation, or required verification is inadequate; worker fixes it.
- **Optional note:** no required behavior or project constraint is violated; no mandatory fix or review round.

Unresolved contract blockers and missing required verification block acceptance regardless of report severity or a generic positive verdict. If final-acceptance tasks remain unchecked pending parent commands, report them as pending gates; do not claim the change is accepted or archive-ready.

For implementation defects, resume the same worker with findings and affected checks, then the same reviewer with only the changed files and questions to confirm. Reopen broader review only when fixes change scope or invalidate earlier conclusions. If a disagreement needs a contract decision or further rounds produce no new evidence, escalate to the parent; optional style notes alone never force another round.

## 4. Accept and deliver

The parent checks the combined review/verification result for required coverage and unresolved blockers, without repeating the reviewer's full scenario analysis. After the last edit, independently run the final mechanical gates required by the project and change. For source changes, include `npm run check` and `npm test`; run the applicable strict OpenSpec validation. Team review alone and strict validation alone do not replace the completed `/opsx-verify` result.

Final-acceptance task checkboxes are updated by the parent only after the worker has relinquished the tree and their specified gates pass. If acceptance tasks need factual content beyond checkbox updates, assign that work before final checks. A checkbox-only completion update does not require a new code review; any later source, test, documentation, or contract edit invalidates affected results and requires the appropriate checks and reviewer follow-up. Recheck current task completeness and report the final disposition of previously pending verification gates.

Confirm proposal Doc Impact and required ADR outputs, using the project's archive rules. Suggest `/opsx-archive` only when all required tasks and gates pass and no contract blocker remains. Never archive automatically. If a gate fails, report implementation progress and the outstanding gate rather than completion.

Delivery includes the change, implemented scope, review/verification disposition, final check outcomes, and remaining optional notes. Available timing or cost metadata may be summarized; do not create a reporting artifact or invent totals.

## Handoff templates

### Worker

```text
Change, cwd, resolved artifact paths, apply context/guidance
Assigned task IDs and write boundary
Relevant project constraints and authoritative fact sources
Verification commands and completion criteria
Return: changed files, task status, checks, blockers
Stop: contract conflict, added scope, or infrastructure failure
```

### Reviewer

```text
Change, cwd, resolved artifact paths, cached apply instructions
Changed-file inventory and current diff/new-file inspection
Worker verification claims and affected high-risk boundaries
Read-only review + openspec-verify; one combined result
Return: completeness/correctness/coherence, classified findings,
checks performed, skipped checks with reasons, pending final gates
```
