# Storage + Trace Schemas (SQLite, Rollout JSONL, Event Channel, Rollout Trace)

This note captures the on-disk schemas and runtime plumbing for:

- the SQLite state/log DBs
- rollout JSONL files
- the `send_event_raw` event channel
- rollout trace bundles (`CODEX_ROLLOUT_TRACE_ROOT`)

It is intentionally source-driven: each section cites the primary reader/writer aggregates and the struct/table shapes they own.

## SQLite ERD (state DB + logs DB)

### State DB (tables + relationships)

```mermaid
erDiagram
    threads {
        TEXT id PK
        TEXT rollout_path
        INTEGER created_at
        INTEGER updated_at
        TEXT source
        TEXT model_provider
        TEXT cwd
        TEXT title
        TEXT sandbox_policy
        TEXT approval_mode
        INTEGER tokens_used
        INTEGER has_user_event
        INTEGER archived
        INTEGER archived_at
        TEXT git_sha
        TEXT git_branch
        TEXT git_origin_url
    }

    thread_dynamic_tools {
        TEXT thread_id PK
        INTEGER position PK
        TEXT name
        TEXT description
        TEXT input_schema
    }

    stage1_outputs {
        TEXT thread_id PK
        INTEGER source_updated_at
        TEXT raw_memory
        TEXT rollout_summary
        INTEGER generated_at
    }

    thread_goals {
        TEXT thread_id PK
        TEXT goal_id
        TEXT objective
        TEXT status
        INTEGER token_budget
        INTEGER tokens_used
        INTEGER time_used_seconds
        INTEGER created_at_ms
        INTEGER updated_at_ms
    }

    jobs {
        TEXT kind PK
        TEXT job_key PK
        TEXT status
        TEXT worker_id
        TEXT ownership_token
        INTEGER started_at
        INTEGER finished_at
        INTEGER lease_until
        INTEGER retry_at
        INTEGER retry_remaining
        TEXT last_error
        INTEGER input_watermark
        INTEGER last_success_watermark
    }

    backfill_state {
        INTEGER id PK
        TEXT status
        TEXT last_watermark
        INTEGER last_success_at
        INTEGER updated_at
    }

    thread_spawn_edges {
        TEXT parent_thread_id
        TEXT child_thread_id PK
        TEXT status
    }

    agent_jobs {
        TEXT id PK
        TEXT name
        TEXT status
        TEXT instruction
        TEXT output_schema_json
        TEXT input_headers_json
        TEXT input_csv_path
        TEXT output_csv_path
        INTEGER auto_export
        INTEGER created_at
        INTEGER updated_at
        INTEGER started_at
        INTEGER completed_at
        TEXT last_error
    }

    agent_job_items {
        TEXT job_id PK
        TEXT item_id PK
        INTEGER row_index
        TEXT source_id
        TEXT row_json
        TEXT status
        TEXT assigned_thread_id
        INTEGER attempt_count
        TEXT result_json
        TEXT last_error
        INTEGER created_at
        INTEGER updated_at
        INTEGER completed_at
        INTEGER reported_at
    }

    remote_control_enrollments {
        TEXT websocket_url PK
        TEXT account_id PK
        TEXT app_server_client_name PK
        TEXT server_id
        TEXT environment_id
        TEXT server_name
        INTEGER updated_at
    }

    device_key_bindings {
        TEXT key_id PK
        TEXT account_user_id
        TEXT client_id
        INTEGER created_at
        INTEGER updated_at
    }

    threads ||--o{ thread_dynamic_tools : "thread_id"
    threads ||--|| stage1_outputs : "thread_id"
    threads ||--|| thread_goals : "thread_id"
    agent_jobs ||--o{ agent_job_items : "job_id"
```

**Source of truth**

