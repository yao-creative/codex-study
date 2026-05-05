# Tools Overview

This note describes how Codex builds and executes tools, focusing on the tool registry control flow.

## End-to-end flow

1. Build tool availability for the turn.
- `built_tools(...)` gathers MCP tools, connector exposure, deferred tools, and parallel MCP server settings.
- It then constructs `ToolRouter::from_config(...)`.

2. Build a registry plan from config.
- `build_tool_registry_plan(...)` creates a plan with:
  - tool specs shown to the model
  - handler kinds for runtime dispatch
- The plan is config-driven (`shell_type`, code mode, MCP availability, search tool, etc.).

3. Materialize specs + concrete handlers.
- `build_specs_with_discoverable_tools(...)` converts plan handler kinds into concrete Rust handler instances (`ShellHandler`, `McpHandler`, `ApplyPatchHandler`, etc.).
- `ToolRegistryBuilder` stores:
  - specs (`ConfiguredToolSpec`)
  - `ToolName -> handler` map
- `builder.build()` returns `(specs, ToolRegistry)`.

4. Expose model-visible tools.
- Prompt construction uses `router.model_visible_specs()`.
- Deferred dynamic tools can be hidden from model-visible specs while still remaining loadable through search/discovery paths.

5. Parse model output into a typed tool call.
- On stream item completion, `handle_output_item_done(...)` calls `ToolRouter::build_tool_call(...)`.
- This converts `ResponseItem` into `ToolCall` with a typed payload:
  - function
  - MCP
  - tool_search
  - local_shell/custom

6. Execute tool call through runtime gate.
- `ToolCallRuntime::handle_tool_call_with_source(...)` checks `tool_supports_parallel(...)`.
- It uses a read/write lock:
  - read lock for tools allowed in parallel
  - write lock for serialized execution
- Then it invokes router dispatch.

7. Registry dispatch executes the handler.
- `ToolRegistry::dispatch_any(...)`:
  - resolves handler by `ToolName`
  - validates payload kind compatibility
  - runs pre-tool-use hooks
  - checks if tool is mutating and waits on the tool gate
  - executes handler
  - runs post-tool-use hooks
  - records telemetry/traces and goal progress
  - returns `AnyToolResult`

8. Return tool output back to model loop.
- Runtime converts `AnyToolResult` into `ResponseInputItem` and pushes it into conversation history.
- The sampling loop continues with a follow-up request using this tool output.

## Why the registry exists

The registry separates:
- tool definition/visibility (specs shown to model)
- runtime behavior (handler implementations)

That indirection allows the system to:
- change tool exposure per turn/config
- route multiple names/namespaces to shared handlers
- enforce consistent pre/post hooks, telemetry, and safety checks in one dispatch path
- support MCP + local + dynamic tools with the same execution contract
