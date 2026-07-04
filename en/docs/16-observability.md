# Chapter 16: Observability — The EXPLAIN of a Prompt

> A distributed database can run `EXPLAIN` on a single SQL query: which operators it passes through, how many rows each one read, how long each took, whether there is data skew. Claude Code does the same thing to **itself** — one prompt spans multiple turns, and for each turn it records which API calls fired, which tools ran, how many tokens and dollars were spent, where it waited on a permission prompt, whether anything errored and retried — all captured as structured data and stitched together. This chapter is about how it pulls that off.

## 16.1 The Observable Path of One Prompt

Let's frame the problem precisely. You type "fix the 500 error on login" into the terminal. What follows is not "one LLM call" but an **execution chain**:

```
user prompt
  └─ turn 1: LLM request (intends to read a file) → Read tool → tool_result injected back
  └─ turn 2: LLM request (locate the bug) → Grep tool → tool_result injected back
  └─ turn 3: LLM request (propose a fix) → Edit tool → wait for your approval → execute → tool_result
  └─ turn 4: LLM request (run tests) → Bash tool → tool_result
  └─ turn 5: LLM request (summarize) → done
```

The reader's question is exactly: **how does Claude Code internally record and stitch this chain together?** This isn't "how the harness helps Claude write observable code" — it's Claude Code's ability to introspect its own runtime (self-observability).

The answer: CC doesn't fudge this with one big log file. It builds **three orthogonal observability planes plus one persistence substrate**, each owning a slice of the job:

| Plane | Records | Form | On by default? | Granularity | Lands in |
|---|---|---|---|---|---|
| **Metrics** | Aggregates: session count, tokens, cost, lines of code, decision counts, active time | OTel Counter | Needs `CLAUDE_CODE_ENABLE_TELEMETRY` (user side) / org gate (first-party BigQuery) | Low-cardinality, additive | Your own OTLP backend / Anthropic BigQuery |
| **Events (Logs)** | Discrete events: each API request, each tool result, each permission decision, each hook run | OTel Log Record | Needs `CLAUDE_CODE_ENABLE_TELEMETRY` + `OTEL_LOGS_EXPORTER` (ordinary events are not beta; only the detailed `system_prompt`/`tool` tracing events are) | One record per event | OTLP logs endpoint |
| **Trace (Spans)** | Causal tree: one interaction → each LLM request / tool call / permission wait, with parent-child + timing | OTel Span | BETA, needs enhanced/beta tracing turned on | Causal nesting | OTLP traces endpoint |
| **Transcript JSONL** | A complete, replayable session record: every message, usage, parentUuid chain | Local append-only JSONL | **On by default, present day-0** | Per-message | `~/.claude/projects/…/<sessionId>.jsonl` |

**Why four layers instead of one big log?** Because four kinds of questions demand four data structures:

- Want to chart "how many tokens this org burned this week" → you need a **low-cardinality, aggregatable** metric, and you must *never* stuff each prompt's ID into a dimension (it would explode the time-series database's cardinality).
- Want to look up "who exactly approved that `git push` last Tuesday" → you need **discrete events filterable by field**.
- Want to see "across the 5 turns of this prompt, which tool in which turn was slowest" → you need a **span tree with causal parenting** — which is precisely `EXPLAIN`'s operator tree.
- Want to **fully reproduce** a session (resume, audit, community analysis tools) → you need a **per-message, replayable** persistent record.

### The anchor that stitches one prompt together: `prompt.id`

To reassemble one prompt's observability data into "one chain," you rely on a correlation anchor. When input enters the system, `processTextPrompt` mints a UUID, stashes it in global state, and — **at the same spot** — both opens a root span and emits a `user_prompt` event, tying trace and events onto the same prompt:

```typescript
// utils/processUserInput/processTextPrompt.ts:31
const promptId = randomUUID()
setPromptId(promptId)
// …
startInteractionSpan(userPromptText)          // ← open a root span
void logOTelEvent('user_prompt', {            // ← emit a user_prompt event
  prompt_length: String(otelPromptText.length),
  prompt: redactIfDisabled(otelPromptText),   // <REDACTED> by default
  'prompt.id': promptId,
})
```

From then on, **every OTel event automatically carries this `prompt.id`**:

```typescript
// utils/telemetry/events.ts:49
// Add prompt ID to events (but not metrics, where it would cause unbounded cardinality)
const promptId = getPromptId()
if (promptId) {
  attributes['prompt.id'] = promptId
}
```

The same ID is written into the transcript's **user rows** (`sessionStorage.ts:1045` stamps `promptId` only when `message.type === 'user'`; `types/logs.ts:230`: `promptId?: string // Correlates with OTel prompt.id for user prompt messages`). The official docs make its purpose crisp:

> "The `prompt.id` attribute lets you tie all of those events back to the single prompt that triggered them."

But be precise — `prompt.id` is **not** the same field evenly spread across four layers; each layer correlates differently:

