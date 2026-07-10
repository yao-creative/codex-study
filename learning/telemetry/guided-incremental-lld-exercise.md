# Guided Incremental LLD Exercise: Build Your Own Agent Runtime, Storage, and Trace System

This exercise is for learning the system concept by concept.

You are not trying to clone `codex-rs` line-for-line. You are trying to rebuild the same design pressures in a smaller system, in an order that forces you to understand why each layer exists.

The target system has two layers:

1. upstream agent/runtime state machines
2. downstream storage and trace pipelines

The concrete capabilities you will grow into are:

1. run one agent session with explicit turn state machines
2. fan out live events to clients
3. append durable session history
4. keep queryable metadata in SQLite
5. write raw trace bundles and reduce them offline

If you do the stages in order, storage and trace should feel downstream of control flow, not parallel to it.

## Learning Goal

By the end, you should be able to explain:

- why state machines should exist before persistence layers
- how turn control flow produces the events that storage later records
- why append-only history and query/index state should be separate
- why live event delivery should not be the source of truth
- why raw trace capture and reduced trace state should be separate
- where a writer aggregate should own ordering guarantees
- where a reducer aggregate should own interpretation

## Constraints

Use these constraints for the exercise:

- one process only
- local disk only
- one language of your choice
- SQLite allowed
- no network
- no UI required
- write tests at each stage before adding the next layer

## The Core Domain

Your system models one `Session` that owns:

- session-lifetime state
- one active logical turn at a time
- turn-local mutable state
- an event bus projection
- durable rollout history
- optional trace capture

The session emits:

- `ResponseItem`
- `EventMsg`
- `TurnContext`
- `SessionMeta`

The session has two responsibilities:

1. produce runtime outputs in order
2. persist enough evidence for replay/debugging later

Do not add reducers, analytics, or multi-agent support in the first persistence stage. Earn those later.

## Stage 0: Spec the Smallest Useful Domain

Write a short spec before coding.

### What to define

- `SessionId`
- `RolloutItem` as a tagged union
- `SessionMeta`
- `ResponseItem`
- `EventMsg`
- `TurnContext`
- `TurnState`
- `SessionState`

### Deliverable

One page answering:

1. Which of these are model-visible?
2. Which are runtime-only?
3. Which must be replayable exactly?
4. Which can be derived later?
5. Which belong to session lifetime versus turn lifetime?

### Stop-and-check questions

- If the process crashes after generating an event but before sending it to a client, what survives?
- If the process crashes after sending it to a client but before disk write, what survives?
- Which side do you want to be authoritative?

### Expected conclusion

Control flow comes first. Persistence records the consequences of control flow. Live delivery is secondary.

## Stage 1: Session Ownership and State Buckets

Before you build persistence, define who owns mutable state.

### Goal

Split state by lifetime and mutation authority.

### Minimum owners

- `Session`
- `ActiveTurn`
- `TurnState`
- `EventBus`

Optional later:

- `Mailbox`
- `TraceContext`

### Exercise

Write the ownership tree for your mini runtime.

Example shape:

```text
Session
- active_turn
- session_state
- event_bus
- storage_handle

ActiveTurn
- task_id
- turn_context
- turn_state

TurnState
- pending_input
- pending_waits
- pending_tool_calls
- follow_up_required
```

### Stop-and-check questions

- Which component is allowed to mutate `pending_input`?
- Which component is allowed to decide a turn is complete?
- Which state must survive across turns?
- Which state must be dropped when a turn ends?

### Minimum tests

1. turn-local state is reset between turns
2. session-lifetime state survives across turns
3. only the turn owner can mutate turn-local queues

### What you should learn

- state-machine clarity starts with ownership clarity
- most runtime confusion is really ownership confusion

## Stage 2: Turn Lifecycle State Machine

Now define the main upstream machine.

### Goal

Model one logical turn as explicit control flow.

### Required states

- `Idle`
- `Running`
- `WaitingExternal`
- `Completed`
- `Aborted`

### Exercise

Implement the turn loop without any persistence yet.

You can drive it with a fake model/runtime:

- start turn
- optionally block on external input
- resume
- complete or abort

### Interfaces to design

```text
Session::start_turn(input)
Session::resume_turn(resolution)
Session::abort_turn(reason)
```

