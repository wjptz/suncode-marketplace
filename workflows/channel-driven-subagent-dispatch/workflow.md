# Channel-Driven Sub-Agent Dispatch Workflow

---

## Core Principles

1. **Plan before code** — define the task, planning artifacts, and acceptance criteria before implementation.
2. **The main session coordinates** — the main session clarifies requirements, plans the task, dispatches workers, updates specs, commits, and finishes the work.
3. **Implementation and checking run in channel workers** — use `suncode channel spawn` for implement/check workers by default instead of host-native sub-agents.
4. **Pass context explicitly** — each claimed DAG node gets a verified context manifest; channel workers receive the manifest/content paths, not the main transcript.
5. **Keep results auditable** — use `suncode channel messages --raw` for worker events; pretty output is an operator dashboard and may truncate progress.
6. **Persist decisions** — requirements, research, plans, and review conclusions belong in task files.

---

## Suncode System

### Developer Identity

Initialize your identity on first use:

```bash
python3 ./.suncode/scripts/init_developer.py <your-name>
```

### Spec System

`.suncode/spec/` stores project engineering guidelines. Before writing code, load the package/layer specs relevant to the task:

```bash
python3 ./.suncode/scripts/get_context.py --mode packages
```

### Task System

Each task has its own directory under `.suncode/tasks/{MM-DD-name}/` with `task.json`, `prd.md`, optional `design.md`, optional `implement.md`, optional `research/`, and `implement.jsonl` / `check.jsonl`.

Common commands:

```bash
python3 ./.suncode/scripts/task.py create "<title>" [--slug <name>] [--parent <dir>]
python3 ./.suncode/scripts/task.py start <name>
python3 ./.suncode/scripts/task.py current --source
python3 ./.suncode/scripts/task.py finish
python3 ./.suncode/scripts/task.py archive <name>
python3 ./.suncode/scripts/task.py validate <name>
```

### Channel System

Channels are the worker collaboration and event-audit layer. Use `--ephemeral` for temporary implementation/check channels. Use `--type forum` for durable discussion boards; a `thread` is an item inside a forum.

Stable worker handles:

- `implement` — implementation worker
- `check` — default check worker
- `check-cc` — Claude check worker
- `check-cx` — Codex check worker

---

<!--
  WORKFLOW-STATE BREADCRUMB CONTRACT

  [workflow-state:STATUS] blocks are the single source for per-turn prompt injection.
  Do not delete tags or change tag syntax. The body can change; parsers should not.
-->

## Phase Index

```
Phase 1: Plan    -> classify, get task-creation consent, then write planning artifacts
Phase 2: Execute -> implement/check through suncode channel workers
Phase 3: Finish  -> verify, update spec, commit, and wrap up
```

### Request Triage

- Simple conversation or small task: ask only whether this turn should create a Suncode task. If the user says no, skip Suncode for this session.
- Complex task: ask whether you may create a Suncode task and enter planning. If the user says no, do not do broad inline implementation.
- User approval to create a task is not approval to start implementation. Implementation waits until artifacts are reviewed and `task.py start` has run.

### Planning Artifacts

- `prd.md` — requirements, constraints, and acceptance criteria.
- `design.md` — technical design for complex tasks.
- `implement.md` — human-readable implementation view and legacy serial fallback.
- `execution.json` — canonical dependency DAG shared by channel and inline executors.
- `implement.jsonl` / `check.jsonl` — worker context manifests. Put spec and research files here, not code files.

Lightweight tasks may keep PRD-only human documentation but use a one-node DAG when enabled. Complex tasks require a reviewed dependency-aware `execution.json` when policy requires it.

### Parent / Child Task Trees

Use a parent task when one request contains several independently verifiable deliverables. Child tasks own deliverables that can be planned, implemented, checked, and archived independently. Parent/child structure is not a dependency system; dependencies must be written in the child `prd.md` / `implement.md`.

[workflow-state:no_task]
No active task. First classify the current turn and ask for task-creation consent before creating any Suncode task.
Simple conversation / small task: ask only whether this turn should create a Suncode task. If the user says no, skip Suncode for this session.
Complex task: ask the user if you can create a Suncode task and enter the planning phase. If the user says no, explain, clarify scope, or suggest a smaller split.
[/workflow-state:no_task]

### Phase 1: Plan

