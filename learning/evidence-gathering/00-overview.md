# Evidence Gathering Overview

This folder explains the main evidence-gathering architecture around risky actions and post-hoc debugging in `codex-rs`.

The important split is:

- `monitor`: live risk triage for actions that are about to run.
- `guardian`: approval review that packages transcript evidence plus the exact planned action for a reviewer model.
- `rollout-trace`: opt-in local forensic capture for reconstructing what happened after the fact.

`monitor` and `guardian` are part of the execution path for risky actions. `rollout-trace` is diagnostic infrastructure that observes runtime behavior without trying to decide policy on the hot path.

## 1) System Overview

```mermaid
flowchart TD
  A[Pending risky action]
  A --> B[MCP approval path in core/src/mcp_tool_call.rs]

  B --> C{Auto-approved by policy?}
  C -->|Yes| D[ARC monitor]
  C -->|No| E[Interactive approval path]

  D --> D1[Build compact history + action payload]
  D1 --> D2[POST to /codex/safety/arc]
  D2 --> D3{Outcome}
  D3 -->|Ok| F[Continue without interruption]
  D3 -->|AskUser| E
  D3 -->|SteerModel| G[Block and interrupt]

  E --> E1{Approval reviewer routed to Guardian?}
  E1 -->|Yes| H[Guardian review session]
  E1 -->|No| I[Direct user / MCP elicitation prompt]

  H --> H1[Collect transcript evidence]
  H1 --> H2[Attach exact planned action JSON]
  H2 --> H3[Run read-only child review turn]
  H3 --> H4[Approval decision]

  F --> J[Tool executes]
  I --> J
  H4 --> J

  J --> K[Optional rollout-trace capture]
  K --> K1[trace.jsonl + payloads/*.json]
  K1 --> K2[Offline reducer]
  K2 --> K3[state.json semantic graph]
```

## 2) Runtime Control Flow

```mermaid
sequenceDiagram
  participant Model as Model / Tool Planner
  participant MCP as MCP Approval Path
  participant ARC as ARC Monitor
  participant Guardian as Guardian Review
  participant User as User / UI
  participant Tool as Tool Runtime
  participant Trace as Rollout Trace

  Model->>MCP: Propose MCP tool call
  MCP->>MCP: Check approval mode and remembered approvals
  alt Auto-approved by policy
    MCP->>ARC: monitor_action(action, compact_history)
    ARC-->>MCP: Ok / AskUser / SteerModel
  end

  alt AskUser or prompt required
    alt Routed to Guardian
      MCP->>Guardian: review_approval_request(...)
      Guardian-->>MCP: Accept / Decline / escalation
    else Direct approval prompt
      MCP->>User: request_user_input / elicitation
      User-->>MCP: decision
    end
  end

  alt Approved
    MCP->>Tool: Execute tool
    Tool->>Trace: Emit runtime observations if enabled
  else Blocked
    MCP-->>Model: Interrupt or decline
  end
```

## 3) Ownership Map

- `core/src/mcp_tool_call.rs`: main orchestration for MCP approval, ARC monitor, Guardian routing, and direct user prompts.
- `core/src/arc_monitor.rs`: compact evidence packaging for live safety monitoring.
- `core/src/guardian/prompt.rs`: transcript evidence selection and prompt assembly for approval review.
- `core/src/guardian/review_session.rs`: executes the Guardian review turn in read-only mode.
- `rollout-trace/src/thread.rs`: thread-scoped evidence producer used by runtime code.
- `rollout-trace/src/writer.rs`: raw event and payload writer.
- `rollout-trace/README.md`: best high-level description of the trace architecture.

## 4) Why the Design Is Split

- `monitor` is optimized for fast allow / escalate / block decisions before a risky action proceeds.
- `guardian` is optimized for richer authorization review with explicit transcript evidence and exact action JSON.
- `rollout-trace` is optimized for replayability and debugging, not live policy decisions.

## 5) Cross-References

- Monitor deep dive: [01-monitor.md](/Users/yao/projects/codex/learning/evidence-gathering/01-monitor.md)
- Rollout trace deep dive: [02-rollout-trace.md](/Users/yao/projects/codex/learning/evidence-gathering/02-rollout-trace.md)