### Stop-and-check questions

- What transitions are valid from `WaitingExternal`?
- Can `Completed` transition back to `Running`?
- Is turn completion decided by the model, the tool runtime, or the session controller?

### Minimum tests

1. `Idle -> Running -> Completed`
2. `Idle -> Running -> WaitingExternal -> Running -> Completed`
3. `Idle -> Running -> Aborted`

### What you should learn

- the turn loop is the upstream producer of later storage facts
- invalid transitions should be impossible or explicit errors

## Stage 3: Pending Interaction State Machine

Add waits and resumptions.

### Goal

Teach the runtime to pause on external dependencies without losing turn identity.

### Add these concepts

- pending user input
- pending approval
- pending tool permission or generic external wait

### Exercise

Refine `WaitingExternal` into typed wait reasons. Do not persist them yet.

### Interfaces to design

```text
TurnState::register_wait(call_id, kind)
TurnState::resolve_wait(call_id, resolution)
TurnState::has_blocking_waits() -> bool
```

### Minimum tests

1. resolving an unknown wait fails cleanly
2. one turn can have multiple pending waits
3. resolving the last wait allows the turn to continue

### What you should learn

- waiting is part of turn execution, not an exceptional edge case
- correlation IDs belong in turn-local state

## Stage 4: Event Emission Contract

Only now define what the runtime emits.

### Goal

Make state transitions observable as events.

### Emit events for

- turn started
- waiting started
- waiting resolved
- item completed
- turn completed
- turn aborted

### Exercise

For each turn transition, decide:

1. which internal state changes first
2. which event is emitted second
3. which parts are model-visible versus runtime-visible

### Minimum tests

1. emitted events reflect real transitions in order
2. aborted turns emit terminal events only once
3. waiting/resume events preserve correlation IDs

### What you should learn

- events are projections of state transitions
- if event semantics are blurry, persistence semantics will also be blurry

## Stage 5: Model Sampling and Streaming State Machine

Add the core loop that turns model output into runtime actions.

### Goal

Model how one running turn consumes streamed assistant output, accumulates partial state, and decides whether follow-up work is needed.

### Add these concepts

- assistant text deltas
- item-completed boundaries
- end-of-response marker
- follow-up required flag

### Exercise

Implement a fake sampler that streams chunks and completed items into the turn.

Do not add tools yet. First make plain streaming coherent.

### Interfaces to design

```text
Session::run_sampling_pass(turn_id)
TurnState::on_text_delta(item_id, delta)
TurnState::on_item_completed(item)
TurnState::on_response_completed(end_turn)
```

### Minimum tests

1. deltas accumulate into one completed assistant item
2. `end_turn = false` reopens the same logical turn loop
3. a completed response without follow-up transitions toward stop

### What you should learn

- streamed state and completed state are different
- the turn loop needs a precise post-pass decision point

## Stage 6: Tool Execution Lifecycle

Now make the agent do work beyond text generation.

### Goal

Add the state machine for tool calls inside a running turn.

### Required states

- `ToolDiscovered`
- `ToolRunning`
- `ToolCompleted`
- `ToolFailed`

### Exercise

Teach the sampling pass to hand completed tool-call items to a tool runtime, then feed the resulting outputs back into the same logical turn.

### Interfaces to design

```text
ToolRuntime::execute(call) -> ToolResult
TurnState::register_tool_call(call_id)
TurnState::complete_tool_call(call_id, result)
```

### Stop-and-check questions

- who decides whether a tool result becomes model-visible input?
- does a tool failure abort the turn, or become a normal result item?
- when exactly does a tool call imply `follow_up_required = true`?

### Minimum tests

1. a tool call produces a tool result item and reopens follow-up
2. tool failures are represented consistently
3. multiple tool calls in one turn preserve ordering

### What you should learn

- tool execution is subordinate to the turn loop, not a separate product
- most agent complexity starts here

## Stage 7: Context Assembly and Input Queueing

Add the logic that decides what the model sees on each pass.

### Goal

Separate durable history from the model-visible request assembled for the next sampling pass.

### Add these concepts

- pending input queue
- accepted versus deferred input
- system/session context
- turn-local additions

### Exercise

Implement a request builder that turns durable history plus turn-local pending inputs into the next model request.

