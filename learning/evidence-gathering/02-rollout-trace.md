# Evidence Gathering: Rollout Trace

This note describes `codex-rs/rollout-trace`, which is the local forensic evidence system for reconstructing what happened during a Codex session.

Its core design principle is:

- observe first
- interpret later

The runtime writes raw evidence cheaply on the hot path. A reducer later converts that evidence into a semantic graph.

## 1) Big Picture

```mermaid
flowchart TD
  A[Codex runtime]
  A --> B[ThreadTraceContext]
  B --> C[TraceWriter]

  C --> D[manifest.json]
  C --> E[trace.jsonl]
  C --> F[payloads/*.json]

  D --> G[replay_bundle]
  E --> G
  F --> G

  G --> H[state.json]
  H --> H1[threads + turns]
  H --> H2[conversation_items]
  H --> H3[inference_calls / tool_calls / code_cells]
  H --> H4[terminal_operations / compactions]
  H --> H5[interaction_edges]
  H --> H6[raw_payload_refs]
```

## 2) Runtime Sequence

```mermaid
sequenceDiagram
  participant Session as Session Startup
  participant Thread as ThreadTraceContext
  participant Writer as TraceWriter
  participant Runtime as Inference / Tools / Terminal / Agents
  participant Reducer as replay_bundle

  Session->>Thread: start_root_or_disabled(metadata)
  alt Child thread spawn
    Session->>Thread: start_child_thread_trace_or_disabled(metadata)
  end

  Runtime->>Writer: write_json_payload(...)
  Runtime->>Writer: append_with_context(...)
  Runtime->>Writer: append more ordered events

  Note over Writer: payload written before referencing event

  Runtime->>Thread: record_ended(status)
  Writer-->>Reducer: manifest + trace.jsonl + payload files
  Reducer->>Reducer: replay in seq order
  Reducer-->>Session: reduced state graph
```

## 3) Why It Exists

Normal conversation history is not enough to explain many runtime questions. `rollout-trace` preserves enough evidence to reconstruct:

- which model request produced a tool call
- whether output came from model-visible transcript or runtime-only execution
- which code-mode `exec` cell issued nested tools
- how terminal operations relate to tool calls
- how spawned child agents interacted with their parent

## 4) Raw Evidence vs Reduced Graph

```mermaid
flowchart LR
  A[Model-facing requests and responses] --> C[Raw payload files]
  B[Runtime observations] --> C
  C --> D[Reducer]

  D --> E[ConversationItem]
  D --> F[ToolCall]
  D --> G[CodeCell]
  D --> H[TerminalOperation]
  D --> I[InferenceCall]
  D --> J[InteractionEdge]
  D --> K[RawPayloadRef]
```

The important rule is that raw payloads are evidence, not the final semantic model. The reducer decides what became model-visible conversation and what remained runtime-only structure.

## 5) Bundle Structure

- `manifest.json`: trace identity and top-level metadata
- `trace.jsonl`: ordered raw event spine
- `payloads/*.json`: larger raw evidence blobs
- `state.json`: optional reducer output

`TraceWriter` guarantees that payload files are written before the event that refers to them. That keeps replay deterministic even if the process is interrupted.

## 6) Multi-Agent Shape

```mermaid
flowchart LR
  A[Root session] --> B[Root bundle]
  B --> C[Root thread events]
  B --> D[Child thread events]
  B --> E[Interaction edges]

  E --> E1[spawn_agent]
  E --> E2[task delivery]
  E --> E3[child result]
  E --> E4[close_agent]
```

Spawned child threads do not get their own isolated bundles if they belong to the same rollout tree. They share the root trace writer so one reduced `state.json` can explain the whole interaction graph.

## 7) Main Components

- `ThreadTraceContext`
  - no-op capable handle for one thread
  - starts root or child thread tracing
  - records typed protocol, turn, tool, and agent-result events

- `TraceWriter`
  - owns bundle layout
  - writes payload files
  - appends ordered events with sequence numbers

- reducer
  - replays raw evidence in `seq` order
  - builds semantic runtime objects and edges
  - keeps `RawPayloadRef` pointers back to exact evidence

## 8) Why This Is Different from Monitor

`monitor` asks, “Do we have enough evidence to decide whether this risky action should proceed right now?”

`rollout-trace` asks, “Can we later reconstruct what really happened across inference, tools, terminals, code mode, and child threads?”

That is why `monitor` sends a compact approval-oriented request, while `rollout-trace` records raw local evidence and defers interpretation to an offline reducer.

## 9) Key Files

- `codex-rs/rollout-trace/README.md`
- `codex-rs/rollout-trace/src/thread.rs`
- `codex-rs/rollout-trace/src/writer.rs`
- `codex-rs/core/src/session/session.rs`
