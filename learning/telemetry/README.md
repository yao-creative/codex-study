# Telemetry and Evaluable Data

This note answers two questions for `codex-rs`:

1. Where does telemetry drain to?
2. Where does replayable or eval-usable session data drain to?

The short answer is that there are three different pipelines:

- OTEL/runtime metrics: aggregate counters, histograms, durations
- analytics events: reduced product telemetry shipped to the backend
- rollout evidence: local or storage-backed session history and optional raw trace bundles

They are related, but they are not the same sink.

## 1) The Three Pipelines

| Pipeline | Primary purpose | Main entrypoints | Drain destination |
| --- | --- | --- | --- |
| OTEL metrics | runtime counters/histograms/timers | `SessionTelemetry::counter`, `histogram`, `record_duration` | configured metrics exporter via `MetricsClient` |
| Analytics events | product/event telemetry | `AnalyticsEventsClient::track_*` | background reducer, then HTTP POST to `/codex/analytics-events/events` |
| Rollout persistence | replay/debug/eval evidence | `Session::record_conversation_items`, `send_event_raw` | `thread-store`, usually local rollout JSONL plus `state_db` metadata |
| Rollout trace | exact raw diagnostic evidence | `ThreadTraceContext::*` | local trace bundle under `CODEX_ROLLOUT_TRACE_ROOT` |

## 2) OTEL Metrics

The lowest-friction telemetry path is `SessionTelemetry`.

- It is attached to the session in `core/src/session/session.rs`.
- Callers record counters, histograms, and durations throughout turn execution, tool calls, websocket/SSE handling, and related runtime paths.
- `SessionTelemetry` does not keep a local event log. It forwards measurements into `MetricsClient`.
- If no metrics exporter is configured, calls become a no-op.

Primary code:

- `codex-rs/otel/src/events/session_telemetry.rs`
- `codex-rs/core/src/tasks/mod.rs`
- `codex-rs/core/src/mcp_tool_call.rs`

Design shape:

- low-overhead, aggregate-only
- not replayable
- not a turn transcript
- meant for operational measurement rather than forensic reconstruction

## 3) Analytics Events

`AnalyticsEventsClient` is the explicit product-analytics drain.

### 3.1 Queue and Reduction Design

- `AnalyticsEventsClient::record_fact()` pushes `AnalyticsFact` into a bounded Tokio `mpsc` queue.
- Queue size is `256`.
- If the queue is full, analytics events are dropped with a warning.
- A background task owns an `AnalyticsReducer`.
- The reducer joins together request, response, notification, thread, and turn facts until it can emit a higher-level `TrackEventRequest`.

This is important: analytics is not sent one-for-one from each callsite.

Instead, the reducer waits until enough context exists. For example, a full turn event is only emitted once it has:

- thread id
- connection id
- resolved config
- completion status
- token usage if available

Primary code:

- `codex-rs/analytics/src/client.rs`
- `codex-rs/analytics/src/reducer.rs`

### 3.2 Final Drain Destination

When the reducer emits events:

- `send_track_events()` posts them to:
  - `{base_url}/codex/analytics-events/events`
- auth is attached through the normal auth provider
- non-success responses are logged and not retried by a durable local spool

This means analytics is:

- asynchronous
- reduced
- lossy under queue pressure
- not the source of truth for session replay

## 4) Rollout Persistence

The closest thing to "evaluable data" in the codebase is rollout persistence.

This is the durable session history used for:

- replay
- resume
- inspection
- thread listing and metadata lookup
- tests that assert actual persisted interaction history

### 4.1 What Gets Written

Session code persists both:

- model-visible `ResponseItem`s
- selected `EventMsg`s wrapped as `RolloutItem::EventMsg`

Main session callsites:

- `Session::record_conversation_items()`
- `Session::send_event_raw()`
- `Session::persist_rollout_items()`

Primary code:

- `codex-rs/core/src/session/mod.rs`
- `codex-rs/rollout/src/recorder.rs`
- `codex-rs/thread-store/src/live_thread.rs`

### 4.2 Storage Abstraction

The session does not write directly to files.

It writes through `LiveThread`, which delegates to `ThreadStore`.

That means the sink can be:

- local rollout storage via `LocalThreadStore`
- remote persistence via `RemoteThreadStore`

Primary code:

- `codex-rs/thread-store/src/live_thread.rs`
- `codex-rs/thread-store/src/local/live_writer.rs`
- `codex-rs/thread-store/src/remote/mod.rs`

### 4.3 Local Writer Design