- Writer/reader aggregate: `codex_state::StateRuntime` (`codex-rs/state/src/runtime.rs`) owns opening + migrating both SQLite files and exposes the storage APIs through its submodules (`threads`, `goals`, `memories`, `remote_control`, `device_key`, `agent_jobs`, `logs`). See `StateRuntime::init` and the `pool` / `logs_pool` split. (`codex-rs/state/src/runtime.rs:87`)
- Table schemas: `codex-rs/state/migrations/*.sql` (state DB) and `codex-rs/state/logs_migrations/*.sql` (logs DB), including FK constraints in `thread_dynamic_tools`, `stage1_outputs`, `agent_job_items`, and `thread_goals`.

### Logs DB (tables)

```mermaid
erDiagram
    logs {
        INTEGER id PK
        INTEGER ts
        INTEGER ts_nanos
        TEXT level
        TEXT target
        TEXT feedback_log_body
        TEXT module_path
        TEXT file
        INTEGER line
        TEXT thread_id
        TEXT process_uuid
        INTEGER estimated_bytes
    }
```

**Source of truth**

- Writer/reader aggregate: `codex_state::StateRuntime` uses a dedicated `logs_pool` and runs logs migrations via `runtime_logs_migrator`. (`codex-rs/state/src/runtime.rs:87`)
- Logs schema: `codex-rs/state/logs_migrations/0001_logs.sql` and `codex-rs/state/logs_migrations/0002_logs_feedback_log_body.sql`.

## Rollout JSONL schema (rollout files)

### File shape

Each line is a JSON object with:

- `timestamp`: RFC3339-ish UTC timestamp formatted by the rollout writer
- the rollout item payload, flattened at the top level (`#[serde(flatten)]`)

```mermaid
flowchart TB
    Line["RolloutLineRef\n{ timestamp, ...RolloutItem }"]
    Item["RolloutItem\n(tagged union)"]
    Line --> Item
```

**Source of truth**

- Writer aggregate: `codex_rollout::RolloutRecorder` + `JsonlWriter` (`codex-rs/rollout/src/recorder.rs:78`) writes `RolloutLineRef { timestamp, item: &RolloutItem }` via `JsonlWriter::write_rollout_item`.
- Reader aggregate: `RolloutRecorder::get_rollout_history` (`codex-rs/rollout/src/recorder.rs:923`) reads the JSONL, parses into `RolloutItem`s, and returns `InitialHistory::Resumed`.
- Rollout item union: `codex_protocol::protocol::RolloutItem` (`codex-rs/protocol/src/protocol.rs:2775`).

### `RolloutItem` variants (tagged union)

```mermaid
classDiagram
    class RolloutItem {
      <<enum>>
      SessionMeta(SessionMetaLine)
      ResponseItem(ResponseItem)
      Compacted(CompactedItem)
      TurnContext(TurnContextItem)
      EventMsg(EventMsg)
    }
```

**Source of truth**

- `codex-rs/protocol/src/protocol.rs:2775` defines `#[serde(tag = "type", content = "payload", rename_all = "snake_case")] pub enum RolloutItem`.

## Event channel: `send_event_raw` and `tx_event`

### Runtime flow

```mermaid
flowchart TD
    Caller["Session code\n(e.g. handlers/turn/goals)"]
    Send["Session::send_event_raw(Event)"]
    Persist["persist_rollout_items([EventMsg])\nrollout JSONL"]
    Trace["rollout_thread_trace.record_protocol_event(EventMsg)\nrollout trace bundle (optional)"]
    Deliver["deliver_event_raw(Event)"]
    Chan["async_channel::Sender<Event>\n(tx_event)"]
    Rx["Codex::rx_event\n(Receiver<Event>)"]

    Caller --> Send
    Send --> Persist
    Send --> Trace
    Send --> Deliver
    Deliver --> Chan
    Chan --> Rx
```

**Source of truth**