- 1.0 Create task `[required · once]`
- 1.1 Requirement exploration `[required · repeatable]`
- 1.2 Research `[optional · repeatable]`
- 1.3 Configure context `[required · once]`  (sub-agent-dispatch platforms only)
- 1.4 Plan execution DAG `[required · once when execution.dag.enabled]`
- 1.5 Activate task `[required · once]`
- 1.6 Completion criteria

[workflow-state:planning]
Load `suncode-brainstorm`; stay in planning.
Lightweight: `prd.md` can be enough. Complex: finish `prd.md`, `design.md`, and `implement.md`; ask for review before `task.py start`.
Execution DAG: DAG finalization is not a per-turn action. While blocking questions remain or planning artifacts are still changing, do not scaffold or validate. Once planning converges, if `execution.json` is missing, scaffold it once, review/edit it, and validate once; if the existing plan still matches, reuse it without scaffold or validation; if a material planning change affects deliverables, dependencies, scopes/resources, or validation, edit the existing plan in place and validate once. Never automatically run `scaffold --force`. Include the reviewed DAG in the latest final planning summary before asking for subsequent implementation approval. Preserve safe independent roots/siblings so channel workers can fan out, and end all branches at an integration/check barrier.
Multi-deliverable scope: consider a parent task plus independently verifiable child tasks; parent/child position is not an execution dependency.
Channel-worker mode: curate `implement.jsonl` and `check.jsonl` as spec/research manifests before start.
[/workflow-state:planning]

[workflow-state:planning-inline]
Load `suncode-brainstorm`; stay in planning.
Lightweight: `prd.md` can be enough. Complex: finish `prd.md`, `design.md`, and `implement.md`; ask for review before `task.py start`.
Execution DAG: DAG finalization is not a per-turn action. While blocking questions remain or planning artifacts are still changing, do not scaffold or validate. Once planning converges, if `execution.json` is missing, scaffold it once, review/edit it, and validate once; if the existing plan still matches, reuse it without scaffold or validation; if a material planning change affects deliverables, dependencies, scopes/resources, or validation, edit the existing plan in place and validate once. Never automatically run `scaffold --force`. Include the reviewed DAG in the latest final planning summary before asking for subsequent implementation approval. Inline runs the same graph with concurrency one.
Multi-deliverable scope: consider a parent task plus independently verifiable child tasks; parent/child position is not an execution dependency.
Inline mode: skip jsonl curation; each claimed node reads its manifest plus specs via `suncode-before-dev`.
[/workflow-state:planning-inline]

### Phase 2: Execute

- 2.1 Implement `[required · repeatable]`
- 2.2 Quality check `[required · repeatable]`
- 2.3 Rollback `[on demand]`

Channel-driven sub-agent dispatch is the default execution model for this workflow. The main session uses `suncode channel create`, `suncode channel spawn`, `suncode channel send`, and `suncode channel wait` to coordinate workers. Fall back to native host sub-agents only when the user explicitly asks for native dispatch or a host-only capability is required.

[workflow-state:in_progress]
Flow: validate/start DAG with `--executor channel` -> claim and spawn every safe ready node before waiting -> wait-any NodeResult -> immediately unlock successors -> final integration/check barrier -> `suncode-update-spec` -> commit (Phase 3.4) -> `/suncode:finish-work`.
Main-session coordinator: dispatch only claimed nodes, pass each envelope's `channelArgs` context files, and never wait before the selected safe fan-out is spawned. Workers execute one node and do not coordinate the graph.
Worker prompt starts with `Active task:` then `Suncode context manifest:`. Read raw result events when precision matters; record NodeResult v1 with `execution complete` and recompute ready immediately.
[/workflow-state:in_progress]

[workflow-state:in_progress-inline]
Flow: start the same DAG with `--executor inline` -> claim/execute one node from its manifest -> record NodeResult -> next ready node -> final integration/check barrier -> `suncode-update-spec` -> commit (Phase 3.4) -> `/suncode:finish-work`.
Inline implementation consumes the same graph with concurrency one; do not invent a separate task-wide serial plan.
Read the claimed manifest before editing, plus relevant specs loaded by skills without broadening node boundaries.
[/workflow-state:in_progress-inline]

### Phase 3: Finish

- 3.2 Debug retrospective `[on demand]`
- 3.3 Spec update `[required · once]`
- 3.4 Commit changes `[required · once]`
- 3.5 Wrap-up reminder

> Note: step 3.1 was folded into 2.2 (last-iteration full-scope check) and 3.4 (commit preamble). Numbering kept stable to avoid breaking external references.