In the local case, `LocalThreadStore` uses `RolloutRecorder`.

`RolloutRecorder`:

- owns a bounded `mpsc` command channel
- spawns one background writer task
- buffers `RolloutItem`s in memory
- materializes the JSONL file lazily on `persist()`
- flushes eagerly after `AddItems` if the file is already materialized
- updates `state_db` after writes so listing and metadata queries stay in sync

Important commands:

- `AddItems`
- `Persist`
- `Flush`
- `Shutdown`

Important properties:

- async write boundary is explicit
- persistence is append-only JSONL
- write failures stay buffered for retry paths like later `persist()` or `flush()`
- `state_db` is an index/metadata companion, not the full transcript

Primary code:

- `codex-rs/rollout/src/recorder.rs`

### 4.4 What This Produces

For local threads, the durable artifact is typically a rollout file under the Codex home sessions tree, for example:

- `sessions/YYYY/MM/DD/rollout-<timestamp>-<thread_id>.jsonl`

The exact path is computed by the rollout recorder and exposed through `LiveThread::local_rollout_path()`.

## 5) Rollout Trace Bundles

Rollout traces are separate from normal rollout JSONL.

They are opt-in and diagnostic.

If `CODEX_ROLLOUT_TRACE_ROOT` is set:

- a root session creates a trace bundle directory
- child spawned threads join the same rollout tree rather than creating independent bundles

Bundle layout:

- `manifest.json`
- `trace.jsonl`
- `payloads/*.json`

Primary code:

- `codex-rs/rollout-trace/src/thread.rs`
- `codex-rs/rollout-trace/src/writer.rs`
- `codex-rs/rollout-trace/src/bundle.rs`

Design intent:

- preserve exact raw payloads for inference/tool/compaction/runtime debugging
- keep heavyweight payloads out of normal reduced state
- allow later replay/reduction into a structured `RolloutTrace`

This is the highest-fidelity local evidence sink in the codebase.

## 6) What Is Actually "Evaluable"?

The code does not define a single first-class concept named "evaluable data".

In practice, the most evaluation-friendly artifacts are:

- rollout JSONL:
  - user input
  - assistant messages
  - persisted tool/runtime events
  - enough history to replay or inspect a session
- `state_db`:
  - indexed metadata for discovery, repair, filtering, and resume support
- rollout trace bundles:
  - exact raw request/response payload evidence for deep debugging

By contrast:

- OTEL metrics are aggregate measurements
- analytics events are reduced product telemetry

Those two are useful for monitoring and product analysis, but they are not the full replay/eval substrate.

## 7) Sequence Diagram

```mermaid
sequenceDiagram
    participant Runtime as Core Runtime
    participant Telemetry as SessionTelemetry
    participant Metrics as MetricsClient / OTEL Exporter
    participant Analytics as AnalyticsEventsClient
    participant Reducer as AnalyticsReducer Worker
    participant Store as LiveThread / ThreadStore
    participant Recorder as RolloutRecorder Worker
    participant DB as state_db
    participant Trace as ThreadTraceContext / TraceWriter
    participant Backend as Analytics Backend

    Runtime->>Telemetry: counter / histogram / duration
    Telemetry->>Metrics: emit aggregate metric

    Runtime->>Analytics: track_* fact
    Analytics->>Analytics: enqueue AnalyticsFact
    Analytics->>Reducer: background recv()
    Reducer->>Reducer: join request/response/notification/turn context
    Reducer->>Backend: POST /codex/analytics-events/events

    Runtime->>Store: append_items(RolloutItem...)
    Store->>Recorder: queue AddItems / Persist / Flush
    Recorder->>Recorder: append JSONL when materialized
    Recorder->>DB: sync metadata after write

    Runtime->>Trace: record_protocol_event / inference/tool trace
    Trace->>Trace: append trace.jsonl event
    Trace->>Trace: write payloads/*.json as needed
```

## 8) Analytics Drain Schema

The analytics drain payload is rooted at `TrackEventsRequest`:

- top-level request: `events: Vec<TrackEventRequest>`
- each item is an untagged event object with:
  - `event_type`
  - event-specific `event_params`

Primary schema source:

- `codex-rs/analytics/src/events.rs`

### 8.1 Hierarchical Tree