### Interfaces to design

```text
RequestBuilder::build(session, turn_state) -> ModelRequest
TurnState::queue_input(item)
TurnState::drain_accepted_input() -> Vec<InputItem>
```

### Minimum tests

1. queued input appears in the next model request in order
2. blocked or deferred input does not disappear
3. durable history and turn-local pending input are not conflated

### What you should learn

- request construction is an upstream design concern
- persistence stores evidence; request builders decide visibility

## Stage 8: Cancellation, Preemption, and Stop Conditions

Make the runtime robust under interruption.

### Goal

Define how turns stop when cancelled, superseded, or naturally finished.

### Add these concepts

- cancellation token or abort signal
- terminal turn states
- optional follow-up suppression
- single terminal emission rule

### Exercise

Decide which layer can interrupt a running turn and what cleanup is mandatory.

### Minimum tests

1. cancellation produces one terminal outcome
2. cancelled tools do not keep the turn alive forever
3. a completed turn cannot be resumed accidentally

### What you should learn

- good state machines are mostly about terminal behavior
- cancellation rules leak into storage if you do not define them early

## Stage 9: Multi-Agent or Mailbox Upstream Extension

Add exactly one upstream extension before touching storage.

Choose one:

1. mailbox delivery phase
2. child-agent spawn/send/wait/close lifecycle

### Rule

Pick one. Do not do both at once.

### If you choose mailbox delivery

Model:

- `CurrentTurn`
- `NextTurn`

Question:

- when should a queued message join the current turn versus the next one?

### If you choose multi-agent collaboration

Model:

- `Begin`
- `Executing`
- `Completed`
- `Failed`

Question:

- which side owns the child result: the child runtime, the parent runtime, or the event projection?

### Minimum tests

1. mailbox or child-agent events preserve order
2. extension state does not corrupt the base turn lifecycle
3. a failed extension call still leaves the turn state coherent

### What you should learn

- storage complexity comes from upstream runtime complexity
- it is easier to design good persistence once the runtime vocabulary is explicit

## Stage 10: Append-Only Rollout File

Build the smallest durable artifact first: one JSONL file.

### Requirements

- each line has a write timestamp
- each line stores one `RolloutItem`
- writes are append-only
- reads reconstruct the exact ordered history

### Interfaces to design

```text
RolloutRecorder::append(items)
RolloutRecorder::flush()
RolloutRecorder::read_history(path) -> Vec<RolloutItem>
```

### Hard questions

- Do you write synchronously on every append, or buffer behind a writer task?
- If append fails once, what happens to later items?
- Do you allow partial progress or all-or-nothing batching?

### Minimum tests

1. append 3 heterogeneous items and read them back in order
2. restart reader on an existing file and reconstruct history
3. ignore or report malformed trailing lines without losing valid prefix lines

### What you should learn

- append-only storage is simple and robust
- replay order is part of the contract
- log format is not the same thing as query format

## Stage 11: Live Event Channel

Now add in-memory fanout.

### Requirements

- `Session::send_event_raw(event)` must:
  1. persist the event to rollout history
  2. then deliver it on an in-process channel
- closed subscribers must not break persistence

### Interfaces to design

```text
Session::send_event_raw(event)
Session::deliver_event_raw(event)
EventBus::subscribe() -> Receiver<Event>
```

### Stop-and-check questions

- Why is `persist then deliver` safer than `deliver then persist`?
- What user-visible behavior changes if subscribers are slow or absent?
- Should replay read from the channel or from the rollout file?

### Minimum tests

1. emitted event appears in rollout history and on the live receiver
2. closed receiver does not break event persistence
3. event order at the receiver matches persisted order

### What you should learn

- live fanout is a projection, not the source of truth
- ordering must be owned at the session/writer boundary

## Stage 12: Thread Store and Persistence Boundary

Before adding more tables, define the persistence abstraction boundary.

### Goal

Separate the session runtime from the concrete storage backend.

### Exercise

Define a small `ThreadStore` or `SessionStore` abstraction that the runtime depends on.

At minimum it should support:

- create or attach to a thread/session
- append rollout items
- flush or persist
- read history
- query metadata

### Interfaces to design

