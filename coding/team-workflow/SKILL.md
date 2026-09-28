---
name: team-workflow
description: Governed multi-subagent execution workflow (worker + review loop over a plan). Use when the user explicitly requests team workflow, multi-agent coordination ("走多 agent 流程", "team workflow", "用 subagent 团队"), or approves governed execution for high-risk, architectural, cross-module tasks. In OpenSpec projects, layers team execution onto an approved change.
---

# Team Workflow

Governed pipeline for high-risk or architectural changes. The parent keeps user alignment, routing, and final acceptance; subagents execute. Dispatch mechanics (async launch, steer/resume, evidence, isolation) follow the `pi-subagents` skill — do not restate them here. Plan artifacts, task state, and archive gates follow the `openspec-*` skills — do not restate those either.

**Entry condition**: delegation requires an explicit user request ("走多 agent 流程", "team workflow", "用 subagent 团队") or an applicable project/user instruction that mandates team execution. A project rule naming critical paths is such authorization. Otherwise default to solo execution (direct execution or solo `/opsx-apply`); risk alone is a reason to **recommend** team workflow and ask for approval, not to launch agents. State the matched risk when recommending it:
- A modified contract has material compatibility, rollback, persistence, or security risk. A MODIFIED/RENAMED header alone is not enough.
- The change touches a critical path named by the project, even when the project recommends rather than mandates team execution.
- The design introduces a hard-to-reverse decision (for example, "ADR required").
- Existing test assertions must change because observable behavior changes; a test file path alone is not a risk signal.
When the risk is uncertain, explain it and ask rather than delegating by default. When none of these signals match, use solo execution unless the user explicitly requests the team.

## Roster

- `scout` — read-only recon: affected files, dependencies, blast radius. Planning phase only; never during apply.
- `worker` — the single writer during implementation; retain one `worker` session per worktree across slices. The parent may take over a finished worker's worktree only for a bounded cleanup under the review rule below; never write concurrently.
- `reviewer` — fresh-context adversarial review of the worker's diff and evidence.
- `tester` — conditional role, only in the escalation below; writes tests blind to the implementation.

## Pipeline

### 1. Plan source
- **OpenSpec project** (`openspec/` root present): the plan is the selected change's proposal/specs/design/tasks artifacts. The user's apply request permits execution but artifact completion and `openspec validate` prove neither consistency nor acceptance. Do not re-present the plan for approval when it is coherent. Optional recon: the parent may dispatch `scout` while artifacts are being drafted, never during apply.
- **Otherwise**: conditional `scout` recon when the parent doesn't already know the area; then present a bounded plan — objective, target files/seams, interface contracts, verification commands — and get user approval **before any write**.

**Plan-readiness gate (before any writer):** compare required outputs and constraints with the tasks, documentation obligations, and their verification commands. For each consequential requirement, identify the check that would fail if it were broken, at the required boundary (unit, integration, process, or external interaction); a nearby simulation does not establish a stronger claim. Check that no verification forbids content another artifact requires. For non-OpenSpec work, compare the approved plan with its acceptance checks. A material contradiction or unresolved product choice pauses execution: route an OpenSpec conflict to `/opsx-update` or the user; do not let the worker resolve it by deleting required content or treating a CLI `ready` state as semantic proof.

### 2. Execute
Dispatch granularity: one `worker` session per change — or per lane/worktree when the change has separable seams (see the `pi-subagents` multi-lane orchestration reference). Divide long work into verifiable slices at behavior or integration seams. At each slice boundary, leave a brief checkpoint and continue in the same session unless an escalation needs the parent; do not require a new run or approval for every slice. Slices are progress checkpoints, not extra planning artifacts or substitutes for task verification.

For a **new** worker, give the change name, contextFiles, `context` and `operationGuidance` (all from `openspec instructions apply --change "<name>" --json`), assigned tasks, constraints/non-goals, verification commands, and — for documentation, config, or content-producing tasks — **fact sources**: a `fact → authoritative source path` list the parent gathers before dispatch (a two-minute recon; cite the source files or external docs the output must align with). Point to authoritative files and quote only critical constraints; do not paste their full contents again in the handoff. A new worker must read the relevant originals, not rely on the handoff's shorthand.

