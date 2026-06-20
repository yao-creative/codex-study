# ThreadRollbackResponse

This note explains what `ThreadRollbackResponse` is in the app-server v2 protocol and what it does not guarantee.

## Short Answer

`ThreadRollbackResponse` is the response payload for the `thread/rollback` RPC. It returns a single field:

```ts
type ThreadRollbackResponse = {
  thread: Thread;
};
```

Primary sources:
- [common.rs](/Users/yao/projects/codex/codex-rs/app-server-protocol/src/protocol/common.rs:547)
- [v2.rs](/Users/yao/projects/codex/codex-rs/app-server-protocol/src/protocol/v2.rs:4256)
- [ThreadRollbackResponse.ts](/Users/yao/projects/codex/codex-rs/app-server-protocol/schema/typescript/v2/ThreadRollbackResponse.ts:6)

## What `thread/rollback` Means

The request is defined by `ThreadRollbackParams`:

- `thread_id: String`
- `num_turns: u32`

The protocol comment is important: rollback drops turns from the end of the thread history, but it does **not** revert file changes made locally by the agent. Clients must revert those changes themselves.

Primary source:
- [v2.rs](/Users/yao/projects/codex/codex-rs/app-server-protocol/src/protocol/v2.rs:4244)

## What the Response Contains

`ThreadRollbackResponse.thread` is the updated thread after rollback, and its `turns` field is populated.

That matters because the generic `Thread` type only includes populated `turns` on a few APIs:

- `thread/resume`
- `thread/rollback`
- `thread/fork`
- `thread/read` when `includeTurns` is true

Primary sources:
- [v2.rs](/Users/yao/projects/codex/codex-rs/app-server-protocol/src/protocol/v2.rs:4257)
- [v2.rs](/Users/yao/projects/codex/codex-rs/app-server-protocol/src/protocol/v2.rs:5117)

## Important Limitation: The Returned Turn Items Are Lossy

The response does **not** represent a full replayable execution snapshot.

The protocol explicitly says the `ThreadItems` inside each returned `Turn` are lossy because Codex does not persist all agent interactions, including command executions. The rollback response behaves the same way as `thread/resume`.

That means:

- You get a logical thread history view after rollback.
- You do not get every original runtime event back.
- You cannot treat this as filesystem rollback or exact process-state restoration.

Primary sources:
- [v2.rs](/Users/yao/projects/codex/codex-rs/app-server-protocol/src/protocol/v2.rs:4257)
- [ThreadRollbackResponse.json](/Users/yao/projects/codex/codex-rs/app-server-protocol/schema/json/v2/ThreadRollbackResponse.json:1963)

## Mental Model

Use `ThreadRollbackResponse` as:

- the authoritative updated thread record after trimming recent turns
- a client-friendly history payload with populated turns

Do **not** use it as:

- a full execution checkpoint
- a filesystem undo
- a complete event log of the removed or remaining turns
