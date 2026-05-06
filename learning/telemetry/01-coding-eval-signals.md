# Coding Eval Signals

This note answers a narrower question than the main telemetry README:

- which recorded signals are useful for performance-based evals on actual coding?

The short answer is:

- use rollout history and trace evidence for behavior
- use repo diff and tests for correctness
- use analytics turn events for coarse KPI measurement
- do not rely on analytics drain alone for coding eval quality

## 1) Recommended Signal Priority

For coding evals, the most useful sources are:

1. rollout JSONL
2. rollout trace bundles
3. final repo/file diff
4. test results
5. selected analytics events

That order matters.

Analytics mostly tells you that a turn happened and how expensive it was.

Rollout and trace data tell you what the agent actually did.

## 2) Best Sources

### 2.1 Rollout JSONL

This is the best general-purpose source for reconstructing coding behavior.

Useful for:

- user prompt and turn context
- assistant output
- persisted runtime events
- replay and inspection
- session-level behavioral review

Why it matters for coding eval:

- you can inspect the actual interaction path
- you can see whether the agent converged, stalled, got redirected, or completed
- it is much closer to the real task transcript than analytics

Primary code:

- `codex-rs/core/src/session/mod.rs`
- `codex-rs/rollout/src/recorder.rs`

### 2.2 Rollout Trace Bundles

This is the highest-fidelity diagnostic source when enabled.

Useful for:

- exact inference request/response payloads
- tool runtime evidence
- compaction evidence
- timing-oriented debugging

Why it matters for coding eval:

- it captures the exact raw evidence behind the turn
- it is the best source when you need to explain why an agent succeeded or failed

Primary code:

- `codex-rs/rollout-trace/src/thread.rs`
- `codex-rs/rollout-trace/src/writer.rs`

### 2.3 Repo Diff and File Outcomes

For actual coding tasks, the final code change is a first-class eval artifact.

Useful for:

- correctness
- completeness
- whether the requested files changed
- whether the implementation is minimal or noisy
- reviewability

Why it matters for coding eval:

- many coding failures look successful in telemetry but fail in the diff
- behavior-based eval without checking the code artifact is weak

### 2.4 Test Results

Tests are the strongest available correctness signal when present.

Useful for:

- pass/fail outcome
- regression detection
- partial completion detection

Why it matters for coding eval:

- coding performance is not just speed or token cost
- correctness has to be anchored somewhere outside the model’s self-report

## 3) Best Analytics Events

If you need a lightweight eval dashboard, the most useful analytics event is:

- `codex_turn_event`

After that:

- `codex_turn_steer_event`
- `codex_guardian_review`
- `codex_compaction_event`

### 3.1 `codex_turn_event`

This is the best coarse KPI event for coding runs.

Most useful fields:

- `status`
- `duration_ms`
- `input_tokens`
- `output_tokens`
- `reasoning_output_tokens`
- `total_tokens`
- `model`
- `reasoning_effort`
- `approval_policy`
- `sandbox_network_access`
- `steer_count`
- `is_first_turn`

Why it helps:

- good for throughput and cost views
- useful for latency and resource comparisons across models or settings
- useful for measuring how interactive or unstable a run was

Primary schema:

- `codex-rs/analytics/src/events.rs`

### 3.2 `codex_turn_steer_event`

Useful for interactive coding evals.

Most useful fields:

- `expected_turn_id`
- `accepted_turn_id`
- `result`
- `rejection_reason`
- `created_at`

Why it helps:

- measures how often the agent needed user correction or mid-turn steering
- can separate stable coding runs from correction-heavy runs

### 3.3 `codex_guardian_review`

Useful when the coding task is affected by approval/review friction.

Most useful fields:

- `decision`
- `terminal_status`
- `failure_reason`
- `review_timeout_ms`
- `completion_latency_ms`
- token usage fields

Why it helps:

- useful when workflow latency is dominated by safety/approval boundaries
- useful for measuring blocked vs unblocked coding paths

### 3.4 `codex_compaction_event`

Useful for long-running coding tasks.

Most useful fields:

- `trigger`
- `reason`
- `implementation`
- `phase`
- `status`
- `active_context_tokens_before`
- `active_context_tokens_after`
- `duration_ms`

Why it helps:

- useful for measuring context pressure
- useful for finding long coding runs that degrade due to compaction overhead

## 4) Weak Signals for Coding Eval

These analytics events are usually weak as primary coding-performance measures:

- `codex_app_mentioned`
- `codex_app_used`
- `codex_hook_run`
- `skill_invocation`
- plugin installed/enabled/disabled/used events

Why they are weak:

- they mostly describe environment usage, not coding quality
- they can help explain variance, but they should not be the main score inputs

## 5) Important Current Limitation

The analytics turn schema already has reserved tool-count fields, but they are not currently populated.

That means these fields are presently weak or unusable for coding-performance scoring:

- `total_tool_call_count`
- `shell_command_count`
- `file_change_count`
- `mcp_tool_call_count`
- `dynamic_tool_call_count`
- `subagent_tool_call_count`
- `web_search_count`
- `image_generation_count`

This matters because those would otherwise be very useful for coding eval.

Without them, analytics alone cannot tell you:

- how many real shell actions happened
- how many file edits occurred
- how much tool work was required
- how much delegation happened

So today, that information has to come from rollout evidence, trace data, and the final repo diff.

Primary code:

- `codex-rs/analytics/src/events.rs`
- `codex-rs/analytics/src/reducer.rs`

## 6) Recommended Scoring Stack

If you want a pragmatic coding eval stack, use:

- correctness:
  - test results
  - task-specific assertions
  - repo diff inspection
- behavioral fidelity:
  - rollout JSONL
  - rollout trace bundles when available
- performance:
  - `codex_turn_event.duration_ms`
  - token usage fields
  - compaction latency
  - guardian latency where relevant
- interaction friction:
  - `steer_count`
  - `codex_turn_steer_event`
  - guardian review outcomes

## 7) Practical Mapping

If your eval question is:

- "Did the agent solve the coding task correctly?"
  - use tests + diff + rollout
- "How expensive was the coding run?"
  - use `codex_turn_event` token usage and duration
- "How much human correction did it need?"
  - use `steer_count` and `codex_turn_steer_event`
- "Did safety/approval gates slow it down?"
  - use `codex_guardian_review`
- "Did long context windows hurt performance?"
  - use `codex_compaction_event`

## 8) Bottom Line

For actual coding evals:

- rollout and trace data are the behavioral ground truth
- tests and diffs are the correctness ground truth
- analytics is best used as a coarse KPI layer on top

Analytics drain data is useful, but on its own it is too reduced to be the main substrate for judging coding performance.