The worker runs this loop inside its own session, per behavior task:

1. **red, when adding behavior** — write the tests named by the task's `Verification:` clause and run them before implementation. The failure must expose the missing behavior. A broken fixture, timing assumption, or harness error is not red evidence: repair the test and rerun it before recording red.
2. **green** — implement until the task's verification command passes in full. A test preserving existing behavior may remain green; name the regression it would catch instead of manufacturing a failure.
3. **record** — mark the task `- [x]` only after its full behavior and verification are complete. Include valid red/green evidence when applicable, or green evidence plus the regression failure mode for preserved behavior.

Non-behavior tasks (docs, packaging, examples) skip red, but their record MUST carry one of two evidence forms: (a) mechanical verification — command + output excerpt (pack, typecheck, tests); or (b) fact check — a **fact → source mapping table**, each collection-type fact (enum values, path lists, identifiers, versions) citing its authoritative source path and line. A prose "已核对，一致" declaration is not evidence.

Escalate to the parent when the spec or task checks are ambiguous or self-contradictory, or the same failure survives two genuine fix attempts. Also request a checkpoint after roughly 10 minutes without new passing evidence or a resolved hypothesis: report the current failure, attempts, files touched, next hypothesis, and whether the task is blocked. This is a soft progress trigger, not a deadline for a healthy long-running check; the parent decides whether to continue, change the approach, or pause. Near a hard run deadline, checkpoint before timeout rather than starting another speculative debug cycle. The parent routes contract problems to `/opsx-update` or the user; it never settles a product choice by reinterpreting a check. Scope beyond the spec is never absorbed.

**Evidence** — per task, report the command, relevant failure or passing excerpt, and why the result demonstrates the named behavior. Distinguish a genuine pre-implementation failure from a test-authoring error; do not claim an invalid red as proof. Tests are edited only to match the spec scenario better, never to fit the implementation. At each slice boundary, report completed and still-open tasks, the exact checks and outcomes, and remaining uncertainty. Do not create a second task ledger or paste full logs; task checkboxes remain the progress authority.

**Continuation handoff** — when resuming the *same* worker, send only what changed since its last run: current task/slice, newly observed findings, exact files or evidence to revisit, and the next verification. Do not resend the initial assignment or whole plan; a checkpoint is an index back to the authoritative artifacts, not a replacement for them. If its previous context is unavailable and a new worker must take over, use a new-worker handoff plus the checkpoint and independently verify the current tree before resuming writes.

### 3. Review–fix loop (quality gate)
Dispatch `reviewer` in fresh context with: the change's contextFiles and authoritative artifacts, the full changed-file inventory (including untracked additions that ordinary `git diff` omits), how to inspect the current diff and new files, the worker's evidence as claims to audit, and the highest-risk boundaries. Do not send full logs or paraphrase the contract as a substitute for reading it. The reviewer independently checks originals and the current tree, prioritizing irreversible decisions, external inputs, persistence, compatibility, and process boundaries before general code quality. For a targeted re-review by the *same* reviewer, send only the finding, changed files, and affected checks since its previous review. Review on two axes — report them separately, never merge or rerank findings across axes (one axis must not mask the other):

**Spec axis** — against the change artifacts. Quote the spec line for every finding:
- Requirements/scenarios missing or only partially implemented.
- Behavior in the diff nobody asked for (scope creep); specified behavior silently narrowed.
- **Test validity**: every delta scenario has a test; assertions check behavior, not implementation details; no `.skip`/`.only`, swallowed assertions, or test files the runner never picks up.
- **Sensitivity and proof strength**: for each core behavior, name the test that would go red if you broke it and the failure it exercises. Distinguish implemented, behaviorally tested, and verified at the promised boundary. A unit fake cannot alone prove a real process or network guarantee; report missing boundary evidence rather than promoting a nearby test into proof. A suite no obvious bug can break proves nothing.
- **Evidence audit**: when red is applicable, it predates implementation, fails for the missing behavior rather than a fixture bug, and matches the task's `Verification:` clause. For preserved behavior, verify the stated regression failure mode instead.
- **Planning consistency**: identify any required output that a task or its check rules out; quote the conflicting artifact lines. Do not downgrade an unresolved contradiction to a style note.
- Requirement edges and project hard constraints that no scenario encodes (dependencies, compatibility, side effects).