```text
ThreadStore::append(session_id, items)
ThreadStore::flush(session_id)
ThreadStore::read_thread(session_id)
ThreadStore::list_threads(query)
```

### Stop-and-check questions

- why should `Session` not write files and SQLite rows directly everywhere?
- which invariants belong in the runtime versus the store?
- if you later add a remote store, what should not change?

### Minimum tests

1. session code works against an in-memory fake store
2. local store preserves append order and query behavior
3. store failures propagate without corrupting turn state

### What you should learn

- the persistence seam is part of the functional design
- testability improves once the store boundary is explicit

## Stage 13: SQLite Metadata Index

Now separate replay storage from query storage.

### Goal

Do not move transcript history into SQLite. Keep JSONL authoritative for full history. Use SQLite for metadata and lookup.

### Add these tables

Start with only:

- `threads`
- `thread_goals` or `thread_status`

Optional later:

- `thread_dynamic_tools`
- `logs`

### Requirements

- rollout write updates the thread metadata row
- metadata is queryable without opening every JSONL file
- deleting a thread can cascade dependent metadata

### Interfaces to design

```text
StateRuntime::upsert_thread(metadata)
StateRuntime::list_threads(...)
StateRuntime::get_thread(session_id)
```

### Design prompt

Define exactly which fields belong in SQLite:

- session id
- rollout path
- created/updated timestamps
- cwd
- title
- provider/model
- archival flag

Then justify why the full transcript does not belong in the same table.

### Minimum tests

1. writing rollout items updates `threads.updated_at`
2. list API returns metadata without reading JSONL body
3. metadata survives process restart

### What you should learn

- indexed metadata and append-only evidence serve different workloads
- you want one writer aggregate coordinating both so they do not drift arbitrarily

## Stage 14: Resume, Restart, and Recovery

A real agent is not useful if it cannot recover state after process death.

### Goal

Teach the system to reconstruct runnable session state from durable artifacts.

### Exercise

Define what can be resumed from:

- rollout history only
- SQLite metadata only
- both together

### Required decisions

- what is the minimum data required to reopen a session?
- which in-memory states are intentionally not recoverable?
- how do you handle malformed or partial trailing writes?

### Minimum tests

1. restart can list previously persisted sessions
2. restart can read durable history and reconstruct replay state
3. unrecoverable in-memory waits fail explicitly rather than silently resuming

### What you should learn

- recovery semantics reveal which state is truly durable
- many “design” mistakes first appear as restart bugs

## Stage 15: Compaction and Context Trimming

Now handle long-running sessions.

### Goal

Reduce model-visible context size without losing durable evidence.

### Exercise

Add a compaction checkpoint or summarized replacement-history mechanism.

You do not need production-grade summarization. A fake summarizer is enough.

### Required decisions

- what stays in durable rollout history?
- what changes in the next model-visible request?
- how does replay know a compaction happened?

### Minimum tests

1. compaction changes future request construction
2. durable rollout history still explains that compaction occurred
3. replay after compaction remains deterministic

### What you should learn

- transcript history and model-visible context are not identical
- compaction is upstream request-shaping plus downstream evidence recording

## Stage 16: Make the Writer Aggregate Explicit

Up to now you may have smuggled logic everywhere. Fix that.

### Goal

Create one aggregate that owns write ordering across:

- rollout JSONL
- SQLite metadata update
- optional live event fanout trigger

### Exercise

Refactor so only one component is allowed to say:

1. this event is durable
2. metadata is updated
3. subscribers may now observe it

### Deliverable

A short note answering:

- what is the aggregate root?
- what are its invariants?
- what failures can leave the system in a partially updated state?

### Good invariants

- no event is delivered live before it is durably appended
- metadata `updated_at` is never older than the last durable append it indexes
- replay does not depend on SQLite

## Stage 17: Thread Listing, Search, and Operator Queries

Add the downstream query features an operator actually needs.

### Goal

Make stored sessions discoverable without replaying them all.

### Add these queries

- list threads by updated time
- filter archived versus active
- lookup by session/thread id
- optional filter by cwd or title

### Minimum tests

1. listing does not require opening rollout bodies
2. archived filtering works
3. latest-updated ordering is stable

### What you should learn

- operational usability requires indexed metadata, not just durable evidence

## Stage 18: Raw Trace Bundle