- Writer aggregate (event emission): `Session::send_event_raw` and `Session::deliver_event_raw` (`codex-rs/core/src/session/mod.rs:1617`) persist the event into rollout storage, record it into rollout trace, then push it onto the in-process event channel.
- Channel construction: `Codex::spawn_internal` creates `(tx_event, rx_event) = async_channel::unbounded()` (`codex-rs/core/src/session/mod.rs:473`) and the returned `Codex` surfaces `rx_event` to clients (`codex-rs/core/src/session/mod.rs:367`).

## Rollout trace bundle schema (rollout trace)

### Bundle layout

```mermaid
flowchart TB
    BundleDir["<bundle_dir>/"]
    Manifest["manifest.json\nTraceBundleManifest"]
    Events["trace.jsonl\nRawTraceEvent (seq-ordered)"]
    Payloads["payloads/*.json\nraw evidence (by ordinal)"]
    Reduced["state.json\nRolloutTrace (reduced graph)"]

    BundleDir --> Manifest
    BundleDir --> Events
    BundleDir --> Payloads
    BundleDir --> Reduced
```

**Source of truth**

- Writer aggregate: `codex_rollout_trace::TraceWriter` (`codex-rs/rollout-trace/src/writer.rs:36`) writes `manifest.json`, appends raw events to `trace.jsonl`, and writes payloads into `payloads/`.
- Bundle constants + manifest shape: `codex-rs/rollout-trace/src/bundle.rs:1`.
- Reader/reducer aggregate: `codex_rollout_trace::replay_bundle` (`codex-rs/rollout-trace/src/reducer/mod.rs:44`) reads `manifest.json`, streams/parses `trace.jsonl`, and reduces into `RolloutTrace` (`codex-rs/rollout-trace/src/model/mod.rs:54`).

## Dataflow graph (end-to-end)

This is the “what goes where” view that ties together the four systems above.

```mermaid
flowchart LR
    subgraph Core["codex-core session runtime"]
        Handlers["turn/handlers\nemit EventMsg + ResponseItems"]
        SendRaw["Session::send_event_raw"]
        PersistRollout["Session::persist_rollout_items"]
        EventChan["async_channel tx_event/rx_event"]
    end

    subgraph Rollout["codex-rollout (.jsonl)"]
        RolloutRecorder["RolloutRecorder writer task\nJsonlWriter"]
        RolloutFile["rollout-<timestamp>-<thread>.jsonl"]
    end

    subgraph State["codex-state (SQLite)"]
        StateRuntime["StateRuntime\n(pool + logs_pool)"]
        StateDb["state_*.sqlite\nthreads/..."]
        LogsDb["logs_*.sqlite\nlogs"]
    end

    subgraph Trace["codex-rollout-trace bundle"]
        ThreadTrace["ThreadTraceContext\n(no-op capable)"]
        TraceWriter["TraceWriter\nmanifest/trace.jsonl/payloads"]
        TraceBundle["bundle_dir/"]
        Reducer["replay_bundle reducer"]
        Reduced["state.json (RolloutTrace)"]
    end

    UI["clients\n(tui/app-server/mcp)"]

    Handlers --> SendRaw
    SendRaw --> PersistRollout
    PersistRollout --> RolloutRecorder --> RolloutFile
    RolloutRecorder --> StateRuntime --> StateDb
    StateRuntime --> LogsDb

    SendRaw --> ThreadTrace --> TraceWriter --> TraceBundle
    TraceBundle --> Reducer --> Reduced

    SendRaw --> EventChan --> UI
```

**Source of truth**

- `send_event_raw` persists + traces + delivers (`codex-rs/core/src/session/mod.rs:1617`).
- Rollout writer and state DB synchronization (`codex-rs/rollout/src/recorder.rs:78` and `codex-rs/rollout/src/recorder.rs:1670`).
- SQLite runtime init and pool split (`codex-rs/state/src/runtime.rs:101`).
- Trace bundle writer + reducer (`codex-rs/rollout-trace/src/writer.rs:29` and `codex-rs/rollout-trace/src/reducer/mod.rs:43`).
