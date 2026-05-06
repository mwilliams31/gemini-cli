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

This section details the design of the concrete ADK agent that will replace the
legacy runtime.

#### Supported Features

_(To be mapped out later. Examples: VSCode MCP Server integration, chat
compression, etc.)_

#### Unsupported Features

_(To be mapped out later.)_

#### Implementation Constraints & Rationale

- **Elicitation Bypass**: We are explicitly not supporting ADK's native
  `elicitation_request`/`response` for tool approvals. We reuse the existing
  message bus and scheduler callbacks to minimize UI churn.
- **Deferred Security Plugin**: We are not implementing an ADK-native
  `BaseSecurityPlugin` at this time. Security policy enforcement remains in the
  legacy `Scheduler`.

#### Mapping onto `adk-ts`

- Gemini CLI `AgentSend` objects map to standard GenAI content formats expected
  by ADK.
- ADK `Event` streams translate back to Gemini CLI `AgentEvent`s in the protocol
  layer.

#### High-Level Pseudo Code

```ts
// Sketch of the AdkDefaultAgent class
class AdkDefaultAgent implements AgentSession {
  // ... initialization with resolved tools from Config
  // ... run loop translating ADK events to AgentEvents
}
```