Only after normal history works should you add high-fidelity tracing.

### Goal

Capture raw evidence for debugging without polluting the main transcript.

### Bundle layout

Implement:

- `manifest.json`
- `trace.jsonl`
- `payloads/<n>.json`

### Rules

- heavy payloads are written to separate files first
- raw events reference payload files by ID/path
- `trace.jsonl` is append-only and ordered by sequence number

### Interfaces to design

```text
TraceWriter::write_payload(kind, value) -> RawPayloadRef
TraceWriter::append(event)
TraceWriter::append_with_context(context, event)
```

### Design prompt

Why not just put the raw payload inline in every event?

Expected answers:

- duplication
- very large logs
- partial write risk
- replay wants stable references

### Minimum tests

1. payload file exists before event references it
2. raw event sequence numbers are strictly increasing
3. bundle can be reopened and replayed later

### What you should learn

- raw evidence capture is a different concern from user-facing history
- writer ordering matters even more here

## Stage 19: Reduced Trace Model

Now add interpretation.

### Goal

Replay the raw bundle into a semantic graph.

### Important rule

The writer does not build the graph in memory. The reducer does.

### Reduced objects to support

Start with only:

- `SessionTrace`
- `Turn`
- `ConversationItem`
- `RawPayloadRef`

Later add:

- `ToolCall`
- `TerminalOperation`
- `Compaction`

### Interfaces to design

```text
replay_bundle(bundle_dir) -> ReducedTrace
Reducer::apply_event(raw_event)
```

### Stop-and-check questions

- Which facts should stay raw-only?
- Which facts should become first-class reduced objects?
- What happens if a runtime event arrives before the model-visible item it relates to?

### Minimum tests

1. replaying the same bundle twice yields the same reduced graph
2. missing payload file fails replay clearly
3. reducer can tolerate events that require pending state before resolution

### What you should learn

- interpretation belongs offline when possible
- reducer determinism is a design feature, not a convenience

## Stage 20: Add One Advanced Feature Incrementally

Pick only one:

1. tool-call tracing
2. terminal session operations
3. compaction checkpoints
4. child-thread spawn edges

### Rule

Do not add two at once. Add one and force yourself to answer:

- what is the new raw event?
- what payloads does it reference?
- what reduced object does it become?
- what invariant connects it to existing objects?

### Example: terminal operations

Raw events:

- terminal created
- command started
- command completed

Reduced objects:

- `TerminalSession`
- `TerminalOperation`

Invariant:

- every `TerminalOperation` belongs to exactly one `TerminalSession`

## Stage 21: Write the Reader/Writer Aggregate Map

Before declaring the exercise done, document the aggregate boundaries in your own design.

### Template

Use this exact structure:

```text
Aggregate: Session
Writes:
- live event bus projection
- rollout recorder
Reads:
- session state
- active turn handle
Invariants:
- persisted before delivered

Aggregate: ActiveTurn
Writes:
- current turn context
- task membership
Reads:
- turn-local state
Invariants:
- one logical turn owns one turn-local coordination scope

Aggregate: TurnState
Writes:
- pending input
- pending waits
- follow-up flags
Reads:
- current turn-local queues and correlations
Invariants:
- wait resolution and follow-up decisions stay turn-scoped

Aggregate: RequestBuilder
Writes:
- model request payload
Reads:
- durable history
- pending turn input
- session and turn context
Invariants:
- request visibility is derived explicitly, not guessed from storage

Aggregate: ToolRuntime
Writes:
- tool results
- tool lifecycle events
Reads:
- model-visible tool calls
Invariants:
- tool execution outcome is correlated to one call id

Aggregate: RolloutRecorder
Writes:
- rollout JSONL
- state index trigger
Reads:
- buffered append queue
Invariants:
- append order preserved

Aggregate: StateRuntime
Writes:
- SQLite metadata/index rows
Reads:
- listing and lookup queries
Invariants:
- metadata queryable independent of transcript replay

Aggregate: TraceWriter
Writes:
- manifest
- payload files
- raw trace event log
Reads:
- next sequence / payload ordinal state
Invariants:
- payload materialized before reference event

Aggregate: TraceReducer
Writes:
- reduced trace graph
Reads:
- raw trace bundle
Invariants:
- deterministic replay
```

