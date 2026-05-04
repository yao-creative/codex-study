# Planning Data Structures

This document focuses on fundamental runtime data structures used by planning paths (not only wire schemas).

## 1) Fundamental Runtime Data Structures

### 1.1 Hash Maps (`HashMap<K, V>`)

- Where:
  - `TurnState` pending maps in `core/src/state/turn.rs`
  - Plan-mode parsing state in `core/src/session/turn.rs`
  - Agent registry tree index in `core/src/agent/registry.rs`
- Why:
  - O(1)-ish keyed correlation for async request/response and per-item stream state.
- Typical keys:
  - `String` call/turn ids (`pending_user_input`, `pending_approvals`, `pending_dynamic_tools`)
  - Tuple key `(String, RequestId)` for MCP elicitation correlation
  - `item_id` for parser and agent-message buffering
  - Agent path string for registry ownership
- Access/control pattern:
  - Insert at request start (`insert_pending_*`)
  - Remove on response (`remove_pending_*`)
  - Full cleanup on turn teardown (`clear_pending`)
  - In stream parsing: create-on-first-delta, remove on item completion (`finish_item`)

### 1.2 Hash Sets (`HashSet<T>`)

- Where:
  - `started_agent_message_items` in plan-mode stream state (`core/src/session/turn.rs`)
  - used nicknames in agent registry (`core/src/agent/registry.rs`)
- Why:
  - Fast membership checks to enforce idempotent lifecycle transitions (start once, reserve nickname once).
- Access/control pattern:
  - Insert when lifecycle enters started/reserved
  - Check before emitting duplicate events
  - Remove when completion/release occurs

### 1.3 Vectors (`Vec<T>`)

- Where:
  - Schema payloads (`plan: Vec<PlanItemArg>`, `input: Vec<UserInput>`, etc.)
  - Pending input queue (`pending_input: Vec<ResponseInputItem>`) in turn state
  - Parsed stream segment batches (`plan_segments: Vec<ProposedPlanSegment>`)
- Why:
  - Ordered, contiguous representation for payloads and append-heavy queues.
- Access/control pattern:
  - Append (`push`) for newly arrived items
  - Prepend by rebuilding vector when priority insertion is needed (`prepend_pending_input`)
  - Drain/swap at consumption boundary (`take_pending_input`)

### 1.4 Double-Ended Queue (`VecDeque<T>`)

- Where:
  - Mailbox receiver buffer: `pending_mails: VecDeque<InterAgentCommunication>` (`core/src/agent/mailbox.rs`)
- Why:
  - Queue semantics with efficient front-drain after burst `try_recv`.
- Access/control pattern:
  - `push_back` when syncing from channel
  - `drain(..)` to consume in delivery order
  - `has_pending*` checks without forcing full materialization elsewhere

### 1.5 Channels (`oneshot`, `mpsc`, `watch`)

- `oneshot::Sender<T>` in `TurnState` pending maps:
  - Why: exact one-response rendezvous for approvals, request-user-input, elicitation, dynamic tools.
  - Control: sender stored in map; removed and resolved exactly once by correlation key.

- `mpsc::UnboundedSender<T>` in mailbox:
  - Why: fire-and-forget inter-agent message transport.
  - Control: receiver side batches into `VecDeque` via `try_recv`.

- `watch::Sender<u64>` + `AtomicU64` sequence in mailbox:
  - Why: cheap change notification for waiters without polling full queues.
  - Control: monotonic sequence increment on every send; subscribers wake on sequence change.

### 1.6 Logical Tree (Implemented as Hash Map)

- Structure:
  - Agent hierarchy is logically a tree, but stored as `HashMap<String, AgentMetadata>` keyed by path (`/root/...`) in registry.
- Why:
  - Fast lookup/mutation by path while preserving parent/child semantics in the path string.
- Access/control pattern:
  - Insert/update by full path
  - Remove by path
  - Parent-child reasoning derived from normalized path prefixes rather than pointer-linked tree nodes

## 2) Planning State-Control Lifecycle (Quick Reference)

- Start turn:
  - `TurnStartParams` enters app-server handler and becomes core op.
  - `collaboration_mode.mode` gates plan-mode parser behavior.
- During streaming:
  - Parser state keyed by `item_id` in `HashMap`.
  - Plan deltas emitted incrementally (`PlanDeltaEvent` -> `PlanDeltaNotification`).
- During checklist updates:
  - `update_plan` payload (`UpdatePlanArgs`) mapped to `turn/plan/updated`.
- During interactive questions:
  - pending map inserts `oneshot::Sender`; response removes and resolves it.
- End/cleanup:
  - per-item parser state removed on completion
  - pending maps cleared on cancellation/turn teardown

## 3) Cross-References

- Overview: [01-planning-overview.md](/Users/yao/projects/codex/learning/planning/01-planning-overview.md)
- Algorithms: [03-planning-algorithm-patterns.md](/Users/yao/projects/codex/learning/planning/03-planning-algorithm-patterns.md)
- Protocol transport/contracts view: [planning-protocol-structures.md](/Users/yao/projects/codex/learning/protocol/planning-protocol-structures.md)
