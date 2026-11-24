# Agent Interaction Measurement

**Author:** [Yoav Weiss](https://github.com/yoavweiss)

**Status:** Personal draft for proposed WICG incubation. The API and exposure policy are provisional; this is not an implemented feature.

**Discuss:** [Issues](https://github.com/yoavweiss/agent-interaction-measurement/issues) · [Pull requests](https://github.com/yoavweiss/agent-interaction-measurement/pulls)

## Problem

Agents combine tool calls, DOM actuation, and navigation to accomplish tasks on websites. Sites can instrument their own code, but cannot reliably tell who invoked it, where the agent spent its time, or whether the result was useful. A fast, successful tool callback does not necessarily mean a fast, successful user experience.

Developers and RUM providers need a common way to measure these experiences without wrapping every tool or integrating separately with every agent provider. This would help them improve reliability and latency without accessing the user's conversation.

## Use cases

| Question | Proposed signal |
| --- | --- |
| Are my tools used, fast, and completing successfully? | `tool-call` timing and status. |
| Are calls coming from an agent or page script? | Coarse caller attribution. |
| Are tools understood and useful to the agent? | Optional tool assessments and same-origin correlation of repeated calls. |
| Where is time spent across tools, DOM actuation, and user confirmations? | `agent-interaction` timing and steps. |
| Did the experience go well for the user? | Optional `agent-outcome` feedback and assessments. |

## Proposal

Add three new [PerformanceEntry](https://www.w3.org/TR/performance-timeline/#the-performanceentry-interface) types, that cover the above use cases.


```js
if ("PerformanceObserver" in globalThis) {
  const observer = new PerformanceObserver(list => {
    for (const entry of list.getEntries()) {
      // Aggregate selected fields according to entry.entryType.
    }
  });

  for (const type of ["tool-call", "agent-interaction", "agent-outcome"]) {
    if (PerformanceObserver.supportedEntryTypes.includes(type)) {
      observer.observe({ type, buffered: true });
    }
  }
}
```

### 1. `tool-call`

One invocation of a [WebMCP](https://webmachinelearning.github.io/webmcp/) tool, observed by the document that registered it. The design should also cover tools from the [declarative API exploration](https://github.com/webmachinelearning/webmcp/blob/main/declarative-api-explainer.md), whose completion hooks still need definition.

| Field | Meaning |
| --- | --- |
| `name` | Registered tool name. |
| `startTime` | A timestamp indicating the initial tool callback. |
| `duration` | Millisecond duration of the callback's run time. |
| `toolOrigin` | Origin that registered the tool. |
| `declarative` | Whether the tool was registered declaratively. |
| `caller` | `"browser-agent"`, `"extension"`, `"page"`, or `"unknown"`, based on browser-established provenance. |
| `status` | `"success"`, `"error"`, `"invalid-input"`, or `"cancelled"`. |
| `interactionId` | Opaque origin-scoped interaction ID, or `null` when unavailable. |
| `callerOrigin` (optional) | Calling origin, only for `caller: "page"`. |
| `agentBrand` (optional) | Agent brand, only for `caller: "browser-agent"`. |
| `model` (optional) | Model used by the agent. |
| `agentToolAssessment` (optional) | `"usable"`, `"problematic"`, or `"unknown"`. |

`success` means the callback fulfilled, not necessarily that its result was useful. `error` covers a throw or rejection without exposing exception details. `invalid-input` means an attempted invocation failed input validation before the callback ran.

If a callback requests user input after dispatch, its wall-clock duration includes that wait. Interaction steps can identify it separately. Cancellation produces one terminal measurement, even if the callback continues running.

### 2. `agent-interaction`

This entry measures the time it takes for an agent to act on a user request, regardless of whether that request involved using WebMCP.
An interaction starts when the agent begins acting on a user request and ends when it finishes, asks a new conversational question, or gives up. A temporary confirmation or elicitation does not split the turn.

| Field | Meaning |
| --- | --- |
| `name` | Empty string. |
| `interactionId` | Opaque identifier scoped to the receiving origin. |
| `startTime` | A timestamp indicating the start of the interaction. |
| `duration` | Millisecond duration of the interaction. |
| `steps` | Bounded, coarsened array of `{ type, startTime, duration }`. |

Candidate step types are `"tool-call"`, `"dom-actuation"`, `"navigation"`, `"thinking"`, and `"awaiting-user"`. Parallel activity can overlap, so step durations are not necessarily additive.

Calls within the same interaction share an origin-scoped ID that persists across same-origin navigations. New interactions get new IDs, and different origins receive different IDs. Timestamps remain relative to each document's `performance.timeOrigin`; their cross-document representation and precision remain open questions.

Entries would be emitted when the interaction ends. Navigation or visibility-change snapshots could expose partial progress, but would need explicit partial/final semantics to prevent double-counting. Neither a same-origin navigation nor a visibility change ends the interaction by itself.

### 3. `agent-outcome`

Optional feedback associated with the receiving origin's interactions, distinct from tool execution status.

| Field | Meaning |
| --- | --- |
| `name` | Empty string. |
| `startTime` | A timestamp indicating the end of the latest interaction relevant to the outcome. |
| `duration` | `0`. |
| `interactionIds` | Bounded list of relevant IDs from that origin, including earlier same-origin documents. |
| `userFeedback` | `"positive"`, `"negative"`, `"none"`, or `"unknown"`. |
| `agentTaskAssessment` | `"completed"`, `"completed with difficulties"`, `"abandoned"`, `"failed"`, or `"unknown"`. |

Exposing the user feedback is optional and depends on user consent for sharing that data.
The agent's task assessment should be taken as a signal from the agent.

Given that feedback may arrive after navigation or page closure, we should explore a [Reporting API](https://www.w3.org/TR/reporting/) option/alternative to a `PerformanceEntry` for this.

## Information needed from agents

- **Execution:** Browser WebMCP hooks can supply dispatch, completion, cancellation, and known caller provenance.
- **Interactions:** Cooperating agents must supply turn boundaries and processing/waiting states, integrated with browser actuation attribution.
- **Assessments and feedback:** Only the agent or its feedback UI can supply these; they must be optional and safely scoped.

Browser-integrated agents could use internal interfaces. Extension agents need a browser-mediated bridge; injected page script alone may be indistinguishable from other page script. In-page agents may need a web-exposed API to provide interaction data. Local agents external to the browser may need a WebDriver BiDi API that enables them to report this information.

## Privacy and security

The performance entries described in this explainer expose new information.
Details about WebMCP tools callers, their origin, the agent and model and the agent's assessment of the tools are all novel information.
Similarly, the data exposed by agent-interaction and agent-outcome is novel as well.

While this data is required for site operators to ship WebMCP tools at scale and improve them over time, it's data for [ancillary use](https://www.w3.org/TR/privacy-principles/#dfn-ancillary-use).
We should be careful about that data and make sure that it doesn't expose any unnecessary information about the user, and their goals. On top of that, we should make sure that no cross-site information is exposed.

Information derived from the user interactions with the agent (e.g. `userFeedback`) should be subject to user consent.

## Considered alternatives

### Backend communication with agent providers
The agent-bound information could also be reached through proprieary dashboards provided by the agent-providers.
That would require each site to establish a relationship with every agent provider in order to keep track of their agentic experiences. That seems suboptimal.

## Open questions

1. How should agent turns relate to [Event Timing's `interactionId`](https://www.w3.org/TR/event-timing/#dom-performanceeventtiming-interactionid), UI updates, and human interventions? Should agentic interactions count for the purposes of [INP](https://developer.mozilla.org/en-US/docs/Glossary/Interaction_to_next_paint)?
2. What APIs should we provide to agent implementations (in-page or external to the browser) to enable them to provide the necessary data?
3. How can we safely represent agentic interactions that span origins and sites? How can we define the interaction start and end at the origin's boundaries?

## Background

This develops the [straw proposal in WebMCP issue #186](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-6369314489) and the [12 November 2026 WebML CG resolution](https://www.w3.org/2026/11/12-webmachinelearning-minutes.html#cad0) requesting an Agent Interaction Measurement explainer. The discussion includes [demand for RUM without tool wrappers](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-5510686305), [user-control concerns](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-4929491673), and [fingerprinting concerns](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-6413798793).

Thanks to participants in that discussion and the [related observability work](https://github.com/w3c/tpac2026-breakouts/issues/63).