If you cannot fill this in crisply, your design is still blurry.

## Suggested Build Order

Use this exact implementation order:

1. pure domain types
2. session ownership tree
3. turn lifecycle state machine
4. pending interaction state machine
5. model sampling and streaming
6. tool execution lifecycle
7. context assembly and input queueing
8. cancellation and stop conditions
9. one upstream extension
10. rollout JSONL writer/reader
11. session event bus
12. thread store abstraction
13. SQLite metadata index
14. resume and recovery
15. compaction
16. writer aggregate refactor
17. operator query paths
18. raw trace bundle
19. reducer
20. one advanced feature

This order is strict for learning value. If you start with SQLite or the reducer, you will likely build abstractions you have not yet earned. If you start with persistence before the turn machine, you will likely persist the wrong vocabulary.

## Optional Review Rubric

Review your own implementation against these questions:

1. Can I reconstruct truth from disk without any in-memory channel?
2. Can I explain which state is session-lifetime versus turn-lifetime?
3. Can I explain which component is allowed to decide a turn has ended?
4. Can I explain how model-visible requests are assembled from history plus pending input?
5. Can I explain how tool execution feeds back into the same logical turn?
6. Can I query useful metadata without replaying the whole transcript?
7. Is my live channel only a projection?
8. Did I separate raw evidence capture from semantic interpretation?
9. Can one writer aggregate explain ordering guarantees end-to-end?
10. Can one reducer aggregate explain semantic reconstruction end-to-end?

## Non-Functional Requirements

Put these at the end of your own design doc only after the functional design is coherent.

### Correctness and Durability

- durable append order must be preserved
- no live event should be more authoritative than durable history
- restart behavior must be explicit for every state bucket
- malformed tail data should fail safely without corrupting valid prefix history

### Performance

- append path should stay cheap enough for turn-time usage
- query paths should avoid replaying full transcripts for routine listing
- trace capture should not impose large overhead when disabled
- reducers may be slower, but should be deterministic

### Reliability

- writer failures must surface clearly
- partial failures should leave the system diagnosable
- background tasks must not silently drop terminal failures
- cancellation should not leave zombie turn state

### Evolvability

- rollout schema changes should be versionable
- trace bundle schema changes should be versionable
- storage abstraction should allow local and future remote backends
- new runtime states should be addable without rewriting persistence from scratch

### Observability and Debuggability

- every important terminal outcome should have a durable explanation path
- correlation IDs should exist for waits, tool calls, and trace events where needed
- reduced trace should point back to raw evidence
- operator queries should expose enough metadata to find the right session quickly

### Safety and Isolation

- one turn should not mutate another turn's local state
- one tool call should not resolve another call's wait slot
- child-agent or mailbox state should preserve ownership boundaries
- trace and storage should avoid inventing facts the runtime did not actually produce

## Mapping Back to the Real Codex System

After you finish your version, compare your design to these production components:

- upstream state-machine overview: [README.md](/Users/yao/projects/yao-job-search/codex/learning/statemachine/README.md:1)
- turn lifecycle state machine: [02-turn-lifecycle.md](/Users/yao/projects/yao-job-search/codex/learning/statemachine/02-turn-lifecycle.md:1)
- state ownership and mutation: [03-state-ownership-and-mutation.md](/Users/yao/projects/yao-job-search/codex/learning/statemachine/03-state-ownership-and-mutation.md:1)
- normal rollout history: [recorder.rs](/Users/yao/projects/yao-job-search/codex/codex-rs/rollout/src/recorder.rs:78)
- event fanout and ordering: [session/mod.rs](/Users/yao/projects/yao-job-search/codex/codex-rs/core/src/session/mod.rs:1617)
- SQLite runtime/index: [runtime.rs](/Users/yao/projects/yao-job-search/codex/codex-rs/state/src/runtime.rs:87)
- raw trace bundle writer: [writer.rs](/Users/yao/projects/yao-job-search/codex/codex-rs/rollout-trace/src/writer.rs:36)
- offline trace reducer: [reducer/mod.rs](/Users/yao/projects/yao-job-search/codex/codex-rs/rollout-trace/src/reducer/mod.rs:44)

Do that comparison only after your own design exists. Otherwise you will copy names without understanding the pressure behind them.
