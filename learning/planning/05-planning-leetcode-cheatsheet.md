# Planning LeetCode Cheatsheet

This file highlights planning-loop and adjacent tool-control patterns in `codex-rs` that map cleanly to common LeetCode data structures/algorithms.

## 1) Streaming FSM Parsing

- Pattern: finite-state parser over chunked text input.
- Where:
  - [proposed_plan.rs](/Users/yao/projects/codex/codex-rs/utils/stream-parser/src/proposed_plan.rs)
- Why interesting:
  - Handles partial tag boundaries across chunks (`<proposed_plan>` / `</proposed_plan>`).
  - Emits typed segments (`Normal`, `ProposedPlanStart`, `ProposedPlanDelta`, `ProposedPlanEnd`).

## 2) Incremental + Final Reconciliation

- Pattern: online pass + authoritative final pass.
- Where:
  - Streaming state and lifecycle: [turn.rs](/Users/yao/projects/codex/codex-rs/core/src/session/turn.rs:1261)
  - Final extraction: [proposed_plan.rs](/Users/yao/projects/codex/codex-rs/utils/stream-parser/src/proposed_plan.rs:93)
- Why interesting:
  - UI can stream `PlanDelta` immediately, then finalize with canonical extracted plan text.

## 3) HashMap/HashSet Correlation and Dedupe

- Pattern: O(1) average lookup for “seen/pending/state by id”.
- Where:
  - `pending_agent_message_items: HashMap<...>`
  - `started_agent_message_items: HashSet<...>`
  - `parsers_by_item: HashMap<...>`
  - [turn.rs](/Users/yao/projects/codex/codex-rs/core/src/session/turn.rs:1274)
- Why interesting:
  - Prevents duplicate lifecycle events in interleaved streaming paths.

## 4) Queue / BFS in Sub-agent Control

- Pattern: FIFO queue and breadth-first traversal.
- Where:
  - Mailbox pending queue: [mailbox.rs](/Users/yao/projects/codex/codex-rs/core/src/agent/mailbox.rs:17)
  - BFS-style resume queue for descendants: [control.rs](/Users/yao/projects/codex/codex-rs/core/src/agent/control.rs:459)
- Why interesting:
  - Maintains deterministic ordering for cross-agent communications and recovery.

## 5) Set-based Orphan Detection (Call-ID Matching)

- Pattern: precompute key sets, then filter/retain by membership.
- Where:
  - [normalize.rs](/Users/yao/projects/codex/codex-rs/core/src/context_manager/normalize.rs:123)
- Why interesting:
  - Efficiently validates tool outputs against existing call IDs.

## 6) Read/Write Lock Parallel Scheduling

- Pattern: concurrency control via shared/exclusive lock selection.
- Where:
  - [parallel.rs](/Users/yao/projects/codex/codex-rs/core/src/tools/parallel.rs:89)
- Why interesting:
  - Parallel-safe tools run under shared read lock, serialized tools under write lock.

## 7) Checklist Tool as Structured Event Emission

- Pattern: parse structured args and emit typed event.
- Where:
  - `update_plan` handler: [plan.rs](/Users/yao/projects/codex/codex-rs/core/src/tools/handlers/plan.rs)
- Why interesting:
  - Useful example of strict mode gating (`Plan` mode rejects checklist tool) plus typed event output.

## Practice Ideas

1. Re-implement `extract_proposed_plan_text` from a stream of chunks and compare behavior on split tags.
2. Build a mini “started/completed” lifecycle tracker with `HashSet` dedupe and event ordering checks.
3. Model mailbox draining with `VecDeque` and benchmark against naive `Vec` front-removal.