- **Events**: every OTel event carries `prompt.id` (`events.ts:49`) — this is its true "correlation key"; all of one prompt's events are grouped by it.
- **Transcript**: stamped only on **user rows** (both the user-typed row and the tool_result rows injected back — both are `user`-type messages) — it's the **anchor** for this prompt in the local record; assistant rows don't carry it.
- **Trace**: spans do **not** carry `prompt.id` (it's absent from the span attributes in `sessionTracing.ts`); an interaction's parent-child structure is expressed by **span parentage** (everything hangs under the `claude_code.interaction` root, see §16.4). It refers to the same prompt as events/transcript only because it was "opened at the same call site" (the `startInteractionSpan` alongside `logOTelEvent` above).
- **Metrics**: **deliberately carry no** prompt-level id (the `events.ts:49` comment: it would cause unbounded cardinality).

This boundary — **the discrete layer correlates by id, the aggregate layer refuses id, the causal layer uses parentage** — is the first principle of the whole telemetry design, and it recurs throughout.

The next four sections unpack these four layers.

---

## 16.2 The Metrics Plane: OpenTelemetry Export

### One switch, two parallel pipelines

There is an easily misunderstood point about the metrics plane: **it actually feeds two independent pipelines**, controlled by two entirely different switches.

```typescript
// utils/telemetry/instrumentation.ts:458
const telemetryEnabled = isTelemetryEnabled()   // CLAUDE_CODE_ENABLE_TELEMETRY
if (telemetryEnabled) {
  readers.push(...(await getOtlpReaders()))      // ① your own OTLP backend
}
// Add BigQuery exporter (for API customers, C4E users, and internal users)
if (isBigQueryMetricsEnabled()) {
  readers.push(getBigQueryExportingReader())     // ② Anthropic first-party BigQuery
}
```

- **① User-side OTLP export**: gated by `CLAUDE_CODE_ENABLE_TELEMETRY` (`instrumentation.ts:324`), sending metrics to **your own** Prometheus / OTLP collector. This is what enterprises use to build usage dashboards for their teams.
- **② First-party BigQuery export**: for API customers / Enterprise / Team plans (`isBigQueryMetricsEnabled`, `instrumentation.ts:336`), sending metrics to **Anthropic's own** BigQuery for product analytics.

The two are independent: even if you never set `CLAUDE_CODE_ENABLE_TELEMETRY`, the first-party BigQuery export may still run — but it is gated by an **org-level opt-out** (below). Understanding this keeps you from mistaking "I didn't turn on telemetry" for "nothing gets reported."

### Org-level opt-out: a switch engineered to almost never hit the network

The first-party BigQuery export asks "is this org allowed to report?" before each export. The naive implementation calls the API every export, but that would put enormous load on the startup path and the network — especially for one-shot `claude -p` calls that may run hundreds of times a day. `metricsOptOut.ts` uses a **two-tier cache** to compress it down to "roughly one API call per day":

```typescript
// services/api/metricsOptOut.ts:22
const CACHE_TTL_MS = 60 * 60 * 1000          // in-memory TTL: dedupe within a process
const DISK_CACHE_TTL_MS = 24 * 60 * 60 * 1000 // disk TTL: survives across processes
// verbatim comment: This is what collapses N `claude -p` invocations into ~1 API call/day.
```

Before exporting, `bigqueryExporter` calls `checkMetricsEnabled()` (`bigqueryExporter.ts:105`): a fresh disk cache returns immediately with zero network; only when stale does it refresh asynchronously in the background. There's also an **incident kill switch** — when `isEssentialTrafficOnly()` is true (non-essential traffic globally disabled), it returns `enabled:false` and sheds the export right at the consumer (`metricsOptOut.ts:57`). This is a textbook "observability itself must not drag down the main flow" judgment: an observability component must be cheap to turn off, tolerant of stale reads, and non-blocking by default.

### Eight metrics: a stable self-observation base

CC registers only eight internal metrics, defined together in `setMeter`:

```typescript
// bootstrap/state.ts:955
createCounter('claude_code.session.count',        …) // sessions started
createCounter('claude_code.lines_of_code.count',  …) // lines changed (type=added/removed)
createCounter('claude_code.pull_request.count',   …) // PRs created
createCounter('claude_code.commit.count',         …) // commits created
createCounter('claude_code.cost.usage',           …) // session cost (USD)
createCounter('claude_code.token.usage',          …) // tokens (type=input/output/cacheRead/cacheCreation)
createCounter('claude_code.code_edit_tool.decision', …) // accept/reject counts for Edit/Write/NotebookEdit
createCounter('claude_code.active_time.total',    …) // active seconds
```

These eight line up **one-for-one** with the metric list in the official [Monitoring docs](https://code.claude.com/docs/en/monitoring-usage) — a rare "snapshot equals current" stable point (contrast this with the events list, which expanded substantially after the snapshot, next section). By design they are all **Counters (monotonically increasing)**: in observability, prefer a counter over a gauge whenever you can, because counters are far more robust to sampling loss, restarts, and out-of-order delivery, and aggregation only needs a delta. CC even defaults the temporality preference to `delta` (`instrumentation.ts:113`, comment "the more sane default").

**Note that metrics carry no content, only "shape"**: the token count carries `type=input/output/cacheRead/cacheCreation` and `model`, but never the prompt text. The lines-of-code count carries `type=added/removed`, but never file paths or code. This is deliberate, not an oversight — see §16.8.

### Attribute assembly and cardinality control

The common attributes on every metric/event are assembled by `getTelemetryAttributes()` (`telemetryAttributes.ts:29`), and **which attributes get in is itself a configurable cardinality knob**:

```typescript
// utils/telemetryAttributes.ts:10
const METRICS_CARDINALITY_DEFAULTS = {
  OTEL_METRICS_INCLUDE_SESSION_ID: true,   // include session.id by default
  OTEL_METRICS_INCLUDE_VERSION: false,     // omit app.version by default (else every upgrade adds a dimension)
  OTEL_METRICS_INCLUDE_ACCOUNT_UUID: true,
}
```

`app.version` defaults off for the same reason as `prompt.id`: CC ships weekly, and if every version became a new metric dimension, the time-series DB would be shredded by version number. Making these env switches hands the operator the choice of "how much attribution I want vs. how much cardinality I can afford."

### Lazy-loaded exporters

An unassuming detail worth stealing: OTLP supports three protocols — grpc / http-json / http-protobuf — whose exporter packages total ~1.2MB. Rather than statically importing all of them at startup, CC does **on-demand `await import()`** inside the protocol switch:

```typescript
// utils/telemetry/instrumentation.ts:3
// OTLP/Prometheus exporters are dynamically imported inside the protocol
// switch statements below. … static imports would load all 6 (~1.2MB) on every startup.
```

A CLI tool is extremely sensitive to cold-start latency; moving a heavy dependency that's "only needed if telemetry is on and a particular protocol is used" out of the startup critical path is harness-level performance discipline.

---

## 16.3 The Events Plane: The Discrete-Event Pipeline

### One call site, two event streams

This is the second key layering to understanding CC's telemetry: **internal analytics** and **customer observability** are two parallel event streams, often emitted from the same call site. Look at how a successful API call is handled (`services/api/logging.ts`):

```typescript
// stream 1: internal analytics event (Statsig / Datadog / first-party BigQuery)
logEvent('tengu_api_success', { model, inputTokens, outputTokens, costUSD, durationMs, ttftMs, … })
// stream 2: customer OTel event (your own OTLP)
void logOTelEvent('api_request', {
  model, input_tokens, output_tokens, cache_read_tokens,
  cache_creation_tokens, cost_usd, duration_ms, speed,
})
// stream 3: end the trace span for this request (see §16.4)
endLLMRequestSpan(llmSpan, { success: true, inputTokens, … })
```

One API completion, three observability planes fed from the same spot. The division of labor between the two event streams is clear:

| | Internal analytics (`logEvent('tengu_*')`) | Customer observability (`logOTelEvent`) |
|---|---|---|
| Destination | Anthropic's Statsig/Datadog/BigQuery | Your own OTLP backend |
| Purpose | Official product analytics, A/B, behavioral regression | Your own team's usage/health dashboards |
| Scale | Hundreds of `tengu_` event names (a snapshot grep of `logEvent('tengu_*')` names yields several hundred, varying by dedup definition) | A dozen-odd documented event types |
| Gating | GrowthBook feature flags + sampling | `CLAUDE_CODE_ENABLE_TELEMETRY` |
| Privacy | `AnalyticsMetadata_…_NOT_CODE_OR_FILEPATHS` type forbids code/paths | `redactIfDisabled` redacts content by default |

That internal `tengu_` stream has an interesting type-level guardrail: `logEvent`'s metadata **disallows bare strings** — any string you want to pass must be explicitly asserted as `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` (`services/analytics/index.ts:19`) — using a compile-time type to turn "don't accidentally send code/filenames to the analytics backend" into a constraint that won't pass CI. There's also a `_PROTO_*` prefix convention that routes PII fields to access-controlled BigQuery columns and strips them before fanout to Datadog (`stripProtoFields`, `services/analytics/index.ts:45`). The `tengu_skill_loaded` event uses `_PROTO_skill_name` to send the skill name into a privileged column rather than the general metadata blob (`skillLoadedEvent.ts:23`).

### The `logOTelEvent` pipeline: the skeleton of each event

The customer-side OTel events all go through `logOTelEvent` (`utils/telemetry/events.ts:21`). It stamps three "envelope" fields on each event, then merges the common attributes and `prompt.id`:

```typescript
// utils/telemetry/events.ts:42
const attributes: Attributes = {
  ...getTelemetryAttributes(),                 // session.id / user.id / org.id / terminal.type …
  'event.name': eventName,
  'event.timestamp': new Date().toISOString(),
  'event.sequence': eventSequence++,           // monotonic within a session, for ordering
}
```

That monotonic `event.sequence` within a session is key: it's an **emission counter** (`events.ts:8`'s `let eventSequence = 0`, `++` per emit); batching/network reorders arrivals, so with it the backend can **re-sort a prompt's events back into emission order**. Note it is not wall-clock time — for concurrent/overlapping API and tool work, who truly happened first and how long each lasted comes from `event.timestamp` and the §16.4 span durations; `event.sequence` only guarantees "the emission order of discrete events is recoverable."

### What events exist in the snapshot, and how much has grown since

In the v2.1.88 snapshot, the customer events actually emitted via `logOTelEvent` are this set (each confirmable by grep): `user_prompt`, `api_request`, `api_error`, `tool_result`, `tool_decision`, plus `system_prompt` and `tool` on the BETA detailed-tracing path, plus hooks' `hook_execution_start` / `hook_execution_complete`, and `feedback_survey`.

The **current official docs**, meanwhile, list a substantially expanded set, adding `assistant_response`, `api_refusal`, `api_request_body` / `api_response_body`, `api_retries_exhausted`, `permission_mode_changed`, `auth`, `mcp_server_connection`, `internal_error`, `plugin_installed` / `plugin_loaded`, `skill_activated`, `at_mention`, `hook_registered`, `compaction`, and more.

> **Snapshot vs. current**: this chapter's metric list (8) is stable and consistent between the snapshot and the official docs; but the events list grew noticeably after v2.1.88. Any identifier mentioned outside this section that **did not turn up in a snapshot grep** — `assistant_response` / `compaction` / `skill_activated` / `permission_mode_changed` / `OTEL_LOG_RAW_API_BODIES` and the like — is stated per the official [Monitoring docs](https://code.claude.com/docs/en/monitoring-usage), marked "per official docs (beyond the v2.1.88 snapshot)," not as source-verified fact.

### Reading one event: `api_request`

Take the `api_request` event that best answers the reader (`logging.ts:718`). It fires **after every successful API call**, carrying `model` / `input_tokens` / `output_tokens` / `cache_read_tokens` / `cache_creation_tokens` / `cost_usd` / `duration_ms` / `speed`. Pull all `api_request` events with the same `prompt.id` from one prompt and sort by `event.sequence`, and you get **an emission-ordered audit stream of model calls** for that prompt: how many API calls this prompt made in total, how many tokens and dollars each cost, how much cache was hit, how long each took.

The `tool_decision` event (`permissionLogging.ts:230`) and the `tool_result` event (`toolExecution.ts:1381`) fill in "how a tool was allowed/blocked" and "how long a tool ran, whether it succeeded, and how big the result was," respectively. Put the three event types together and the **external-action ledger** of one prompt becomes queryable.

**But mark one honest boundary**: in the v2.1.88 snapshot these events correlate **at prompt granularity, not turn granularity** — `api_request` carries no turn ordinal, and `tool_decision` / `tool_result` carry **no `tool_use_id`** (the current official docs have since added `tool_use_id` to both — a post-snapshot addition). So "which tool_result maps to which api_request / which turn" — the **exact causal attribution** — can't be reconstructed from event fields alone in the snapshot; that's precisely the job of §16.4's span tree and the transcript's parentUuid edges. The events plane gives you "this prompt's ordered action audit"; the causal tree is left to the trace layer.

---

## 16.4 The Trace Plane: An Interaction's Operator Tree + the Session Transcript

This is the heart of the chapter, and the layer closest to the reader's "EXPLAIN of a prompt." It is actually carried by **two things** together: OpenTelemetry's span tree (BETA, precise causality, off by default) and the local transcript JSONL (day-0, on by default, usable by everyone).

### (A) The OTel span tree: modeling one interaction as causal nesting

`sessionTracing.ts` defines six span types whose parent-child relationships are exactly `EXPLAIN`'s operator tree:

```
claude_code.interaction              ← root, one user prompt → reply cycle
  ├─ claude_code.llm_request         ← one model call for a turn
  ├─ claude_code.tool                ← one tool call
  │    ├─ claude_code.tool.blocked_on_user   ← wall-clock time stuck "waiting for user approval"
  │    └─ claude_code.tool.execution         ← time the tool actually ran
  └─ claude_code.hook                ← one hook execution (beta detailed tracing only)
```

(Span-type definitions at `sessionTracing.ts:49`.) The root is opened by the `startInteractionSpan(userPrompt)` from §16.1 (`sessionTracing.ts:176`), recording the prompt length and `interaction.sequence`; on end it adds `interaction.duration_ms` (`sessionTracing.ts:263`).

How is the parent-child relationship established? Via `AsyncLocalStorage`. The interaction span is stashed in an ALS, and when an llm_request / tool span is later opened, the parent span is retrieved from the ALS to serve as the OTel context's parent (`sessionTracing.ts:308`, `495`). This way, even across many layers of intervening `await`, a child span correctly attaches under the current interaction.

Several design details worth learning:

- **`tool.blocked_on_user` is its own span** (`sessionTracing.ts:526`). Why time "waiting for the user to click approve" separately? Because a slow Edit could be a slow model, slow disk, or simply **the human went for coffee**. Peeling "waiting on a human" out of "executing" is what lets a trace show at a glance where the wall-clock time went — which directly decides whether you optimize the model, the IO, or the workflow. On end it records `decision` and `source` (approved or rejected, and who by), from the permissions layer (see §16.6).

- **Parallel requests must be passed the exact span**. One interaction often has multiple concurrent LLM requests (main thread, warmup, topic classifier, file-path extractor). `endLLMRequestSpan`'s comment specifically warns: if you don't explicitly pass the span returned by `startLLMRequestSpan`, attaching the response may land on "whichever happens to be last," causing a response mismatch (`sessionTracing.ts:342`). This is the most classic pitfall in concurrent tracing; CC cures it by "explicitly passing a span handle" rather than "implicitly grabbing the most recent."

- **Span-leak backstop**. The normal path deletes spans immediately via `endInteractionSpan` / `endToolSpan`; but on an aborted stream or a mid-turn exception, a span may never end. CC keeps active spans in `WeakRef`s and runs a 30-minute background cleanup interval that force-flushes and reclaims spans that are no longer referenced or have timed out (`sessionTracing.ts:79`, `100`) — the observability machinery must not itself become a memory leak.

Every span carries `duration_ms` on end, and (for LLM spans) token detail (`sessionTracing.ts:429`). So this tree is naturally an `EXPLAIN` annotated with timing and resource consumption: which turn, which tool call, which wait was the most expensive is immediately visible.

**Why is trace off by default and still BETA?** Because it is expensive and sensitive. The enabling condition is `telemetryEnabled && isEnhancedTelemetryEnabled()` (`instrumentation.ts:628`), the latter requiring `feature('ENHANCED_TELEMETRY_BETA')` at compile time plus a runtime env/GrowthBook allow (`sessionTracing.ts:126`). The more detailed "beta tracing" path (spans that even carry the system prompt and model output) has a higher bar and content redaction (see §16.7 / §16.8). Span count grows linearly with tool calls and each span carries many attributes; turning it on by default would charge both a performance and a privacy bill.

### (B) The transcript JSONL: the trace that's already there by default

If the OTel span tree is the trace you "turn on specially, for an observability backend," then the **transcript JSONL is the one present day-0, on every user's machine, and the one `/resume` and nearly every community analysis tool actually read**. It is the "execution record of one prompt" the vast majority of people can actually use.

**Path and layout**. One append-only JSONL file per session:

```typescript
// utils/sessionStorage.ts:199
function getProjectsDir() { return join(getClaudeConfigHomeDir(), 'projects') }
// utils/sessionStorage.ts:202
export function getTranscriptPath(): string {
  const projectDir = getSessionProjectDir() ?? getProjectDir(getOriginalCwd())
  return join(projectDir, `${getSessionId()}.jsonl`)
}
```

That is, `~/.claude/projects/<cwd escaped into a dir name>/<sessionId>.jsonl`. A subagent (sidechain) writes a separate `agent-<agentId>.jsonl` (`sessionStorage.ts:247`) — each delegation chain in a multi-agent run has its own trace file, kept apart from the main thread.

**It is a parentUuid linked list**. Every message written is stamped with a set of fields, among them a `parentUuid` pointing at the previous one:

```typescript
// utils/sessionStorage.ts:1039
const transcriptMessage: TranscriptMessage = {
  parentUuid: isCompactBoundary ? null : effectiveParentUuid,
  logicalParentUuid: isCompactBoundary ? parentUuid : undefined,
  isSidechain,
  teamName, agentName, agentId,
  promptId: message.type === 'user' ? (getPromptId() ?? undefined) : undefined, // ← same as OTel prompt.id
  ...message,           // usage / content / tool_use / …
  sessionId, version, gitBranch, cwd, userType, entrypoint,
}
```

(Field definitions at `types/logs.ts:221`, `TranscriptMessage`.) A few subtleties:

- **A tool_result's parent points at the assistant message that issued it**, not merely "the previous one." The code handles this explicitly: if a message carries `sourceToolAssistantUUID`, that becomes the `parentUuid` (`sessionStorage.ts:1031`). So "a given tool result" can be traced precisely back to "which tool_use in which assistant turn requested it" — causality is not guessed from timestamps, it's an explicit edge.
- **A compaction boundary breaks the chain but keeps a logical link**. Context compaction replaces a stretch of history with a summary; at that point `parentUuid` is set to `null` (a physical break, to keep the prompt cache stable), but `logicalParentUuid` preserves the original parent pointer, bridged on read (`sessionStorage.ts:1040`). Compaction doesn't sever the trace's logical continuity.
- **`promptId` is stamped only on user messages** — the very landing spot of §16.1's join key in the persistence layer.

**How do you stitch it into "the multiple turns of one prompt"?** The workhorse is the **`parentUuid` edges**: follow `parentUuid` from any message back to the root, or expand downward along child edges, and you get the full causal chain — an assistant turn, the tool_use it issued, and the returning tool_result (whose parent pointer points precisely at the assistant row that issued it) all linked. `promptId` here is a **locating anchor, not a grouping key**: it's stamped only on user rows (assistant rows lack it), so you use it to **find the starting user row of a given prompt**, then walk `parentUuid`/child edges to recover that prompt's assistant-and-tool causality — **grouping by `promptId` alone will not collect every message** (it misses the assistant rows that carry no promptId). That's the transcript version of `EXPLAIN` — and it's already on your disk by default, no telemetry required.

**The read-only entry points**. Users need not parse JSONL by hand; CC bundles several observation panels: `/cost` (this session's token/cost/duration summary, see §16.5), `/status`, `/context`, and the debug log opened by `--debug`. Community tools (like ccusage, which tallies usage from the transcript, or claude-tap, which captures the wire) are all built on this JSONL or its equivalent.

### Worked example: the cross-turn trace of `/goal` and `/loop`

Enough abstraction — two real features ground "how one task across many turns is traced." The following comes from static string extraction of the 2.1.201 client binary plus plaintext reverse-proxy capture (**post-snapshot reverse-engineered facts**, not v2.1.88 source, separately labeled):

- **`/goal` (session-scoped autonomous loop)** carries trace fields natively. After you set a goal, the transcript gets a `goal_status` attachment with fields `{condition, iterations, durationMs, tokens, met}` — **how many rounds each goal iterated, how long it took, how many tokens it burned, whether it was met** — all landing in the session record. Accompanying telemetry events: `tengu_goal_achieved` / `tengu_goal_failed` / `tengu_goal_restored_on_resume`. This is exactly "the execution chain of one goal across many turns" made concrete: a prompt-based Stop hook dispatches an independent evaluator LLM to judge yes/no after each round, and each round's iteration/duration/tokens accumulate into `goal_status`.
- **`/loop` (interval / self-paced loop)** tracks each tick of self-scheduling with a set of `tengu_` events: `tengu_loop_dynamic_wakeup_scheduled` / `_ends_turn` / `_aged_out`, `tengu_loop_keepalive_fired`, `tengu_loop_ended`, plus the background KAIROS daemon's `tengu_kairos_*`. Each loop tick re-injects the same prompt and leaves a trace in this event stream.

Both features illustrate the same thing: **CC's "one task across many turns" is not a black box** — it either lands as a structured attachment in the transcript (goal) or as a stream of semantic `tengu_` events (loop), and either path can reconstruct "how many rounds this autonomous run took, and the state of each round."

---

## 16.5 Cost and Token Accounting: Accrued Per Turn

Behind the `/cost` panel is `cost-tracker.ts`'s per-turn accrual. On each API success, `addToTotalSessionCost` accrues this call's usage into a per-model summary and **feeds two metric counters at the same time**:

```typescript
// cost-tracker.ts:291
getCostCounter()?.add(cost, attrs)                                    // claude_code.cost.usage
getTokenCounter()?.add(usage.input_tokens,  { ...attrs, type: 'input' })
getTokenCounter()?.add(usage.output_tokens, { ...attrs, type: 'output' })
getTokenCounter()?.add(usage.cache_read_input_tokens ?? 0,     { ...attrs, type: 'cacheRead' })
getTokenCounter()?.add(usage.cache_creation_input_tokens ?? 0, { ...attrs, type: 'cacheCreation' })
```

`attrs` carries `model`, plus `speed:'fast'` in fast mode. Tokens are split into input/output/cacheRead/cacheCreation four ways — because the four have **different unit prices**: a cache read is far cheaper than fresh input, so only splitting them lets you cost it correctly. CC even recursively bills "advisor" sub-calls separately and emits a `tengu_advisor_tool_token_usage` event (`cost-tracker.ts:304`), ensuring derived-call cost isn't dropped.

**Cross-session continuity**. Cost state is saved into the project config (`saveCurrentSessionCosts`, `cost-tracker.ts:143`); when you `/resume` the same session, `restoreCostStateForSession` restores it after matching by sessionId (`cost-tracker.ts:130`) — resume a session and `/cost` doesn't start from zero. On process exit, the `useCostSummary` hooked by `costHook.ts` prints the total and persists it (`costHook.ts:6`); the output of `formatTotalCost` (`cost-tracker.ts:228`: Total cost / duration API+wall / code changes / usage by model) is the summary block you see in the terminal.

This section answers "what resource each stage consumes" in the reader's `EXPLAIN`: tokens and dollars aren't a one-shot total, they're a runtime quantity **accrued immediately on each API call, dimensioned by model and token type, and continuable across resume**.

---

## 16.6 Permission Decision Logging: An Observable "Why Allowed / Blocked"

Another link in `EXPLAIN` is "whether each action was normal, whether it was blocked." CC records every tool-permission decision as observable data. The core is `permissionLogging.ts`'s `logPermissionDecision` — the **single exit for all approve/reject** — where one decision fans out to four destinations:

```typescript
// hooks/toolPermission/permissionLogging.ts:181 (simplified)
function logPermissionDecision(ctx, args, permissionPromptStartTimeMs?) {
  // ① internal analytics event: a different event name per source, for an approval funnel
  //    tengu_tool_use_granted_in_config / _in_prompt_permanent / _by_classifier / …
  //    tengu_tool_use_rejected_in_prompt / _denied_in_config
  // ② code-edit tool counter (Edit/Write/NotebookEdit)
  getCodeEditToolDecisionCounter()?.add(1, { decision, source, tool_name, language })
  // ③ store the decision on toolUseContext, for the trace's tool.blocked_on_user span to read
  toolUseContext.toolDecisions.set(toolUseID, { source, decision, timestamp })
  // ④ customer OTel event
  void logOTelEvent('tool_decision', { decision, source, tool_name })
}
```

A few points of observational value:

- **`source` structures "who approved it"**: `config` (allowlist auto-approve), `hook`, `user_permanent` / `user_temporary` (the user clicked "always/this time"), `classifier`, `user_reject` / `user_abort`. When auditing, you can distinguish "was this dangerous op released by a human, or auto-released by a rule" — decisive for a security post-mortem.
- **`waiting_for_user_permission_ms` quantifies "waiting on a human"** (`permissionLogging.ts:189`): recorded only when a prompt was actually shown, not for auto-approvals. It is the same thing as §16.4's `tool.blocked_on_user` span, in a different presentation.
- **The decision is written back into `toolUseContext.toolDecisions`**, so the trace layer's `tool.blocked_on_user` span can be labeled with the accurate `decision`/`source` on end (`toolExecution.ts:1171`) — the three planes are wired together at this permission point.

For the permission model itself (`ask`/`allow`/`deny`, the classifier, hook decisions), see [Chapter 12: Permissions & Security](/en/docs/11-permission-security.md); this chapter only cares about "how the decision is observed."

---

## 16.7 First-Party / Internal-Only Observability (Honest Boundaries)

Not all observability machinery is for external users. Being clear about which you can use, and which is internal-only or eliminated at compile time, keeps you from misjudging.

- **`ant-trace`: an empty shell in external builds**. This command's implementation in the snapshot is a one-line stub: `export default { isEnabled: () => false, isHidden: true, name: 'stub' }` (`commands/ant-trace/index.js`). It's a live specimen of `feature()` compile-time dead-code elimination — **the external package physically contains no real implementation, only a forever-disabled placeholder**. It's also a reminder: not finding a feature's full logic in the public artifact doesn't mean the feature doesn't exist; it's been stripped at compile time (see the methodology: extracting "text" is verifiable, extracting "logic that was stripped" is not).

- **Perfetto local tracing: Ant-only, compile-gated**. This is a local trace viewable as a flame graph in the Perfetto UI (interaction / llm_request / tool spans plus a swarm's agent hierarchy), but **not something external users can just switch on**: the source header reads "Perfetto Tracing for Claude Code (Ant-only)" and "ant-only and eliminated from external builds" (`perfettoTracing.ts:2`, `:7`), and initialization is entirely wrapped in the `feature('PERFETTO_TRACING')` compile gate (`perfettoTracing.ts:260`). That means `CLAUDE_CODE_PERFETTO_TRACE=1` only takes effect when the Perfetto feature is compiled into the build — in external builds this code is DCE'd out and the env var does nothing. It reuses the same start/end instrumentation as the §16.4 OTel spans but lands in a local file on a separate first-party path.

- **First-party BigQuery export**: as §16.2 covered, gated by org-level `metricsOptOut`, going to Anthropic's BigQuery, not handled by external users.

- **FPS self-observation**: `fpsTracker.ts` / `context/fpsMetrics.tsx` track terminal render frame rate (average / low-1%), persisted alongside cost at session end (`cost-tracker.ts:158`). A TUI folding its own render smoothness into self-observation is a fine touch of UX engineering (renderer details in [Chapter 14: User Experience Design](/en/docs/12-user-experience.md)).

- **The beta-tracing visibility matrix**: the detailed-tracing path (`betaSessionTracing.ts`) can also write the system prompt, model output, and tool I/O into spans, but under a strict visibility table — `thinking_output` is **ant-only** (the matrix at `betaSessionTracing.ts:12` plus the `USER_TYPE === 'ant'` gate at `logging.ts:746`); external users cannot get the model's reasoning text even with detailed tracing fully on. The full system prompt is emitted only on this beta path, and only once per unique hash (`betaSessionTracing.ts:266`), to avoid re-reporting large text.

> As for the observation/defense instrumentation CC turns **back on reverse-engineers** (anti-distillation fake-tool injection, steganographic fingerprints, etc.), that's a separate topic; this project takes a clean-room reading and does not go into it — see the repository disclaimer and the reverse-engineering methodology notes.

---

## 16.8 Privacy and Security Boundaries: What Can Be Observed / What Is Deliberately Not Exported

Observability and privacy inherently pull against each other: the more you record the easier to query, but also the more dangerous. CC's overarching principle is — **export only "shape" by default, content strictly opt-in**.

"Shape" means metadata like length, token count, duration, model name, tool name, decision type; "content" means the prompt text, the model's reply, a tool's input/output, code and file contents. The former is exported by default; the latter must be enabled item by item:

| Content | Switch (env) | Default behavior | Source / basis |
|---|---|---|---|
| User prompt text | `OTEL_LOG_USER_PROMPTS` | `<REDACTED>`, only `prompt_length` exported | `events.ts:17` `redactIfDisabled` |
| Assistant response text | `OTEL_LOG_ASSISTANT_RESPONSES` | redacted, only length exported | per official docs (beyond snapshot) |
| Tool parameters/command | `OTEL_LOG_TOOL_DETAILS` | omitted/redacted | `toolExecution.ts:1135`, `metadata.ts:87` |
| Tool I/O content | `OTEL_LOG_TOOL_CONTENT` | omitted | `sessionTracing.ts:738` |
| Raw API request/response bodies | `OTEL_LOG_RAW_API_BODIES` | not exported | per official docs (variable absent from v2.1.88 snapshot) |
| Full system prompt | beta detailed-tracing path only | not in ordinary export | `betaSessionTracing.ts:266` |
| Model thinking text | `USER_TYPE=ant` only | never exported externally | `logging.ts:746` |

A few design intents worth naming:

1. **Code and file contents never enter metrics/events**. The docs state plainly, "Raw file contents and code snippets are not included in metrics or events." On the source side, the `AnalyticsMetadata_…_NOT_CODE_OR_FILEPATHS` type turns it into a compile-time constraint (§16.3). In the default observable surface you can see "changed N lines of TypeScript," but not what was changed.
2. **Privacy gating and cardinality control are two faces of the same knob**. `prompt.id` goes into events only, not metrics; `app.version` is controlled by `OTEL_METRICS_INCLUDE_VERSION` and **defaults off in the shared attribute set** (`telemetryAttributes.ts:40`, a function used for metrics, events, and spans alike, so turning it off is global — though the cardinality-explosion concern is sharpest for metrics). These both guard against cardinality explosion and "shove less attributable info into an aggregation backend." In observability design, privacy and cost often point at the same decision.
3. **An honest note on `OTEL_LOG_RAW_API_BODIES`**: the current official docs do have this switch (with `api_request_body` / `api_response_body` events, truncated at 60KB, with a file mode), but it is **not in the v2.1.88 snapshot** — it postdates the snapshot. This chapter states it per the official docs, not as source-verified fact.
4. **OTel export config does not propagate to subprocesses, but trace context does**. The docs are explicit: "Claude Code doesn't pass `OTEL_*` environment variables to the subprocesses it spawns" — the Bash tool, hooks, MCP servers, and language servers cannot get your OTLP endpoint/headers/credentials, preventing observation config from leaking to third-party processes. **But that isn't "nothing is passed"**: per the current official docs, when tracing is active CC injects the W3C trace context (`TRACEPARENT`, controlled by `CLAUDE_CODE_PROPAGATE_TRACEPARENT`) into Bash/PowerShell subprocesses, so a subprocess's spans attach to the same trace. What's passed is "which trace this is" correlation info, not "where to export, with what credentials" config — keep the two distinct. (`TRACEPARENT` / `CLAUDE_CODE_PROPAGATE_TRACEPARENT` do not appear in the v2.1.88 snapshot; they postdate it and are stated per the official docs.)

---

## Summary

Claude Code's observability is not "dump a pile of logs," but a **plane-separated, use-case-tailored, privacy-first** introspection system. It answers the reader's `EXPLAIN of a prompt` head-on: a prompt's multiple turns, each turn's API and tool actions, their timing/tokens/cost, where it waited on a human, the decision source, whether it errored — with events grouped by `prompt.id`, traces stitched by span parentage, and transcript causality reconstructed from `parentUuid` edges. Core design decisions:

| Design decision | Reason |
|---|---|
| Four-layer split (Metrics / Events / Trace / Transcript) | Aggregation, discrete query, causal analysis, full replay are four problems, each needing its own data structure |
| `prompt.id` correlates events, anchors transcript user rows; trace uses span parentage | Discrete layer correlates by id, causal layer by parentage, aggregate layer refuses id — each layer uses the fittest correlation |
| `prompt.id` is event-only; `app.version` defaults off in the shared attribute set | High-cardinality dimensions blow up a time-series DB — cardinality discipline |
| All metrics are monotonic Counters + delta temporality | Robust to loss/restart/reordering; aggregation only needs a delta |
| Dual pipelines: internal `tengu_` and customer OTel | Official product analytics and users' own observability stay independent, emitted concurrently from one call site |
| `tool.blocked_on_user` as its own span | Peel "waiting on a human" out of "executing" for accurate wall-clock attribution |
| Concurrent LLM requests pass an explicit span handle | Cures response mismatch in parallel tracing |
| WeakRef + 30-min span cleanup backstop | The observability machinery must not itself leak memory |
| transcript = parentUuid linked list + tool_result points at the issuing assistant | Causality by explicit edges, not guessed from timestamps |
| metricsOptOut two-tier cache + kill switch | Observability must not drag down the main flow: cheap to disable, tolerant of stale reads |
| Export "shape" by default, content opt-in item by item | The observability-vs-privacy tension resolved by "redact by default + explicit enable" |
| Lazily `import()` exporters | Keep heavy deps off the cold-start critical path |

> **Version note**: this chapter does a source-level reading against the v2.1.88 leaked source snapshot, cross-checked with the official [Monitoring docs](https://code.claude.com/docs/en/monitoring-usage). The 8 metrics are stable and consistent between snapshot and current docs; the events list expanded substantially after the snapshot, and every identifier beyond the snapshot is marked "per official docs." The `/goal` and `/loop` trace details come from post-snapshot reverse engineering of the 2.1.201 client (static strings + reverse-proxy capture), separately labeled and not conflated with snapshot-verified facts.
