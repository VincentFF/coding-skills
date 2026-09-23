---
name: team-workflow
description: Governed multi-subagent execution workflow (worker + review loop over a plan). Use when the user explicitly requests team workflow, multi-agent coordination ("走多 agent 流程", "team workflow", "用 subagent 团队"), or approves governed execution for high-risk, architectural, cross-module tasks. In OpenSpec projects, layers team execution onto an approved change.
---

# Team Workflow

Governed pipeline for high-risk or architectural changes. The parent keeps user alignment, routing, and final acceptance; subagents execute. Dispatch mechanics (async launch, steer/resume, evidence, isolation) follow the `pi-subagents` skill — do not restate them here. Plan artifacts, task state, and archive gates follow the `openspec-*` skills — do not restate those either.

**Entry condition**: delegation is authorized (explicit user request or an approved proposal). Not authorized → work directly, do not enter this pipeline.

## Roster

- `scout` — read-only recon: affected files, dependencies, blast radius. Planning phase only; never during apply.
- `worker` — the single writer; all edits go through one `worker` session per worktree.
- `reviewer` — fresh-context adversarial review of the worker's diff and evidence.
- `tester` — conditional role, only in the escalation below; writes tests blind to the implementation.

## Pipeline

### 1. Plan source
- **OpenSpec project** (`openspec/` root present): the plan is an approved change — its proposal/specs/design/tasks artifacts, already reviewed by the user. Do not re-present a plan for approval; go to Execute. Optional recon: the parent may dispatch `scout` while artifacts are being drafted, never during apply.
- **Otherwise**: conditional `scout` recon when the parent doesn't already know the area; then present a bounded plan — objective, target files/seams, interface contracts, verification commands — and get user approval **before any write**.

### 2. Execute
Dispatch granularity: one `worker` session per change — or per lane/worktree when the change has separable seams (see the `pi-subagents` multi-lane orchestration reference). Give the worker: change name, contextFiles, `context` and `operationGuidance` (all from `openspec instructions apply --change "<name>" --json`), assigned tasks, constraints/non-goals, verification commands, and — for documentation, config, or content-producing tasks — **fact sources**: a `fact → authoritative source path` list the parent gathers before dispatch (a two-minute recon; cite the source files or external docs the output must align with).

The worker runs this loop inside its own session, per behavior task:

1. **red** — write the tests named by the task's 「验证：」 clause; run them; capture the failing output.
2. **green** — implement until the task's verification command passes in full.
3. **record** — mark the task `- [x]` and include red/green evidence in the report.

Non-behavior tasks (docs, packaging, examples) skip red, but their record MUST carry one of two evidence forms: (a) mechanical verification — command + output excerpt (pack, typecheck, tests); or (b) fact check — a **fact → source mapping table**, each collection-type fact (enum values, path lists, identifiers, versions) citing its authoritative source path and line. A prose "已核对，一致" declaration is not evidence.

Escalate to the parent only when the spec is ambiguous or self-contradictory, or the same failure survives two genuine fix attempts. The parent routes spec problems to `/opsx-update` or the user; it never decides technical questions itself. Scope beyond the spec is never absorbed.

**Evidence** — per task, the worker's report carries: the red command + failing output excerpt, and the green command + passing output excerpt. Tests are edited only to match the spec scenario better, never to fit the implementation. A test that has never been red proves nothing.

### 3. Review–fix loop (quality gate)
Dispatch `reviewer` in fresh context with: the change's contextFiles, the full diff, and the worker's evidence. Review on two axes — report them separately, never merge or rerank findings across axes (one axis must not mask the other):

**Spec axis** — against the change artifacts. Quote the spec line for every finding:
- Requirements/scenarios missing or only partially implemented.
- Behavior in the diff nobody asked for (scope creep); specified behavior silently narrowed.
- **Test validity**: every delta scenario has a test; assertions check behavior, not implementation details; no `.skip`/`.only`, swallowed assertions, or test files the runner never picks up.
- **Sensitivity**: for each core behavior, name the test that would go red if you broke it. A suite no obvious bug can break proves nothing.
- **Evidence audit**: red evidence predates the implementation and matches each task's 「验证：」 clause.
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

On actionable findings: resume the **same** `worker` run (preserved context), which fixes and re-verifies; then resume the **same** `reviewer` so it can confirm its findings were addressed. If the same dispute survives two rounds, stop and present both positions to the user for arbitration.

### 4. Acceptance

Gate order — cheap mechanical checks before expensive judgment:

1. **Triage** — the parent runs `/opsx-verify` in its own context (no subagent). Objective findings (unchecked tasks, missing artifacts) go straight back to the worker; do not dispatch the reviewer yet. Heuristic findings (untraced requirements, uncovered scenarios) are not bounced back — they go into the reviewer dispatch as leads to confirm or dismiss. A verify pass proves nothing; it never discharges the reviewer.
2. **Review** — the section-3 loop, with verify's report included in the reviewer dispatch.
3. **Artifact validation** — `openspec validate <change> --strict`.
4. Suggest `/opsx-archive`.

Deliver only when the reviewer passes — or the user explicitly accepts residual issues — **and** the worker's verification output is on record.

Outside this workflow (solo `/opsx-apply`), `/opsx-verify` is the only implementation–artifact coherence check before archive; never skip it there.

## Escalations (optional, on demand)

- **tester/implementer split** — only when a task is too large for one context, or the confirmation-bias cost is high (core path). The parent first pins the interface contract (files, export names, signatures, error types), extracted from design.md if present, otherwise derived and backfilled via `/opsx-update`. `tester` is dispatched blind to the implementation; red evidence is recorded **before** the implementation lands. The checkbox is marked by whichever side runs full verification.
- Architectural fork with multiple defensible branches → OpenSpec: resolve in the design artifact via `/opsx-update`, not mid-apply. Otherwise: consult `oracle` (bounded read-only critique) before finalizing the plan.
- Lane infrastructure failure (launch/tooling/prompt runtime) → stop, report run/worktree state; no silent fallback to direct execution.