[workflow-state:completed]
Code committed. Run `/suncode:finish-work`; if dirty, return to Phase 3.4 first.
[/workflow-state:completed]

---

## Rules

1. Identify the current Phase, then continue from the next step in that Phase.
2. Run steps in order inside each Phase; `[required]` steps cannot be skipped.
3. Phase 2 uses channel workers by default. Do not implement large changes directly in the main session unless the user asked for inline work or the task is small enough.
4. Worker briefs must state the active task, goal, editable scope, validation commands, and forbidden actions.
5. `suncode channel messages --raw` is the precise audit path; pretty output is only for quick status checks.
6. After a worker completes, the main session integrates the result and runs check workers when needed. Final judgment stays with the main session.

### Active Task Routing

[Claude Code, Cursor, OpenCode, codex-sub-agent, Kiro, Gemini, Qoder, CodeBuddy, Copilot, Droid, Pi]

- Planning or unclear requirements -> `suncode-brainstorm`.
- `in_progress` implementation -> `suncode channel spawn --agent implement`.
- `in_progress` quality check -> `suncode channel spawn --agent check`.
- Repeated debugging -> `suncode-break-loop`; spec updates -> `suncode-update-spec`.

[/Claude Code, Cursor, OpenCode, codex-sub-agent, Kiro, Gemini, Qoder, CodeBuddy, Copilot, Droid, Pi]

[codex-inline, Kilo, Antigravity, Devin]

- Planning or unclear requirements -> `suncode-brainstorm`.
- Before editing -> `suncode-before-dev`; after editing -> prefer a channel-driven `check` worker.
- Repeated debugging -> `suncode-break-loop`; spec updates -> `suncode-update-spec`.

[/codex-inline, Kilo, Antigravity, Devin]

---

## Phase 1: Plan

Goal: clarify requirements, get task-creation consent, and produce planning artifacts that must be reviewed before implementation.

#### 1.0 Create task `[required · once]`

Create the task directory only after task-creation consent:

```bash
python3 ./.suncode/scripts/task.py create "<task title>" --slug <name>
```

Run only `create` here. Do not also run `start`. `start` switches status to `in_progress`, which moves the breadcrumb into execution.

#### 1.1 Requirement exploration `[required · repeatable]`

Load `suncode-brainstorm` and write user requirements into `prd.md`. Complex tasks also need `design.md` and `implement.md`.

Requirements:

- Ask one question at a time.
- Prefer researching over asking for information that can be discovered.
- Update task artifacts immediately when requirements change.
- Split broad work into parent task + child tasks.
- Keep `prd.md` focused on requirements and acceptance criteria, not implementation checklists.

#### 1.2 Research `[optional · repeatable]`

When research is needed, write results to `{TASK_DIR}/research/`. Research files must be usable by later workers.

#### 1.3 Configure context `[required · once]`

Curate worker context manifests:

- `implement.jsonl` — specs and research needed by the implementation worker.
- `check.jsonl` — quality specs, test specs, and research needed by the check worker.

Do not put code files in jsonl. Workers read code during execution.

#### 1.4 Plan execution DAG `[required · once when execution.dag.enabled]`

This is a planning-finalization gate, not a per-turn action. Enter it only after blocking questions are empty and the PRD/design/implementation view has converged.

- If `execution.json` is missing, scaffold it once, review/edit the graph, validate once, and show it in the final planning summary.
- If the existing plan still matches, reuse it without scaffold or validation.
- If a material planning change affects the DAG, edit affected nodes and edges in place, validate once, present the changed graph in a new final planning summary, and wait for fresh approval.

Never automatically run `scaffold --force`.

```bash
python3 ./.suncode/scripts/task.py execution scaffold <task-dir>  # missing plan only
python3 ./.suncode/scripts/task.py execution validate <task-dir>  # new or materially changed plan only
python3 ./.suncode/scripts/task.py execution show <task-dir> --json
```

Review the scaffold. Complex tasks must preserve real independent roots/siblings, declare read/write scopes and resource locks, attach validation/minimal context to every node, keep shared-worktree checks read-only, and feed every branch into a final integration/check barrier.

#### 1.5 Activate task `[required · once]`

Do not re-run `execution validate` here for an unchanged reviewed DAG. If artifacts changed materially after the final summary, return to Phase 1.4, update and validate once, present the new summary, and wait for fresh approval.

After the user approves the latest summary, start the task:

```bash
python3 ./.suncode/scripts/task.py start <task-dir>
```

#### 1.6 Completion criteria

