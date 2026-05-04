# Planning Overview

This document covers planning-specific architecture in `codex-rs`, separated from protocol transport details.

## 1) Planning Structure Tree

```mermaid
flowchart TD
  A[Planning in Codex]
  A --> B[Checklist Planning]
  A --> C[Plan-Mode Proposed Plan Streaming]

  B --> B1["Tool call: update_plan"]
  B --> B2["Payload: UpdatePlanArgs"]
  B2 --> B21["PlanItemArg { step, status }"]
  B --> B3["Core event: EventMsg::PlanUpdate"]
  B --> B4["App-server notif: turn/plan/updated"]
  B4 --> B41["TurnPlanUpdatedNotification"]
  B41 --> B42["TurnPlanStep[]"]

  C --> C1["Mode gate: ModeKind::Plan"]
  C --> C2["Input: assistant text chunks"]
  C2 --> C21["AssistantMessageStreamParsers"]
  C21 --> C22["PlanModeStreamState"]
  C22 --> C23["ProposedPlanItemState"]
  C --> C3["Core delta: PlanDeltaEvent"]
  C --> C4["App-server notif: item/plan/delta"]
  C4 --> C41["PlanDeltaNotification"]
  C --> C5["Final item: PlanItem { id, text }"]
```

## 2) Planning-Relevant Type Index

- `protocol/src/config_types.rs`: `ModeKind`, `CollaborationMode`, `CollaborationModeMask`
- `protocol/src/plan_tool.rs`: `UpdatePlanArgs`, `PlanItemArg`
- `protocol/src/protocol.rs`: `PlanDeltaEvent`
- `protocol/src/items.rs`: `PlanItem`
- `app-server-protocol/src/protocol/v2.rs`: `TurnStartParams`, `TurnPlanUpdatedNotification`, `TurnPlanStep`, `PlanDeltaNotification`
- `core/src/session/turn.rs`: `AssistantMessageStreamParsers`, `PlanModeStreamState`, `ProposedPlanItemState`

## 3) Cross-References

- Data structures and state control: [02-planning-data-structures.md](/Users/yao/projects/codex/learning/planning/02-planning-data-structures.md)
- Algorithm patterns: [03-planning-algorithm-patterns.md](/Users/yao/projects/codex/learning/planning/03-planning-algorithm-patterns.md)
- Protocol transport/contracts view: [planning-protocol-structures.md](/Users/yao/projects/codex/learning/protocol/planning-protocol-structures.md)
