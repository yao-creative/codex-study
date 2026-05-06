# Minimal PRD: Recreate the Planning + Execution Loop

## 1. Objective
Build the smallest viable planning loop that supports:
- a model-emitted checklist plan (`pending`, `in_progress`, `completed`)
- tool-driven execution work
- background/child-agent messaging with wait semantics
- deterministic turn lifecycle and resumable pending work

This is a runtime orchestration PRD, not a UI redesign.

## 2. Non-Goals
- No DAG planner or topological dependency solver.
- No advanced optimization/re-planning heuristics.
- No multi-provider abstraction beyond one model backend.
- No full plugin marketplace or dynamic tool discovery for v1.

## 3. Minimal User Stories
1. As a user, I can see a structured plan update while the agent works.
2. As a user, I can get progress even when the agent must wait for external events.
3. As a user, I can continue work after interruptions without losing pending tasks.
4. As a user, I can run parallel sub-agents and wait until one/all results are available.

## 4. System Model (Minimal)
1. Turn Engine
- Maintains exactly one active turn per thread.
- Starts a turn when new user input or pending work exists.
- Emits `TurnStarted`, `TurnCompleted`, `TurnAborted`.

2. Planning Channel (Checklist)
- `update_plan` tool accepts an ordered list of steps with statuses.
- Runtime emits `PlanUpdate` event.
- Plan is observable state, not internal scheduler input.

3. Execution Channel (Tools)
- Function tool dispatch with per-tool handlers.
- Handler returns structured output + optional event emissions.
- Track tool-call count and per-turn pending inputs.

4. Collaboration Channel (Sub-agents)
- `spawn_agent`: create child execution context.
- `send_input`: send message to existing child.
- `wait_agent`: wait on mailbox sequence changes with timeout.
- `close_agent`: close child session.

5. Mailbox + Pending Work
- FIFO mailbox queue for inter-agent messages.
- Monotonic sequence counter with watcher/subscription.
- Pending-work trigger starts a turn when idle.

## 5. Required Data Structures
1. `PlanItem`
- `step: string`
- `status: enum { pending, in_progress, completed }`

2. `UpdatePlanArgs`
- `explanation?: string`
- `plan: PlanItem[]`

3. `TurnState`
- `pending_input: ResponseInputItem[]`
- `pending_user_input: map<id, resolver>`
- `pending_approvals: map<id, resolver>`
- `mailbox_delivery_phase: enum { current_turn, next_turn }`
- `tool_calls: number`

4. `Mailbox`
- `queue: deque<InterAgentMessage>`
- `next_seq: u64`
- `watch_seq: subscriber<u64>`

5. `AgentRegistry`
- `agents: map<agent_id, metadata>`
- `spawn_count / max_threads`

## 6. Event Contract (Minimal)
1. Turn lifecycle
- `turn/started`
- `turn/completed`
- `turn/aborted`

2. Planning
- `turn/plan/updated` (full checklist snapshot)

3. Collaboration lifecycle
- `collab/spawn/begin`, `collab/spawn/end`
- `collab/wait/begin`, `collab/wait/end`
- `collab/message/begin`, `collab/message/end`
- `collab/close/begin`, `collab/close/end`

4. Optional user-input handshake (if enabled)
- `tool/request_user_input`
- `tool/request_user_input_response`

## 7. Core Runtime Rules
1. Single active turn lock
- If a turn is active, enqueue work; do not start another.

2. Pending-work wakeup
- If idle and (`pending_input` not empty OR mailbox has `trigger_turn` message), start a regular turn.

3. `update_plan` semantics
- Accept and emit checklist updates.
- Never treat checklist as an executable dependency graph.

4. `wait_agent` semantics
- Fast path: if mailbox has pending items, return immediately.
- Else wait for sequence change until timeout.
- Return explicit `{ timed_out: bool, message }`.

5. Abort semantics
- Cancel active tasks.
- Clear pending resolvers safely.
- Re-check pending work and restart turn if appropriate.

## 8. API Surface (Minimal)
1. Tool: `update_plan(args: UpdatePlanArgs) -> {}`
2. Tool: `spawn_agent(args) -> { agent_id }`
3. Tool: `send_input(args) -> { accepted: bool }`
4. Tool: `wait_agent(args: { timeout_ms? }) -> { message, timed_out }`
5. Tool: `close_agent(args) -> { closed: bool }`

## 9. Persistence (Minimal)
- Required: event log and thread id continuity.
- Optional v1: full plan history snapshots.
- Optional v1: goal accounting tables.

## 10. Failure Handling (Must Have)
1. Missing pending resolver on response: log warning, safe no-op.
2. Wait timeout: explicit timeout result, no silent retries.
3. Child not found: return structured error in collab end event.
4. Turn transition during wait: resolve waiters as canceled/empty.

## 11. Acceptance Criteria
1. A turn can emit at least 2 `update_plan` snapshots and complete normally.
2. `wait_agent` returns immediately when mailbox already has entries.
3. `wait_agent` times out correctly when no new mailbox sequence arrives.
4. Pending mailbox message with `trigger_turn=true` starts a turn when idle.
5. Aborting an active turn does not lose queued pending work.

## 12. Implementation Plan (Smallest Sequence)
1. Implement core `TurnState`, mailbox, and active-turn lock.
2. Implement event bus + turn lifecycle events.
3. Add `update_plan` tool and event emission.
4. Add sub-agent registry + `spawn_agent`/`send_input`/`close_agent`.
5. Add `wait_agent` with seq-watch timeout behavior.
6. Add pending-work scheduler hook for idle wakeups.
7. Add tests for turn lock, plan emission, mailbox wait, and abort recovery.

## 13. Test Matrix (Minimal)
1. Unit
- `update_plan` argument parse/validation.
- mailbox seq increment + watcher wake.
- wait timeout clamping/validation.

2. Integration
- end-to-end turn with plan update -> tool call -> completion.
- spawn child -> send message -> wait -> observe completion.
- abort turn during pending wait -> resume with queued work.

## 14. Open Decisions
1. Should `wait_agent` wait for any child or specific target ids by default?
2. Should plan updates be full snapshots only, or support diffs?
3. Should collaboration events be persisted as first-class timeline items?

## 15. CLI Integration (Minimal)
1. Command loop responsibilities
- Submit user text as a new turn request.
- Stream runtime events and render incrementally.
- Forward control actions (`interrupt`, `approve`, `deny`, `user_input_response`) back to runtime.

2. Required CLI-rendered events
- Turn lifecycle: show start/complete/abort status.
- Plan updates: render checklist snapshot on each `turn/plan/updated`.
- Collaboration events: render begin/end rows for spawn/send/wait/close.
- Tool request events: pause prompt and collect user choice/input when required.

3. Input/response handshake
- On `tool/request_user_input`, CLI blocks current prompt, displays choices, captures response, and submits `tool/request_user_input_response`.
- On approval requests, CLI captures allow/deny and submits decision with the matching request id.

4. Minimal UX behavior
- Keep one active prompt owner (runtime or user) to avoid interleaved stdin races.
- Preserve event order by event timestamp/sequence from the runtime stream.
- After abort/interrupt, immediately return terminal control to user while background cleanup completes.

5. Acceptance criteria (CLI)
1. User can submit a prompt and see live turn status transitions.
2. `update_plan` events visibly update checklist output in the same session.
3. `wait_agent` start/end is visible and timeout is clearly surfaced.
4. A `request_user_input` event can be answered from CLI and unblocks execution.
