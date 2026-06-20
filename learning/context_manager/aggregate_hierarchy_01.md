# Aggregate Hierarchy And Injection Breakdown

This repo does not define a formal `Aggregate` trait or explicit DDD aggregate-root layer.

For this writeup, "aggregate" means the real runtime ownership roots that:

- collect durable or turn-scoped state,
- own subordinate state objects,
- coordinate model-visible context injection,
- or persist the baseline used for later context diffs.

Primary sources used:

- `codex-rs/core/src/thread_manager.rs`
- `codex-rs/core/src/codex_thread.rs`
- `codex-rs/core/src/session/session.rs`
- `codex-rs/core/src/state/session.rs`
- `codex-rs/core/src/state/service.rs`
- `codex-rs/core/src/session/turn_context.rs`
- `codex-rs/core/src/context_manager/history.rs`
- `codex-rs/core/src/context_manager/updates.rs`
- `codex-rs/core/src/context/*.rs`
- `codex-rs/core-skills/src/injection.rs`
- `codex-rs/core/src/plugins/injection.rs`

## 1. Runtime Aggregate Hierarchy

Top-down ownership:

```text
ThreadManager
└── CodexThread
    └── Codex
        └── Session
            ├── SessionState
            │   └── ContextManager
            │       └── reference_context_item: TurnContextItem
            ├── SessionServices
            ├── ActiveTurn
            │   └── TurnContext
            │       ├── TurnSkillsContext
            │       ├── TurnMetadataState
            │       ├── TurnTimingState
            │       └── ToolsConfig / provider / env selection
            └── RealtimeConversationManager
```

## 2. Aggregate-by-Aggregate Breakdown

### ThreadManager

Role:
- Process-level root that creates and tracks threads.

Owns/injects into child thread creation:
- `auth_manager`
- `models_manager`
- `environment_manager`
- `skills_manager`
- `plugins_manager`
- `mcp_manager`
- `skills_watcher`
- `thread_store`
- `session_source`
- optional `analytics_events_client`

What it creates:
- `Codex`
- then wraps it in `CodexThread`

Source:
- `codex-rs/core/src/thread_manager.rs`

### CodexThread

Role:
- User-facing thread/conversation wrapper around `Codex`.
- Thread-level conduit for submit, steer, shutdown, rollout flush, and lazy context injection.

Owns:
- `codex: Codex`
- `session_configured`
- `rollout_path`
- `session_source`
- watch registration

Context interaction:
- If the session has no baseline yet, `CodexThread::inject_response_items()` forces
  `record_context_updates_and_set_reference_context_item(...)` before injecting pending items.

Source:
- `codex-rs/core/src/codex_thread.rs`

### Codex

Role:
- Queue-pair facade over the running session loop.

Owns:
- submission channel
- event receiver
- status receiver
- `session: Arc<Session>`

Injected during spawn:
- config
- auth manager
- models manager
- environment manager
- skills manager
- plugins manager
- mcp manager
- skills watcher
- conversation history
- session source
- agent control
- dynamic tools
- persistence flags
- optional analytics client
- thread store

Source:
- `codex-rs/core/src/session/mod.rs`

### Session

Role:
- Main runtime aggregate root for a thread.
- Coordinates state, services, active turn, history, rollout persistence, and context injection.

Owns:
- `state: Mutex<SessionState>`
- `services: SessionServices`
- `active_turn`
- mailbox
- conversation manager
- guardian review manager
- goal runtime

Key context responsibilities:
- `build_initial_context(&TurnContext)`
- `build_settings_update_items(...)`
- `record_context_updates_and_set_reference_context_item(&TurnContext)`
- persists rollout `TurnContextItem`

Injected into Session at construction:
- `SessionState`
- `SessionServices`
- conversation infrastructure
- mailbox
- guardian / goal / event plumbing

Source:
- `codex-rs/core/src/session/session.rs`
- `codex-rs/core/src/session/mod.rs`

### SessionState

Role:
- Durable session-scoped mutable state previously stored directly on `Session`.