```mermaid
flowchart TD
    A["TrackEventsRequest"] --> B["events[]: TrackEventRequest"]

    B --> C["skill_invocation"]
    B --> D["codex_thread_initialized"]
    B --> E["codex_guardian_review"]
    B --> F["codex_app_mentioned"]
    B --> G["codex_app_used"]
    B --> H["codex_hook_run"]
    B --> I["codex_compaction_event"]
    B --> J["codex_turn_event"]
    B --> K["codex_turn_steer_event"]
    B --> L["codex_plugin_used"]
    B --> M["codex_plugin_installed / uninstalled / enabled / disabled"]

    C --> C1["skill_id"]
    C --> C2["skill_name"]
    C --> C3["event_params"]
    C3 --> C31["product_client_id?"]
    C3 --> C32["skill_scope?"]
    C3 --> C33["repo_url?"]
    C3 --> C34["thread_id?"]
    C3 --> C35["invoke_type?"]
    C3 --> C36["model_slug?"]

    D --> D1["event_params"]
    D1 --> D11["thread_id"]
    D1 --> D12["app_server_client"]
    D1 --> D13["runtime"]
    D1 --> D14["model"]
    D1 --> D15["ephemeral"]
    D1 --> D16["thread_source?"]
    D1 --> D17["initialization_mode"]
    D1 --> D18["subagent_source?"]
    D1 --> D19["parent_thread_id?"]
    D1 --> D110["created_at"]

    E --> E1["event_params"]
    E1 --> E11["app_server_client"]
    E1 --> E12["runtime"]
    E1 --> E13["guardian_review fields"]
    E13 --> E131["thread_id / turn_id / review_id"]
    E13 --> E132["target_item_id?"]
    E13 --> E133["approval_request_source"]
    E13 --> E134["reviewed_action"]
    E13 --> E135["decision / terminal_status / failure_reason?"]
    E13 --> E136["risk_level? / user_authorization? / outcome?"]
    E13 --> E137["guardian_thread_id? / guardian_session_kind?"]
    E13 --> E138["guardian_model? / guardian_reasoning_effort?"]
    E13 --> E139["tool_call_count? / time_to_first_token_ms? / completion_latency_ms?"]
    E13 --> E1310["started_at / completed_at?"]
    E13 --> E1311["input_tokens? / cached_input_tokens? / output_tokens? / reasoning_output_tokens? / total_tokens?"]

    F --> F1["event_params: CodexAppMetadata"]
    G --> F1
    F1 --> F11["connector_id?"]
    F1 --> F12["thread_id?"]
    F1 --> F13["turn_id?"]
    F1 --> F14["app_name?"]
    F1 --> F15["product_client_id?"]
    F1 --> F16["invoke_type?"]
    F1 --> F17["model_slug?"]

    H --> H1["event_params"]
    H1 --> H11["thread_id?"]
    H1 --> H12["turn_id?"]
    H1 --> H13["model_slug?"]
    H1 --> H14["hook_name?"]
    H1 --> H15["hook_source?"]
    H1 --> H16["status?"]

    I --> I1["event_params"]
    I1 --> I11["thread_id / turn_id"]
    I1 --> I12["app_server_client"]
    I1 --> I13["runtime"]
    I1 --> I14["thread_source? / subagent_source? / parent_thread_id?"]
    I1 --> I15["trigger / reason / implementation / phase / strategy / status"]
    I1 --> I16["error?"]
    I1 --> I17["active_context_tokens_before / active_context_tokens_after"]
    I1 --> I18["started_at / completed_at / duration_ms?"]

    J --> J1["event_params"]
    J1 --> J11["thread_id / turn_id"]
    J1 --> J12["submission_type?"]
    J1 --> J13["app_server_client"]
    J1 --> J14["runtime"]
    J1 --> J15["ephemeral / thread_source? / initialization_mode"]
    J1 --> J16["subagent_source? / parent_thread_id?"]
    J1 --> J17["model? / model_provider"]
    J1 --> J18["sandbox_policy? / reasoning_effort? / reasoning_summary?"]
    J1 --> J19["service_tier / approval_policy / approvals_reviewer"]
    J1 --> J110["sandbox_network_access / collaboration_mode? / personality?"]
    J1 --> J111["num_input_images / is_first_turn"]
    J1 --> J112["status / turn_error? / steer_count?"]
    J1 --> J113["tool-count fields?"]
    J1 --> J114["token usage fields?"]
    J1 --> J115["duration_ms? / started_at? / completed_at?"]

    K --> K1["event_params"]
    K1 --> K11["thread_id"]
    K1 --> K12["expected_turn_id? / accepted_turn_id?"]
    K1 --> K13["app_server_client"]
    K1 --> K14["runtime"]
    K1 --> K15["thread_source? / subagent_source? / parent_thread_id?"]
    K1 --> K16["num_input_images"]
    K1 --> K17["result / rejection_reason?"]
    K1 --> K18["created_at"]

    L --> L1["event_params"]
    L1 --> L11["plugin fields"]
    L1 --> L12["thread_id? / turn_id? / model_slug?"]
    M --> M1["event_params: plugin fields"]
    L11 --> P1["plugin_id? / plugin_name? / marketplace_name?"]
    L11 --> P2["has_skills? / mcp_server_count?"]
    L11 --> P3["connector_ids?"]
    L11 --> P4["product_client_id?"]

    D12 --> S1["CodexAppServerClientMetadata"]
    E11 --> S1
    I12 --> S1
    J13 --> S1
    K13 --> S1
    S1 --> S11["product_client_id"]
    S1 --> S12["client_name? / client_version?"]
    S1 --> S13["rpc_transport"]
    S1 --> S14["experimental_api_enabled?"]

    D13 --> R1["CodexRuntimeMetadata"]
    E12 --> R1
    I13 --> R1
    J14 --> R1
    K14 --> R1
    R1 --> R11["codex_rs_version"]
    R1 --> R12["runtime_os / runtime_os_version / runtime_arch"]
```

