# Gemini CLI Replatforming Design

## Goals

1. **Fast ADK Replatforming:** Transition the default Gemini CLI agent to use
   the ADK execution engine to reduce maintenance overhead on the legacy run
   loop.
2. **Reuse Existing Frameworks:** Minimize UI and internal API churn by keeping
   existing callback mechanisms and frameworks. Specifically, we will _not_
   rewrite the existing tool approval and model fallback mechanisms to use ADK's
   `elicitation_request`/`elicitation_response` events. We will reuse the
   existing message bus and scheduler integrations.
3. **Unified Core Engine:** Ensure the main interactive chat, non-interactive
   mode, and subagents execute via the exact same core ADK engine.

## Non-Goals

1. **No Public SDK Readiness:** We are no longer refactoring the core codebase
   to expose a polished, public-facing SDK.
2. **No BYOA / Multi-Agent Orchestration:** We are explicitly not building
   extensibility for third-party agents or external runtime orchestration.
3. **No UI Modernization for Elicitations:** We will not refactor the React TUI
   or underlying message bus to adopt standard ADK elicitation events. Tool
   approvals and system interrupts will continue to use the legacy callback
   bridges.

---

## Milestones

1. AgentSession Interface Create `AgentSession` abstraction flexible enough to
   support the legacy runtime and ADK.
2. Adapting the Non-Interactive Runtime Adapt the existing non-interactive CLI
   to conform to the new `AgentSession` API.
3. Initial TUI Adaptation & Session Creation Move the main agent behind the
   `AgentSession` interface, adapting the TUI via `useAgentStream` and
   `LegacyAgentProtocol` while reusing the message bus. **Includes implementing
   the Agent Creation Factory/Function.**
4. Subagent Orchestration Decouple subagents from the legacy runtime by wrapping
   them in the `AgentSession` interface.
5. ADK Execution Engine Build the unified ADK agent conforming to
   `AgentSession`, handling all modes (Main, Non-Interactive, Subagents).

## Progress