**Standards axis** — against the repo's documented standards (AGENTS.md etc.), plus the smell baseline below. A documented repo standard always overrides the baseline; baseline smells are labelled judgement calls, never hard violations; skip anything tooling already enforces.

- Mysterious Name → rename
- Duplicated Code → extract the shared shape
- Feature Envy → move the method onto the data
- Data Clumps → bundle into a type
- Primitive Obsession → give the concept a small type
- Repeated Switches → polymorphism or one shared map
- Shotgun Surgery → gather what changes together
- Divergent Change → split so each module changes for one reason
- Speculative Generality → delete; inline until a real need shows
- Message Chains → hide the walk behind one method
- Middle Man → cut the delegate
- Refused Bequest → composition over inheritance

Classify each finding separately from its severity: **contract blocker** (ambiguous or conflicting required behavior; pause for artifact update or user decision), **implementation defect** (fix now, including a required guarantee with inadequate proof), or **optional note** (may remain without changing the contract). A positive verdict with notes MUST NOT contain an unresolved contract blocker or a required boundary left unverified; a user decision that changes required behavior goes into the plan before work resumes. Normally resume the **same** `worker` for implementation defects and re-verification, then the **same** `reviewer` for targeted confirmation. For a clearly behavior-preserving, localized cleanup only, after the worker has finished and relinquished the worktree, the parent may make the patch and run focused checks; request reviewer confirmation if the edit affects behavior, security, a contract, or the reviewer asks for it. Do not force a fix or another review round solely for optional style notes. If the same dispute survives two rounds, stop and present both positions to the user for arbitration.

### 4. Acceptance

Gate order — check cheap objective defects before deep review, then validate the final tree:

1. **Triage** — the parent checks checkbox completeness, missing artifacts, and focused verification on the current tree (targeted tests/typecheck as applicable). Objective defects go straight back to the worker before reviewer dispatch. Do not duplicate scenario coverage, sensitivity analysis, or evidence auditing in parent context; give candidate leads to the reviewer. A mechanical pass never discharges the reviewer.
2. **Review** — run the section-3 loop. Contract blockers stop it for a plan update or user decision, not an `OK with notes` verdict.
3. **Final-tree gate** — after the last edit or reviewer fix, the parent independently reruns the project's required mechanical checks (including its full suite when required) on that exact tree and records the result. Any later edit invalidates affected results. For OpenSpec team changes, run `openspec validate <change> --strict` **and complete `/opsx-verify`** before calling the change accepted; team review does not substitute for coherence verification. Avoid repeating an expensive full gate before review when focused checks can triage the diff. A commit or checked task list is not acceptance.
4. Suggest `/opsx-archive` only when the project's archive gates are met and no contract blocker remains; never archive automatically.

Deliver as accepted only when the reviewer passes, the worker's evidence is on record, and the final-tree gate including any required coherence verification passes. Otherwise report implementation progress and the outstanding gate, not completion. Optional notes may remain; changed required behavior must be reconciled in the plan rather than waived by a positive verdict. In solo `/opsx-apply`, do not skip the project's `/opsx-verify` coherence check.

**Measure the workflow, not just the result** — use available run metadata and the final handoff to note wall time, tokens/cost when available, timeouts or stalled cycles, review findings by class, and any deferred verification. Do not invent totals for parent work or create a mandatory reporting artifact. Compare these with later changes alongside escaped defects; lower cost alone is not evidence of improvement.

## Escalations (optional, on demand)

- **tester/implementer split** — only when a task is too large for one context, or the confirmation-bias cost is high (core path). The parent first pins the interface contract (files, export names, signatures, error types), extracted from design.md if present, otherwise derived and backfilled via `/opsx-update`. `tester` is dispatched blind to the implementation; red evidence is recorded **before** the implementation lands. The checkbox is marked by whichever side runs full verification.
- Architectural fork with multiple defensible branches → OpenSpec: resolve in the design artifact via `/opsx-update`, not mid-apply. Otherwise: consult `oracle` (bounded read-only critique) before finalizing the plan.
- Lane infrastructure failure (launch/tooling/prompt runtime) → stop, report run/worktree state; no silent fallback to direct execution.