### 8.2 Shared Object Shapes

The reducer reuses a few important sub-objects across many event kinds:

- `CodexAppServerClientMetadata`
  - `product_client_id`
  - `client_name?`
  - `client_version?`
  - `rpc_transport`
  - `experimental_api_enabled?`
- `CodexRuntimeMetadata`
  - `codex_rs_version`
  - `runtime_os`
  - `runtime_os_version`
  - `runtime_arch`
- `TokenUsage`-style fields
  - `input_tokens?`
  - `cached_input_tokens?`
  - `output_tokens?`
  - `reasoning_output_tokens?`
  - `total_tokens?`

### 8.3 Practical Reading Guide

If you want the most important analytics event families first, start here:

- `codex_turn_event`
  - the main joined turn-level record
- `codex_turn_steer_event`
  - steer acceptance/rejection outcomes
- `codex_thread_initialized`
  - thread/session lifecycle start
- `codex_guardian_review`
  - approval-review outcomes
- `codex_compaction_event`
  - compaction decisions and timings

Everything else is more specialized:

- app mention/use
- hook runs
- skill invocation
- plugin state/use

## 9) Flow Diagram

```mermaid
flowchart TD
    A["Runtime emits data"] --> B{"Data kind"}

    B -->|metric| C["SessionTelemetry"]
    C --> D["MetricsClient"]
    D --> E["Configured OTEL exporter"]

    B -->|analytics fact| F["AnalyticsEventsClient"]
    F --> G["bounded mpsc queue (256)"]
    G --> H["AnalyticsReducer background task"]
    H --> I["TrackEventRequest"]
    I --> J["HTTP POST /codex/analytics-events/events"]

    B -->|rollout item| K["Session::persist_rollout_items"]
    K --> L["LiveThread / ThreadStore"]
    L --> M{"Store kind"}
    M -->|local| N["RolloutRecorder"]
    N --> O["rollout JSONL"]
    N --> P["state_db metadata sync"]
    M -->|remote| Q["RemoteThreadStore service"]

    B -->|trace evidence| R["ThreadTraceContext"]
    R --> S["trace.jsonl"]
    R --> T["payloads/*.json"]
    R --> U["manifest.json"]
```

## 9) Practical Mental Model

If you are asking "where did this runtime fact go?", use this map:

- "I need counters, timings, rates"
  - OTEL metrics
- "I need product telemetry for backend analysis"
  - analytics reducer and HTTP events
- "I need to replay or inspect what actually happened in a session"
  - rollout JSONL and `state_db`
- "I need exact raw request/response/tool payload evidence"
  - rollout trace bundle

## 10) Most Important Files

- `codex-rs/analytics/src/client.rs`
- `codex-rs/analytics/src/reducer.rs`
- `codex-rs/otel/src/events/session_telemetry.rs`
- `codex-rs/core/src/session/mod.rs`
- `codex-rs/core/src/session/session.rs`
- `codex-rs/core/src/session/turn.rs`
- `codex-rs/thread-store/src/live_thread.rs`
- `codex-rs/thread-store/src/local/live_writer.rs`
- `codex-rs/rollout/src/recorder.rs`
- `codex-rs/rollout-trace/src/thread.rs`
- `codex-rs/rollout-trace/src/writer.rs`
