# Planning Algorithm Patterns

This maps planning/protocol runtime behavior to familiar LeetCode-style patterns.

## 1) Algorithm Patterns (LeetCode-Style Mapping)

### 1.1 Hash Map for Correlation IDs

- Pattern: `Two Sum`-style index map (`id -> state`).
- In code:
  - `TurnState.pending_*` maps store `id -> oneshot sender`.
- Why it fits:
  - Insert on request, O(1)-ish lookup/remove on response.
  - Converts async matching from scan-based to direct-addressed.

### 1.2 Queue / FIFO Drain

- Pattern: `Queue using deque` + batched drain.
- In code:
  - `MailboxReceiver.pending_mails: VecDeque<_>` with `push_back` and `drain(..)`.
- Why it fits:
  - Preserves delivery order while allowing burst ingestion from channel.

### 1.3 Streaming Parser State Machine

- Pattern: finite-state machine over character stream.
- In code:
  - assistant text parsing that separates normal text vs `<proposed_plan>` segments.
- Why it fits:
  - Handles split tags across chunk boundaries using persistent parser state.
  - Emits transitions: normal text, plan start, plan delta, plan end.

### 1.4 Incremental + Final Reconciliation

- Pattern: online processing + final authoritative pass.
- In code:
  - stream `PlanDelta` during chunks, then re-extract finalized plan text from completed message.
- Why it fits:
  - Real-time UX with eventual canonical output.
  - Similar to "first pass approximate, second pass exact".

### 1.5 Set-Based Idempotency Guards

- Pattern: visited set / duplicate suppression.
- In code:
  - `started_agent_message_items: HashSet<String>`.
- Why it fits:
  - Prevents duplicate start events in interleaved stream conditions.

### 1.6 Monotonic Sequence + Observer

- Pattern: producer sequence counter + subscriber notification.
- In code:
  - mailbox `AtomicU64 next_seq` + `watch::Sender<u64>`.
- Why it fits:
  - Waiters block on sequence changes instead of polling the whole queue.

### 1.7 Tree-as-Path-Keyed Map

- Pattern: trie/tree semantics represented as hashed path keys.
- In code:
  - agent hierarchy in `HashMap<String, AgentMetadata>` keyed by agent path.
- Why it fits:
  - Faster random lookup/update than pointer-walk trees for this workload.
  - Parent/child relationships derived from path conventions.

### 1.8 Phase Enum as State Machine

- Pattern: explicit state machine (`enum`) controlling transitions.
- In code:
  - `MailboxDeliveryPhase::{CurrentTurn, NextTurn}`.
- Why it fits:
  - Encodes legal behavior transitions and avoids implicit boolean flags.

## 2) Pattern-to-Problem Cheat Sheet

- Correlate async responses by id:
  - Use `HashMap<Id, PendingState>` (insert/remove once).
- Preserve message order under bursts:
  - Use `VecDeque`, push on receive, drain in order.
- Parse structured tags in streaming text:
  - Use incremental FSM parser with retained buffer/state.
- Avoid duplicate lifecycle events:
  - Use `HashSet` guard before emitting.
- Wake waiters efficiently on new work:
  - Use monotonic sequence + watch/observer channel.

## 3) Cross-References

- Overview: [01-planning-overview.md](/Users/yao/projects/codex/learning/planning/01-planning-overview.md)
- Data structures: [02-planning-data-structures.md](/Users/yao/projects/codex/learning/planning/02-planning-data-structures.md)
- Protocol transport/contracts view: [planning-protocol-structures.md](/Users/yao/projects/codex/learning/protocol/planning-protocol-structures.md)