| Condition | Required |
| --- | :---: |
| `prd.md` exists | yes |
| user confirms task should enter implementation | yes |
| `task.py start` has run | yes |
| `design.md` exists for complex tasks | yes |
| `implement.md` exists for complex tasks | yes |
| `execution.json` validates when DAG is enabled; complex branches feed a final integration/check barrier | yes |
| `implement.jsonl` and `check.jsonl` each contain at least one real curated entry (seed row does not count) | yes |

Runtime consumers tolerate missing or seed-only manifests for compatibility, but that tolerance is not a planning-ready state.

---

## Phase 2: Execute

Goal: the main session turns reviewed planning artifacts into checked code through channel workers.

#### 2.1 Implement `[required · repeatable]`

The main session is the only coordinator:

```bash
TASK=.suncode/tasks/<active-task>
python3 ./.suncode/scripts/task.py execution start-run "$TASK" --executor channel --json
python3 ./.suncode/scripts/task.py execution ready "$TASK" --json
```

For every node in `selected`, call `execution claim "$TASK" <node-id> --json` **before waiting**. Create one channel for the run, then spawn every claimed node using the agent matching `dispatch.role`, the two `--context-file` values in `dispatch.channelArgs`, and `dispatch.prompt` as its brief. Use the node ID as the worker handle. Do not reassemble task-wide JSONL/PRD/design/implement context: the manifest is the deterministic, bounded, redacted source of truth.

After all selected workers are spawned, wait without `--all` so the first completion is processed immediately. Read that worker's raw result, store NodeResult v1, call `execution complete`, and immediately call `execution ready` again. Spawn newly unlocked successors while older siblings keep running. This is wait-any; waiting for the full batch discards the DAG's parallelism.

On interruption, run `execution recover`. Only idempotent nodes retry automatically. Shared-worktree conflicts are serialized by the runtime. Native sub-agent fallback is allowed only when the user explicitly asks for it or a host-only capability is required.

Inline fallback starts the same graph with `--executor inline`, claims one node at a time, verifies its manifest, loads `suncode-before-dev` when coding, and records NodeResult v1. It does not create a separate serial plan.

#### 2.2 Quality check `[required · repeatable]`

Independent check nodes are ordinary ready nodes and may fan out across providers when their scopes/resources are safe. In a shared worktree they are read-only: each returns structured findings and validation evidence through NodeResult v1. They do not directly fix code.

Aggregate findings into one dependent `fix` node with an explicit write scope, then run another check/integration node. The final barrier must cover all implementation branches and perform full-scope spec, lint, type-check, test, and cross-layer validation before completion.

#### 2.3 Rollback `[on demand]`

- If check finds a PRD defect -> return to Phase 1, fix artifacts, then execute again.
- If an implement worker goes off-track -> narrow the brief, redispatch, or revert that work.
- If more research is needed -> write to `{TASK_DIR}/research/`, then redispatch.

---

## Phase 3: Finish

Goal: verify quality, capture lessons, and commit the work.

#### 3.2 Debug retrospective `[on demand]`

If the same class of issue recurred, load `suncode-break-loop` and record root cause plus prevention.

#### 3.3 Spec update `[required · once]`

Load `suncode-update-spec` and decide whether new patterns, pitfalls, or technical decisions should be written back to `.suncode/spec/`.

#### 3.4 Commit changes `[required · once]`

**Spec-sync preamble**: before drafting commits, ask: did this task fix a bug or surface non-obvious knowledge that should land in `.suncode/spec/` so future-you (or future-AI) doesn't repeat the mistake? If yes, return to Phase 3.3 first — spec writes belong in the same task's commit batch, not as a forgotten follow-up.

The main session commits work changes. Before committing, separate AI-edited files from unknown dirty files.

```bash
git status --porcelain
git log --oneline -5
```

Do not amend. Do not push.

#### 3.5 Wrap-up reminder

After committing, remind the user to run `/suncode:finish-work` to archive the task and record the session.

---

## Customizing Suncode

This workflow is customized through `.suncode/workflow.md`. Scripts parse tags and headings; they do not store fallback prose.

### Change a step

Edit the corresponding Phase 1 / 2 / 3 step body.

### Change per-turn prompt text

Edit the body of the matching `[workflow-state:STATUS]` block. Do not change tag names or syntax.

### Add a custom status

Add:

```text
[workflow-state:my-status]
...
[/workflow-state:my-status]
```

A lifecycle hook or script must write `task.json.status` to that value, otherwise the block is never read.