| Milestone                                    | Owner                            | Status      | Relevant PRs / Branches                                                                                                                                                                                                                                                                                                                                      | Time Estimate |
| :------------------------------------------- | :------------------------------- | :---------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------ |
| 1. AgentSession Interface                    | Adam Weidman, Michael Bleigh     | ✅ Complete | [PR #22270](https://github.com/google-gemini/gemini-cli/pull/22270), [PR #23159](https://github.com/google-gemini/gemini-cli/pull/23159), [PR #23548](https://github.com/google-gemini/gemini-cli/pull/23548)                                                                                                                                                | -             |
| 2. Adapting the Non-Interactive Runtime      | Adam Weidman                     | ✅ Complete | [PR #22984](https://github.com/google-gemini/gemini-cli/pull/22984), [PR #22985](https://github.com/google-gemini/gemini-cli/pull/22985), [PR #22986](https://github.com/google-gemini/gemini-cli/pull/22986), [PR #24439](https://github.com/google-gemini/gemini-cli/pull/24439), [PR #22987](https://github.com/google-gemini/gemini-cli/pull/22987)      | -             |
| 3. Initial TUI Adaptation & Session Creation | Michael Bleigh                   | 🚧 WIP      | [PR #24275](https://github.com/google-gemini/gemini-cli/pull/24275), [PR #24287](https://github.com/google-gemini/gemini-cli/pull/24287), [PR #24292](https://github.com/google-gemini/gemini-cli/pull/24292), [PR #24297](https://github.com/google-gemini/gemini-cli/pull/24297), [Issue #25046](https://github.com/google-gemini/gemini-cli/issues/25046) | 3-5 days      |
| 4. Subagent Orchestration                    | Adam Weidman                     | 🚧 WIP      | [PR #25302](https://github.com/google-gemini/gemini-cli/pull/25302), [PR #25303](https://github.com/google-gemini/gemini-cli/pull/25303)<br>Branches: `agent-session/local-invocation`, `agent-session/remote-invocation`, `agent-session/agent-tool`                                                                                                        | 3-5 days      |
| 5. ADK Execution Engine                      | Adam Weidman, Alexey Kalenkevich | 🚧 WIP      | Alexey Prototype: `eeb9301`                                                                                                                                                                                                                                                                                                                                  | 1-2 weeks     |

_(Note: Previous milestones related to "Unified Elicitations" and "Adopt ADK
Primitives" have been removed as per the updated Non-Goals)._

---

## Detailed Design

### 1. Milestone 1: AgentSession Interface

The `AgentSession` interface is the core abstraction that decouples the TUI from
the specific agent implementation by proposing a purely event-driven loop
boundary (`AgentEvent` streams). _For full rationale and design, see the
[Gemini CLI Agents design document](https://docs.google.com/document/d/1Zv2_VuVNc-PtsFIU5HYApdmaC3EyK5_fNeivYUtL1fs/edit?tab=t.0#heading=h.ok5bx3z7fmr8)._

### 2. Milestone 2: Adapting the Non-Interactive Runtime

The non-interactive runtime conforms to the `AgentSession` API via a legacy
protocol adapter. This adapter translates internal message bus and scheduler
events into standard agent event streams. Because this mode does not require
user interaction, it bypasses complex UI features like tool approvals.

This capability is enabled via the following experimental setting in
`.gemini/settings.json`:

```json
{
  "experimental": {
    "adk": {
      "agentSessionNoninteractiveEnabled": true
    }
  }
}
```

### 3. Milestone 3: Initial TUI Adaptation & Session Creation

Adapting the interactive TUI to consume the `AgentSession` interface is a
complex, multi-layered effort. The core mechanism is an event-streaming hook
that subscribes to the session.

#### Current Event Handling State (Legacy Mechanism)

The legacy system relies on a central `Scheduler` and a `MessageBus` to
orchestrate tool execution and approvals. This is the mechanism we are retaining
to avoid a full UI refactor:

- **Tool Approvals**: When a tool requires user confirmation, the scheduler
  calls `resolveConfirmation`
  ([scheduler.ts:L663](file:///Users/adamfweidman/Desktop/adk-int/gemini-cli/packages/core/src/scheduler/scheduler.ts#L663))
  and updates the tool's status to `AwaitingApproval` in `confirmation.ts`
  ([confirmation.ts:L162](file:///Users/adamfweidman/Desktop/adk-int/gemini-cli/packages/core/src/scheduler/confirmation.ts#L162)).
- **UI Notification**: The `SchedulerStateManager` publishes a
  `TOOL_CALLS_UPDATE` event on the message bus
  ([state-manager.ts:L254](file:///Users/adamfweidman/Desktop/adk-int/gemini-cli/packages/core/src/scheduler/state-manager.ts#L254)).
  The UI hook `useToolScheduler.ts` subscribes to this event to update the React
  state
  ([useToolScheduler.ts:L178](file:///Users/adamfweidman/Desktop/adk-int/gemini-cli/packages/cli/src/ui/hooks/useToolScheduler.ts#L178)).
- **User Interaction**: If any tool requires approval, the TUI transitions to a
  waiting state
  ([useGeminiStream.ts:L174](file:///Users/adamfweidman/Desktop/adk-int/gemini-cli/packages/cli/src/ui/hooks/useGeminiStream.ts#L174))
  and renders the `ToolConfirmationQueue.tsx` component.
- **Response Loop**: Once the user makes a decision, `ToolActionsContext.tsx`
  publishes a `TOOL_CONFIRMATION_RESPONSE`
  ([ToolActionsContext.tsx:L150](file:///Users/adamfweidman/Desktop/adk-int/gemini-cli/packages/cli/src/ui/contexts/ToolActionsContext.tsx#L150)).
  The scheduler, which is blocked in `waitForConfirmation`
  ([confirmation.ts:L168](file:///Users/adamfweidman/Desktop/adk-int/gemini-cli/packages/core/src/scheduler/confirmation.ts#L168)),
  receives the response via the message bus and resumes execution.

#### Transition Plan

To minimize UI churn, the `AgentSession` adapter (like `LegacyAgentProtocol` or
a new interactive adapter) will continue to leverage this legacy infrastructure.
It will subscribe to the message bus and translate these complex flows into
simpler `AgentEvent` streams where possible, while allowing the existing TUI
components to continue handling the heavy lifting of user approvals. Moving the
interactive UI behind the event stream requires building a new protocol adapter
to bridge the interactive session to the legacy engine.

Achieving full TUI parity means addressing many granular lifecycle states. Key
areas of work include:

- **Event Parity:** Parity for standard tool calls, MCP tool calls, and subagent
  display.
- **UI Coalescence:** Leveraging `agent_start` and `agent_end` events for robust
  UI state management.
- **Tool-Controlled Display:** Implementing display modes controlled by the
  agent session.
- **Command Routing:** Pushing client-initiated command routing (e.g., slash
  commands) down to the session, and ensuring `abort` works properly.
- **System Notices:** Generic notice events for system notifications.

**Note on Elicitations & Event Routing:** We originally planned to refactor the
TUI to use ADK `elicitation` events for tool approvals. Given the new mandate
for a fast replatforming, this refactor is dropped. We will move read-only
observational events (like `tool_update`, `message`, `usage`) behind the new
`AgentSession` interface stream. However, interactive/blocking events like tool
approvals and elicitations will _not_ move to the new interface and will
continue to use the legacy message bus and scheduler callbacks to minimize UI
churn.

To handle session creation, we'll need a factory or creation function that
injects the correct protocol adapter (and eventually the ADK agent) based on the
configuration.

_(Note: This section is intentionally left open so others can add more color to
the implementation details.)_

> [!IMPORTANT] **TODO (@Jacob Richman)**: Scope out the extensive work required
> for full TUI adaptation. Refer to issue
> [#22702](https://github.com/google-gemini/gemini-cli/issues/22702) for more
> context on the work to be scoped.

### 4. Milestone 4: Subagent Orchestration

Subagents invoke both local and remote execution via a unified `AgentTool` that
wraps them in `AgentSession` instances. This pattern translates low-level
execution events into standard `AgentEvent`s for the parent session.

### 5. Milestone 5: ADK Execution Engine

This is the milestone that replaces the legacy `GeminiClient` + `Turn` loop
with an ADK `Runner` driving an `LlmAgent`, behind the `AgentSession` interface
established in Milestones 1–3.

#### Principle: ADK is wrapped, not adopted

Milestone 5 does the smallest amount of work that makes the existing gemini-cli
services callable from inside an ADK `Runner`. Auth, model fallback, routing,
tool approval, policy enforcement, MCP, chat compression, and tool output
masking all stay on the gemini-cli side. ADK contributes the agent loop and the
event stream; gemini-cli contributes everything else.

No new abstractions are introduced when an existing service can be called. No
legacy code is deleted in Milestone 5 — that's a later milestone.

#### Reference Implementation

The starting point for the implementation is Alexey Kalenkevich's prototype at
[commit `eeb9301a`](https://github.com/google-gemini/gemini-cli/commit/eeb9301a489a1609f7d74e6c61569d82c5742821).
It contains a working ADK `Runner` shell with translator, model wrapper, and
plugin scaffolding, but predates several scope decisions in this doc (it still
uses ADK-native confirmation events, calls tools via `buildAndExecute` instead
of the legacy Scheduler, and does not implement auth/fallback/loop detection).
Milestone 5 incorporates its working parts and replaces the rest.

Files from the prototype that Milestone 5 will use as the base:

```
packages/core/src/agent/adk-agent/
├── adk-agent.ts             — GeminiCliAgent extends BaseAgent; LlmAgent + tools build
├── adk-agent-session.ts     — AdkAgentSession extends AgentSession; AdkAgentProtocol
├── adk-event-translator.ts  — ADK Event → AgentEvent[]
├── agent-run.ts             — AgentRunInvocation; status tracking; runs the Runner
├── model.ts                 — GeminiCliModel extends BaseLlm
├── plugins.ts               — MaxTurns / TokenLimit / MaxTime plugins (and loop detection)
└── type-utils.ts            — AgentSend payload guards
```

#### Feature Support Matrix

Parity with the legacy runtime is the target. Every feature below is supported
under the ADK runtime via the listed mechanism. None are deferred to a later
milestone unless explicitly stated.

| Feature | Mechanism under ADK |
| --- | --- |
| OAuth / ADC / Compute auth | `GeminiCliModel` injects auth into `request.config.httpOptions.headers` (`google_llm.ts:142–147` merges them) |
| Model fallback (429, terminal) | `GeminiCliModel` retry loop calls existing `handleFallback(config, currentModel, authType, err)` |
| Dynamic model routing | `GeminiCliModel` consults `ModelConfigService` per request; updates `currentModel` for next iteration |
| Model availability | Same path; `ModelAvailabilityService` mutated by fallback handler |
| Tool execution | ADK `BaseTool` shim delegates to legacy `Scheduler.executeToolCall(...)` rather than `tool.buildAndExecute(...)` |
| Tool approval / policy | Legacy `Scheduler` + `MessageBus` + `ToolConfirmationQueue.tsx` (unchanged) |
| Tool output masking | Existing `ToolOutputMaskingService` called from request pipeline |
| Chat compression | Existing `ChatCompressionService` projects outgoing requests (Phase 1 keeps it outside `BaseContextCompactor`) |
| Loop detection | `LoopDetectionAdkPlugin` (new) calls existing `LoopDetectionService`; on Strike-2, aborts the runner |
| Plan mode | `InstructionProvider` for prompt switching + mode-aware tool filtering keyed on `ApprovalMode` |
| User steering | `beforeModelCallback` reads `InjectionService` queue and prepends to the request |
| MCP tools | Existing `mcp-tool.ts` wrappers shaped as `BaseTool`s; ADK sees them through the same tool registry (prototype's direct `MCPToolset` approach is replaced) |
| Subagents | Milestone 4's `AgentTool` wraps subagent `AgentSession`s; surfaced to ADK as tools |
| Telemetry / Clearcut | Translator emits same telemetry hooks; explicit instrumentation on hot paths via plugins |
| Session persistence (resume) | Existing `ChatRecordingService` continues; replay reads from JSONL |
| Skills | Already shaped as tools; no special path |
| Slash commands | Stay in CLI layer; unchanged |
| ACP | Same `AgentSession` interface; ACP entry point picks runtime by flag |
| Rewind | Existing JSONL rewind continues |
| Max turns / max budget / max time | Existing `MaxTurnsAdkPlugin` / `TokenLimitAdkPlugin` / `MaxTimeAdkPlugin` from the prototype |

#### Explicit Non-Goals for Milestone 5

These are deferred to later milestones, not "unsupported under ADK":

- Native ADK confirmation/resumption — legacy `MessageBus` stays as the approval channel
- Native `BasePolicyEngine` adoption — legacy `Scheduler` enforces policy
- Native `BaseContextCompactor` adoption — existing service projects outgoing requests
- Native `BaseSessionService` persistence — existing JSONL chat recording continues
- Live/bidi runtime — blocked on upstream ADK `runLiveFlow` (currently throws `not implemented`)
- Mid-stream user interrupt — still queued-and-prepended at tool boundaries
- Native subagent decomposition into sub-`LlmAgent`s — subagents stay on legacy executor; surfaced via Milestone 4's `AgentTool`
- Direct ADK `MCPToolset` adoption — keep existing `mcp-tool.ts` wrappers for discovery/prompt/resource registry parity

#### Architecture — concrete interfaces

The Milestone 5 implementation lives in `packages/core/src/agent/adk-agent/`,
introduced by Alexey's prototype. The classes and their responsibilities:

**`AdkAgentProtocol implements AgentProtocol`** (in `adk-agent-session.ts`)

Replaces the legacy `Turn` loop. Owns the `AgentEvent` stream, the subscriber
fan-out, the per-turn `streamId` lifecycle, and the translation boundary. Holds
exactly one `GeminiCliAgent` and exactly one ADK `Session` per instance.

```ts
class AdkAgentProtocol implements AgentProtocol {
  send(payload: AgentSend): Promise<{streamId: string | null}>;
  subscribe(callback: (event: AgentEvent) => void): Unsubscribe;
  abort(): Promise<void>;
  readonly events: readonly AgentEvent[];
}
```

`send()` is the only async surface. It branches on payload kind (`message`,
`update`, `action`) — the `elicitations` branch is deleted in Milestone 5
because the legacy `MessageBus` handles approvals out-of-band.

**`GeminiCliAgent extends BaseAgent`** (in `adk-agent.ts`)

ADK's `BaseAgent.runAsyncImpl` yields events. This class builds the
`LlmAgent` per invocation (one TODO from the prototype: reuse across
invocations once mutation contract is clarified), wires the `GeminiCliModel`,
registers tools via `toAdkTool` shims, and yields the runner's event stream
upward.

The prototype's `toAdkTool` calls `tool.buildAndExecute(params, abortSignal)`
directly. **Milestone 5 changes this** to delegate to the legacy `Scheduler`:

```ts
function toAdkTool(
  tool: AnyDeclarativeTool,
  scheduler: CoreToolScheduler,
  abortSignal: AbortSignal,
): FunctionTool {
  return new FunctionTool({
    name: tool.name,
    description: tool.description,
    parameters: tool.getSchema().parametersJsonSchema as Schema,
    execute: async (params) => {
      // Route through legacy scheduler so approval/policy/UI/output-masking
      // all run unchanged.
      return scheduler.executeToolCall(tool, params, abortSignal);
    },
  });
}
```

`scheduler.executeToolCall` is the new entry point we add to the legacy
scheduler — a public version of what the legacy `Turn` loop calls internally.
It does approval, policy, masking, telemetry, and returns the `llmContent` +
`returnDisplay`. The ADK function-response shape is built from that result.

**`GeminiCliModel extends BaseLlm`** (in `model.ts`)

The prototype delegates to a wrapped `Gemini` client. Milestone 5 extends this
with:

```ts
async *generateContentAsync(
  request: LlmRequest,
  stream?: boolean,
  abortSignal?: AbortSignal,
): AsyncGenerator<LlmResponse, void> {
  while (true) {
    try {
      // 1. Resolve auth on each call (ADC tokens refresh)
      this.injectAuth(request);
      // 2. Routing — consult ModelConfigService for the resolved model
      this.applyRouting(request);
      // 3. Delegate to wrapped Gemini client
      yield* this.currentModel.generateContentAsync(request, stream, abortSignal);
      return;
    } catch (err) {
      // 4. Fallback — call existing handleFallback; loop retries on activate
      const decision = await handleFallback(
        this.config, this.currentModel.model, this.authType, err,
      );
      if (decision === null || decision === false) throw err;
      this.refreshFromConfig(); // currentModel getter pulls from Config.getModel()
    }
  }
}
```

The fallback loop and the auth injection are the two pieces of business logic
gemini-cli already owns. ADK does not need to know about them — the wrapped
model just appears as a normal `BaseLlm` that occasionally retries.

**`AdkEventTranslator`** (in `adk-event-translator.ts`)

Already exists in the prototype as `translateEvent`. The prototype handles
user content, agent text, thoughts, inline data, function calls, function
responses, and tool-confirmation requests. Milestone 5 changes:

- **Delete** the `event.actions.requestedToolConfirmations` branch (lines 158–185 of the prototype). Approvals are handled out-of-band by the legacy Scheduler; they never reach the translator.
- **Add** `usage` event emission from `event.usageMetadata`
- **Add** `error` event emission from `event.errorCode` / `event.errorMessage` with `_meta.code` for `LOOP_DETECTED`, `MAX_TURNS_EXCEEDED`, `TOKEN_LIMIT`, `MAX_TIME`
- **Add** partial-content coalescing keyed on `invocationId` — flush a single `message` event on transition to non-partial or end-of-turn, instead of one per token
- **Reuse** existing `mapUsage`, `mapError`, and `ensureStreamStart` helpers from `event-translator.ts` (extract to a shared module)

The lifecycle events (`agent_start`, `agent_end`) are emitted by the session
shell (`AdkAgentProtocol`), not the translator, matching the legacy split.

**`AgentRunInvocation`** (in `agent-run.ts`)

A status-tracked wrapper around `Runner.runAsync`. Keep as-is from the
prototype. Provides `AgentRunStatus.PENDING/RUNNING/COMPLETED/ABORTED/FAILED`,
a `Deferred<void>` for completion tracking, and error-code translation from
plugin errors to `AgentRunFailReason`. The abort path needs to be updated once
the ADK Runner supports `abortSignal` natively (currently the prototype works
around this by checking `abortSignal.aborted` in the model wrapper and the
agent's `runAsyncImpl`).

**Plugins** (in `plugins.ts`)

The prototype implements:

- `MaxTurnsAdkPlugin` — increment per `beforeModelCallback`; throw on exceed
- `TokenLimitAdkPlugin` — same shape, tracks usage
- `MaxTimeAdkPlugin` — same shape, tracks elapsed time

Milestone 5 adds:

- `LoopDetectionAdkPlugin` — in `onEventCallback`, filter `event.partial`, feed coalesced content + function calls to existing `LoopDetectionService.addAndCheck` (overload accepting ADK `Event`). On Strike-2, the plugin sets `lastEvent.errorCode = 'LOOP_DETECTED'` / `errorMessage = '...'` AND calls `invocationContext.abortSignal.abort()` (or equivalent). The translator turns the errorCode event into `error{_meta.code:'LOOP_DETECTED'}` and the session emits `agent_end{reason:'completed'}`.

`event.actions.escalate=true` is intentionally not used — it only stops
`LoopAgent`, not `LlmAgent`. The error+abort combination is the
`LlmAgent`-compatible stop.

**Composition root** (in `adk-agent-session.ts` or a new `adk-agent-service.ts`)

```ts
export function createAgentSession(config: Config, userId: string): AgentSession {
  if (config.getAdkRuntime()) {
    return new AdkAgentSession({ userId, config });
  }
  return new LegacyAgentSession({ config, userId });
}
```

The feature flag is `config.getAdkRuntime()` — defaults `false` until
Milestone 5 ships.

#### Execution Plan — Four Parallel Tracks

Each track is independently scopable, with a clean contract to the others.
Tracks A, B, C can start in parallel after Track A.1. Integration happens at
Track D.

**Track A — Protocol shell and translator parity** (Adam)

| # | What lands | Depends on |
| --- | --- | --- |
| A.1 | `adk-agent/` directory imported from prototype (minus the elicitation branch); compiles; no behavior changes elsewhere | — |
| A.2 | Translator scenario-parity test file mirroring `event-translator.test.ts` | A.1 |
| A.3 | Translator: add `usage`, `error`, partial-content coalescing; delete confirmation branch | A.1 |
| A.4 | `AdkAgentProtocol.send` payload handling: `message`, `update`, `action` (delete `elicitations`) | A.1 |
| A.5 | `agent_start` / `agent_end` lifecycle, `streamId` synthesis, subscriber fan-out, `events` array | A.4 |

**Track B — Model wrapper** (second engineer)

| # | What lands | Depends on |
| --- | --- | --- |
| B.1 | `GeminiCliModel` skeleton: extend `BaseLlm`, delegate, add `abortSignal` plumbing | A.1 |
| B.2 | Auth injection into `request.config.httpOptions.headers`; verify both unary and live paths | B.1 |
| B.3 | Fallback retry loop calling existing `handleFallback`; `currentModel` getter from `Config.getModel()` | B.1 |
| B.4 | Dynamic routing integration (consult `ModelConfigService` per request) | B.1 |
| B.5 | `connect()` stub throwing `UNIMPLEMENTED` for now (live runtime is deferred) | B.1 |

**Track C — Tool integration via legacy Scheduler** (second engineer)

| # | What lands | Depends on |
| --- | --- | --- |
| C.1 | `Scheduler.executeToolCall(tool, args, abortSignal)` public entry point — happy path only | — |
| C.2 | Approval flow through public entry point — listener-leak audit | C.1 |
| C.3 | `toAdkTool(tool, scheduler, abortSignal)` shim builds `FunctionTool` that calls the public entry point | C.1, A.1 |
| C.4 | MCP tool registration: keep existing `mcp-tool.ts` wrappers; ADK sees them through the registry (replace prototype's `MCPToolset` direct path) | A.1 |
| C.5 | Subagent tool registration via Milestone 4's `AgentTool` | A.1, M4 complete |

**Track D — Integration, plugins, and end-to-end** (Adam)

| # | What lands | Depends on |
| --- | --- | --- |
| D.1 | `LoopDetectionAdkPlugin` + `LoopDetectionService.addAndCheck(Event)` overload | A.1 |
| D.2 | `beforeModelCallback` for user injection (steering) | B.4 |
| D.3 | `InstructionProvider` + mode-aware tool filter for plan mode | A.5 |
| D.4 | **`AdkAgentService` selector + `config.getAdkRuntime()` feature flag** | A.5, B.5, C.4 |
| D.5 | **End-to-end non-interactive parity test under both runtimes** (fixture: text + thought + tool call + approval + fallback + Strike-2 loop) | D.4 |
| D.6 | Telemetry plumbing through translator + plugins | D.4 |

Critical-path lanes: A.1 unblocks everything. A.1 + B.1 + C.1 can all start
day one. Integration commitments are D.4 and D.5.

#### Engineer Scoping

Milestone 5 splits naturally into two halves with a clean interface between
them:

**Adam — External Interface (Tracks A + D, ~6 PRs):** Protocol shell,
translator, plugin registration, composition root, feature flag, end-to-end
integration. Continuous ownership with Milestones 1, 2, 4.

**Second engineer — Internal Execution (Tracks B + C, ~5–7 PRs):** Model
wrapper (auth, fallback, routing), tool shim via legacy Scheduler. Self-
contained vertical with a clean boundary (input: `LlmRequest` or function
call; output: `LlmResponse` or function response). Doesn't require deep
familiarity with Milestone 1–3 design decisions; concrete sub-PRs listed
above. Has clear demoable progress at each PR.

**What the second engineer scopes themselves:**

- Track B vs Track C ordering (model-first proves auth/fallback parity; tool-first proves execution parity)
- Whether `LoopDetectionAdkPlugin` goes in their half or Adam's
- Resolution of the **Scheduler re-entrancy risk** (see Risks below) — they own the design call

**Integration contract between the two halves** (agreed at A.1 / D.4):

- `GeminiCliAgent` constructor takes `{ config, scheduler, abortSignal }`
- `toAdkTool(tool, scheduler, abortSignal)` is the only place ADK sees the legacy Scheduler
- `GeminiCliModel` constructor takes `{ config, abortSignal }` and reads everything else from `config`
- `LoopDetectionAdkPlugin` constructor takes `{ loopDetector, onEscalate }` — `onEscalate` is supplied by `AdkAgentProtocol`

**If the second engineer is new to the codebase:** give them only Track C.
It's smaller, more mechanical, and has the cleanest interface boundary
(`Scheduler.executeToolCall` is the entire surface). Adam takes A + B + D.

#### Acceptance Criteria

- Non-interactive CLI under `config.getAdkRuntime()=true` produces an `AgentEvent` stream structurally equivalent to the legacy runtime on a representative fixture prompt covering: text, thought, tool call + approval, tool response, model fallback, Strike-2 loop detection
- All existing legacy-runtime tests pass with no modifications
- Translator parity test passes scenario-for-scenario against `event-translator.test.ts`
- New ADK-runtime tests for tool shim, fallback, loop detection, end-to-end all pass
- Feature flag default is `false`; flipping to `true` is the only opt-in mechanism
- No `tool.buildAndExecute` calls from the ADK path (Scheduler is the only execution entry)

#### Risks

- **Scheduler re-entrancy under ADK** — ADK may invoke tools in parallel within one turn. The legacy Scheduler assumes single-threaded turn execution. Resolve in Track C.1 before C.2 lands.
- **Tool ID correlation** — ADK's `functionCall.id` vs Scheduler's `callId`. The shim must establish a deterministic mapping or share an ID generator. Prototype's translator already keys on `functionCall.id`; verify ADK auto-generates one if missing.
- **Partial-event coalescing** — ADK emits per-token `partial: true` events. Default policy: flush on non-partial events only. Storms on long streams otherwise.
- **Loop-detection stop mechanism** — `event.actions.escalate=true` does NOT stop `LlmAgent`. Plugin must set `errorCode`/`errorMessage` AND abort the invocation. Verify abort propagation reaches the model wrapper.
- **MessageBus is process-global** — not a Phase 1 issue (no concurrent top-level sessions today), but document the constraint in `AdkAgentSession`.
- **Agent reuse across invocations** — prototype rebuilds `LlmAgent` per invocation. Reuse requires understanding ADK's agent-mutation contract (tool registry diffs, instruction provider changes, model swaps). Phase 1 keeps the prototype's rebuild-per-invocation; revisit if telemetry shows it as a hot path.
- **Abort propagation** — ADK Runner does not yet accept an `AbortSignal` in `runAsync` (per prototype TODO). Workaround: signal is checked inside `GeminiCliModel.generateContentAsync` and `GeminiCliAgent.runAsyncImpl`. Remove the workaround when upstream lands.