Owns:
- `session_configuration`
- `history: ContextManager`
- rate limits
- dependency env values
- prompted MCP dependency set
- previous turn settings
- startup prewarm handle
- connector selection
- granted permissions
- first-turn flag

Important subordinate aggregate:
- `ContextManager`

Source:
- `codex-rs/core/src/state/session.rs`

### ContextManager

Role:
- Transcript/history aggregate for model-visible thread state.
- Stores normalized response items plus the baseline snapshot used for future diffs.

Owns:
- `items: Vec<ResponseItem>`
- `history_version`
- `token_info`
- `reference_context_item: Option<TurnContextItem>`

What gets injected into it:
- initial context items from `Session::build_initial_context(...)`
- diff/update items from `build_settings_update_items(...)`
- ordinary response items and tool outputs through `record_items(...)`

Special baseline:
- `reference_context_item` is the durable "current context snapshot" used to decide whether
  the next turn needs full reinjection or only diffs.

Source:
- `codex-rs/core/src/context_manager/history.rs`

### TurnContext

Role:
- Per-turn aggregate containing the entire effective runtime view for one turn.

Injected fields:
- config
- auth manager
- model info
- session telemetry
- provider
- reasoning effort / summary
- session source
- selected environments
- cwd
- current date
- timezone
- app-server client name
- developer instructions
- compact prompt
- user instructions
- collaboration mode
- personality
- approval policy
- permission profile
- network proxy
- windows sandbox level
- shell environment policy
- tools config
- managed features
- ghost snapshot config
- final output schema
- tool executable paths
- tool gate
- truncation policy
- dynamic tools
- `TurnSkillsContext`
- `TurnMetadataState`
- `TurnTimingState`

What it emits:
- `to_turn_context_item()` creates the persisted context baseline shape:
  `cwd`, `date`, `timezone`, approval policy, sandbox/policy data, network domain data,
  model, personality, collaboration mode, realtime flag, reasoning settings, instructions,
  JSON schema, truncation policy.

Source:
- `codex-rs/core/src/session/turn_context.rs`

### TurnSkillsContext

Role:
- Turn-local skill state.

Injected into `TurnContext`:
- `outcome: Arc<SkillLoadOutcome>`
- `implicit_invocation_seen_skills`

Source:
- `codex-rs/core/src/session/turn_context.rs`

### SessionServices

Role:
- Service dependency container injected into `Session`.

Contains:
- MCP connection manager
- unified exec manager
- shell paths
- analytics client
- hooks
- rollout trace context
- user shell
- shell snapshot sender
- exec policy manager
- auth manager
- models manager
- session telemetry
- approval store
- guardian rejection stores
- runtime handle
- skills manager
- plugins manager
- MCP manager
- skills watcher
- agent control
- network proxy
- network approval service
- state DB
- live thread handle
- thread store
- model client
- code mode service
- environment manager

This is the main dependency-injection bundle for runtime behavior.

Source:
- `codex-rs/core/src/state/service.rs`
- construction in `codex-rs/core/src/session/session.rs`

## 3. Context Aggregate Hierarchy

Model-visible context is built in two layers:

```text
TurnContext
└── TurnContextItem (persisted baseline snapshot)

Session
├── build_initial_context()
│   ├── developer message bundle
│   ├── optional multi-agent usage-hint developer message
│   ├── contextual-user message bundle
│   └── optional separate guardian developer message
└── build_settings_update_items()
    ├── developer diff message
    └── contextual-user diff message
```

Supporting fragment family:

```text
ContextualUserFragment trait
├── EnvironmentContext
├── UserInstructions
├── PermissionsInstructions
├── CollaborationModeInstructions
├── PersonalitySpecInstructions
├── RealtimeStartInstructions
├── RealtimeStartWithInstructions
├── RealtimeEndInstructions
├── AvailableSkillsInstructions
├── AvailablePluginsInstructions
├── AppsInstructions
├── SkillInstructions
├── PluginInstructions
├── ModelSwitchInstructions
├── HookAdditionalContext
├── SubagentNotification
├── ApprovedCommandPrefixSaved
├── NetworkRuleSaved
├── GuardianFollowupReviewReminder
├── TurnAborted
├── ImageGenerationInstructions
└── UserShellCommand
```

