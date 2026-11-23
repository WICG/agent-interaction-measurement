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
| Are tools understood and useful to the agent? | Optional tool assessments and document-local correlation of repeated calls. |
| Where is time spent across tools, DOM actuation, and user confirmations? | `agent-interaction` timing and steps. |
| Did the experience go well for the user? | Optional `agent-outcome` feedback and assessments. |

For example, a search tool may return successfully several times before the agent falls back to the UI. Correlating those actions can identify a workflow worth investigating; an optional assessment might explain whether the tool was usable. Neither repeated calls nor a fulfilled callback alone establishes success or failure.

## Proposal

Add three entry types to the [Performance Timeline](https://www.w3.org/TR/performance-timeline/), observed through `PerformanceObserver`. Asynchronous delivery and buffering let RUM scripts load late without instrumenting each tool's implementation.

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

The following is an API sketch, not a decision to expose every field to page script. Timing, attribution, assessments, and outcomes can be supported independently. Buffers are finite, and optional signals may be unavailable or withheld; receiving no entries is not evidence of failure or the absence of an agent.

### 1. `tool-call`

One invocation of a [WebMCP](https://webmachinelearning.github.io/webmcp/) tool, observed by the document that registered it. The design should also cover tools from the [declarative API exploration](https://github.com/webmachinelearning/webmcp/blob/main/declarative-api-explainer.md), whose completion hooks still need definition.

| Field | Meaning |
| --- | --- |
| `name` | Registered tool name. |
| `startTime` / `duration` | Dispatch through callback settlement or terminal cancellation, excluding any pre-dispatch UA confirmation. |
| `toolOrigin` | Origin that registered the tool. |
| `declarative` | Whether the tool was registered declaratively. |
| `caller` | `"browser-agent"`, `"extension"`, `"page"`, or `"unknown"`, based on browser-established provenance. |
| `status` | `"success"`, `"error"`, `"invalid-input"`, or `"cancelled"`. |
| `interactionId` | Opaque document-local interaction ID, or `null` when unavailable. |
| `callerOrigin` (optional) | Calling origin, only for `caller: "page"`, subject to disclosure policy. |
| `agentToolAssessment` (optional) | `"usable"`, `"problematic"`, or `"unknown"`. |

`success` means the callback fulfilled, not that its result was useful. `error` covers a throw or rejection without exposing exception details. Pre-execution validation failures (`invalid-input`) need a separate timing and exposure decision; this must not become a signal that an agent considered or safety-blocked a tool.

If a callback requests user input after dispatch, its wall-clock duration includes that wait. Interaction steps can identify it separately. Cancellation produces one terminal measurement, even if the callback continues running.

The original straw proposal also suggested optional `agentBrand` for browser agents and `model` identification. Their necessity and privacy cost remain open; neither is assumed to belong in a minimal page-visible API.

### 2. `agent-interaction`

One agent turn, whether or not it uses WebMCP. A turn starts when the agent begins acting on a user request and ends when it finishes, asks a new conversational question, or gives up. A temporary confirmation or elicitation does not split the turn.

| Field | Meaning |
| --- | --- |
| `name` | Empty string, never a task label or user request. |
| `interactionId` | Opaque identifier scoped to the receiving document. |
| `startTime` / `duration` | The document-local portion of the interaction. |
| `steps` | Bounded, coarsened array of `{ type, startTime, duration }`. |

Candidate step types are `"tool-call"`, `"dom-actuation"`, `"navigation"`, `"thinking"`, and `"awaiting-user"`. These describe activity categories, not reasoning content, DOM targets, or navigation destinations. Parallel activity can overlap, so step durations are not necessarily additive.

Calls within an interaction share its ID. IDs do not carry across documents or origins, and the receiving page must not learn about activity before the agent reached it or on other sites. Times use the document's `performance.timeOrigin`; precision and document-participation boundaries remain to be defined.

Entries would be emitted when the local interaction ends. Visibility-change snapshots are another possibility, but would need explicit partial/final semantics to prevent double-counting. A visibility change is not itself task completion or abandonment.

### 3. `agent-outcome`

Optional feedback associated with the receiving document's interactions, distinct from tool execution status.

| Field | Meaning |
| --- | --- |
| `name` | Empty string. |
| `startTime` / `duration` | When the outcome becomes available to a live document; duration is `0`. |
| `interactionIds` | Bounded list of relevant IDs from that document. |
| `userFeedback` | `"positive"`, `"negative"`, `"none"`, or `"unknown"`. |
| `agentTaskAssessment` | `"completed"`, `"abandoned"`, `"failed"`, or `"unknown"`. |

User feedback must be explicit, not inferred from conversation text. `none` means no feedback was supplied through an available channel; `unknown` means it cannot be determined or disclosed. Agent assessments are opinions, not browser-established facts. Missing feedback must not be counted as success.

A task-wide rating is not automatically a rating of each participating site. Outcomes should be withheld when they cannot be scoped without disclosing private intent or off-site activity.

Feedback may arrive after navigation or page closure. We should therefore explore a [Reporting API](https://www.w3.org/TR/reporting/) option using an endpoint configured by the receiving origin. Whether outcomes should be observer entries, reporting-only, or aggregate-only remains open. Report format, consent, retention, and deletion rules would require new specification work.

## Information needed from agents

- **Execution:** Browser WebMCP hooks can supply dispatch, completion, cancellation, and known caller provenance.
- **Interactions:** Cooperating agents must supply turn boundaries and processing/waiting states, integrated with browser actuation attribution.
- **Assessments and feedback:** Only the agent or its feedback UI can supply these; they must be optional and safely scoped.

Browser-integrated agents could use internal interfaces. Extension agents need a browser-mediated bridge; injected page script alone may be indistinguishable from other page script. In-page agents already have author instrumentation, but a standard assessment channel remains an open question. We should not infer agent identity from behavioral fingerprints.

## Privacy, security, and accessibility

- **Minimize disclosure.** Do not expose prompts, conversation history, reasoning, user goals, tool arguments/results, exception text, or cross-site task IDs. Even enum-valued feedback can be sensitive when linked to a visit.
- **Preserve boundaries.** Begin with secure-context documents observing their own activity, without automatic frame aggregation. Cross-origin tool access through `exposedTo` is not permission to inspect another document's timeline. Page-visible data is also available to third-party scripts running with that page's privileges.
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
2. What timing precision, buffer limits, document boundaries, and lifecycle rules preserve utility without exposing off-site activity? Are partial snapshots necessary?
3. How should validation, declarative execution, navigation-triggering tools, and cancellation map to completion measurements?
4. How should agent turns relate to [Event Timing's `interactionId`](https://www.w3.org/TR/event-timing/#dom-performanceeventtiming-interactionid), UI updates, and human interventions? Would an `agentInvoked` flag justify its added exposure?
5. Which lifecycle and assessment signals can agent providers supply reliably? What extension or in-page integration is needed?
6. Is tool-discovery measurement necessary, and can it avoid exposing tool selection or safety decisions?

The next step is to review these use cases and signals with agent providers, RUM developers, and privacy reviewers, coordinating scope with the WebML CG and Web Performance WG.

## Background

This develops the [straw proposal in WebMCP issue #186](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-6369314489) and the [12 November 2026 WebML CG resolution](https://www.w3.org/2026/11/12-webmachinelearning-minutes.html#cad0) requesting an Agent Interaction Measurement explainer. The discussion includes [demand for RUM without tool wrappers](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-5510686305), [user-control concerns](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-4929491673), and [fingerprinting concerns](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-6413798793).

Thanks to participants in that discussion and the [related observability work](https://github.com/w3c/tpac2026-breakouts/issues/63). Their participation does not imply endorsement; no browser implementation commitment is claimed.
