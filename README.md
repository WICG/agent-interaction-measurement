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

## Privacy, security, and accessibility

- **Minimize disclosure.** Do not expose prompts, conversation history, reasoning, user goals, tool arguments/results, exception text, or cross-site task IDs. Even enum-valued feedback can be sensitive when linked to a visit.
- **Preserve boundaries.** Begin with secure-context documents observing their own activity, without automatic frame aggregation. Cross-origin tool access through `exposedTo` is not permission to inspect another origin's timeline. Page-visible data is also available to third-party scripts running with that page's privileges.
- **Respect user choice.** Feedback given to an agent is not automatically intended for a website. Newly exposed agent data needs an appropriate user-control and consent model; configuring an endpoint is not consent.
- **Limit fingerprinting.** Caller categories reveal agent use; model identifiers, step timings, and assessments add probing opportunities. Coarsening, sampling, rate limits, and omitting unnecessary fields need evaluation. Discovery, skipped tools, and safety-block reasons are not part of this sketch.
- **Do not equate reporting with anonymity.** Delayed, credential-less reports can still be linked through URLs, IDs, or network metadata. Restricted or aggregate reporting needs an explicit privacy design.
- **Keep telemetry separate from authority.** Measurements do not authenticate agents or prove authorization, correctness, or user consent. Bound buffers, step lists, and retained reports; define expiry and deletion on permission revocation or browsing-data clearing.
- **Support human oversight.** Confirmations, corrections, and assistive-technology use are not failures. Synthesized input alone must not identify an agent. Metrics do not prove a UI change was perceivable; any consent or feedback UI must be accessible and localizable.

## Alternatives

- **Author instrumentation / User Timing:** Useful today, but requires integration and lacks agent-side context or browser-established provenance.
- **Provider dashboards or feedback tools:** Can provide richer data, but require per-provider arrangements or changes to tool interfaces, with similar feedback-privacy concerns.
- **A separate observer or DOM events:** PerformanceObserver already supplies asynchronous, buffered measurement. Functional `toolactivated` and `toolcancel` events should remain where needed; measurement must not change execution behavior.
- **DevTools or reporting-only diagnostics:** May be better homes for sensitive detail. DevTools alone does not provide field measurement; reporting must balance diagnostic utility against linkability.

## Open questions

1. What is the smallest useful page-visible surface? Which signals belong only in restricted reports, aggregates, or DevTools, and what user controls are needed?
2. How should timing and entry delivery work across same-origin navigations? What precision, buffer limits, and partial-entry rules are appropriate?
3. How should validation, declarative execution, navigation-triggering tools, and cancellation map to completion measurements?
4. How should agent turns relate to [Event Timing's `interactionId`](https://www.w3.org/TR/event-timing/#dom-performanceeventtiming-interactionid), UI updates, and human interventions? Would an `agentInvoked` flag justify its added exposure?
5. Which lifecycle and assessment signals can agent providers supply reliably? What extension or in-page integration is needed?
6. Is tool-discovery measurement necessary, and can it avoid exposing tool selection or safety decisions?

The next step is to review these use cases and signals with agent providers, RUM developers, and privacy reviewers, coordinating scope with the WebML CG and Web Performance WG.

## Background

This develops the [straw proposal in WebMCP issue #186](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-6369314489) and the [12 November 2026 WebML CG resolution](https://www.w3.org/2026/11/12-webmachinelearning-minutes.html#cad0) requesting an Agent Interaction Measurement explainer. The discussion includes [demand for RUM without tool wrappers](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-5510686305), [user-control concerns](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-4929491673), and [fingerprinting concerns](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-6413798793).

Thanks to participants in that discussion and the [related observability work](https://github.com/w3c/tpac2026-breakouts/issues/63). Their participation does not imply endorsement; no browser implementation commitment is claimed.