## 4. What Gets Injected Into Initial Context

Built by `Session::build_initial_context(...)`.

### Developer bundle

Possible injected sections:
- model switch instructions
- permissions instructions
- developer instructions
- memory-tool developer instructions
- collaboration mode instructions
- realtime start/end instruction
- personality spec instructions
- apps instructions
- available skills instructions
- available plugins instructions
- git commit trailer instruction

### Separate developer bundle

Possible injected section:
- multi-agent usage hint text

### Contextual user bundle

Possible injected sections:
- user instructions
- environment context

### Separate guardian developer bundle

Possible injected section:
- developer instructions when the session source is a guardian reviewer

Source:
- `codex-rs/core/src/session/mod.rs`

## 5. What Gets Injected On Context Diffs

Built by `context_manager::updates::build_settings_update_items(...)`.

### Developer diff sections

Possible injected updates:
- model switch instructions
- permissions instructions
- collaboration mode instructions
- realtime start/end instruction
- personality spec instructions

### Contextual user diff section

Possible injected update:
- `EnvironmentContext` diff

Notably incomplete:
- the code explicitly says diffing does not yet cover every model-visible item emitted by
  `build_initial_context()`.

Source:
- `codex-rs/core/src/context_manager/updates.rs`

## 6. Skill Injection Breakdown

There are two distinct skill-related injections.

### Available skill catalog injection

Injected at thread start / full reinjection:
- `AvailableSkillsInstructions`

Inputs:
- loaded skills from `turn_context.turn_skills.outcome`
- budget trimming based on model context window

Source:
- `codex-rs/core/src/session/mod.rs`

### Explicit skill content injection

Injected when the user explicitly mentions skills:
- `build_skill_injections(...)` reads each referenced `SKILL.md`
- each becomes a `SkillInjection { name, path, contents }`
- then core wraps those as model-visible items

Inputs:
- explicitly mentioned skills
- skill load outcome
- analytics client
- telemetry

Source:
- `codex-rs/core-skills/src/injection.rs`
- call path from `codex-rs/core/src/session/turn.rs`

## 7. Plugin Injection Breakdown

There are two plugin-related injections.

### Available plugin catalog injection

Injected at thread start / full reinjection:
- `AvailablePluginsInstructions`

Input:
- loaded plugins from `plugins_manager`

### Explicit plugin mention injection

Injected when plugins are explicitly mentioned:
- `PluginInstructions`

Inputs used to render it:
- mentioned plugin capability summaries
- MCP tools visible to that plugin
- enabled connectors/apps visible to that plugin

Source:
- `codex-rs/core/src/session/mod.rs`
- `codex-rs/core/src/plugins/injection.rs`

## 8. EnvironmentContext Breakdown

`EnvironmentContext` is the main contextual-user aggregate.

Fields:
- `environments`
- `current_date`
- `timezone`
- `network`
- `subagents`

Environment child shape:
- `id`
- `cwd`
- `shell`

Network child shape:
- `allowed_domains`
- `denied_domains`

Build sources:
- `from_turn_context(...)` for full injection
- `from_turn_context_item(...)` for previous baseline reconstruction
- `diff_from_turn_context_item(...)` for turn-to-turn diff emission

Source:
- `codex-rs/core/src/context/environment_context.rs`

## 9. Practical Summary

If you want the shortest accurate answer to "what are the aggregates here?":

Runtime aggregates:
- `ThreadManager`
- `CodexThread`
- `Codex`
- `Session`
- `SessionState`
- `ContextManager`
- `TurnContext`
- `SessionServices`

Context aggregates:
- `TurnContextItem`
- `EnvironmentContext`
- the `ContextualUserFragment` family

Main injection points:
- dependency injection into `SessionServices`
- per-turn runtime injection into `TurnContext`
- full context injection through `Session::build_initial_context(...)`
- diff context injection through `build_settings_update_items(...)`
- skill/plugin explicit mention injection during turn execution
