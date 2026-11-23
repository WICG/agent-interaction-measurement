# Agent Interaction Measurement

**Author:** [Yoav Weiss](https://github.com/yoavweiss)

**Status:** Personal draft for discussion and proposed WICG incubation. This is not an adopted WICG incubation, a W3C standard, or an implemented API. All API names, fields, exposure rules, and delivery mechanisms below are provisional.

**Participate:** [Issues](https://github.com/yoavweiss/agent-interaction-measurement/issues) · [Pull requests](https://github.com/yoavweiss/agent-interaction-measurement/pulls)

This explainer develops the [PerformanceObserver-based straw proposal in WebMCP issue #186](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-6369314489). It also incorporates questions raised in the subsequent discussion and the [12 November 2026 Web Machine Learning CG resolution](https://www.w3.org/2026/11/12-webmachinelearning-minutes.html#cad0) to document use cases, a rough API sketch, and the information needed from agents, then review these with agent providers and privacy reviewers.

## Contents

- [Introduction](#introduction)
- [User-facing problem and use cases](#user-facing-problem-and-use-cases)
- [Goals and non-goals](#goals-and-non-goals)
- [Proposed approach](#proposed-approach)
- [Tool calls](#tool-calls)
- [Agent interactions](#agent-interactions)
- [Agent outcomes](#agent-outcomes)
- [Information needed from agents](#information-needed-from-agents)
- [Delivery, buffering, and document lifecycle](#delivery-buffering-and-document-lifecycle)
- [Security and privacy considerations](#security-and-privacy-considerations)
- [Accessibility and internationalization](#accessibility-and-internationalization)
- [Alternatives considered](#alternatives-considered)
- [Dependencies and related work](#dependencies-and-related-work)
- [Stakeholder feedback and evidence](#stakeholder-feedback-and-evidence)
- [Open questions and next steps](#open-questions-and-next-steps)
- [References and acknowledgements](#references-and-acknowledgements)

## Introduction

Users increasingly ask agents to accomplish tasks on websites. An agent may call a WebMCP tool, operate the DOM, navigate, wait for the user, or combine all of those within one request. A site can often measure the code it runs, but cannot tell whether a slow or unsuccessful experience came from its tools, the agent's use of its interface, or time spent elsewhere. A tool invocation that returns successfully also does not establish that the user got what they wanted.

We propose exploring a common real-user measurement surface built on the [Performance Timeline](https://www.w3.org/TR/performance-timeline/), with separate `tool-call`, `agent-interaction`, and `agent-outcome` entry types. The first measures individual WebMCP invocations; the others address agent-mediated experiences more generally. Some information could come from browser execution hooks, while other information requires cooperation from the agent. The design goal is actionable measurement without exposing prompts, conversation content, tool arguments or results, or a cross-site record of the user's task. Achieving that goal requires more than restricting fields to enums: the exposure and reporting model remains a central design question.

## User-facing problem and use cases

The end-user benefit is websites that work reliably and quickly with the agent the user chooses, without requiring the site to integrate separately with every agent provider or inspect the user's conversation.

### 1. Make tools faster and more reliable

A user asks an agent to find available appointments. The site's availability tool is repeatedly slow or throws. The site and its RUM provider need invocation counts, execution latency, and completion status, including for declarative tools if those become available. Today, this generally requires wrapping each tool implementation. A RUM library loaded after tool registration may not be able to do that reliably.

### 2. Identify retries and fallback paths

An agent successfully calls a product-search tool several times, then falls back to operating the search UI. The callbacks all fulfilled, but the tool may have been difficult to understand or its results unusable. Document-local correlation can reveal repeated calls within one interaction and, where safely available, a transition from tool use to DOM actuation. It must not reveal the user's query, the tool's arguments, or activity on other sites.

### 3. Understand perceived latency, not just callback duration

A tool takes 200 ms, but the user waits several seconds for the agent to finish the turn. Developers need to distinguish tool execution, DOM actuation, navigation, agent processing, and waiting for the user. This helps identify whether changing a site's tools or UI can improve the experience. Waiting for a confirmation is not itself evidence of a performance defect.

### 4. Separate execution success from user success

A checkout tool can fulfill its promise even when the order is not what the user intended. Likewise, an agent can complete a task through DOM actuation without using any tools. An optional, carefully scoped outcome signal could help evaluate whether an intervention improved the experience rather than merely increased the number of successful callbacks.

Explicit user feedback, an agent's assessment, and site-observed outcomes such as a cancellation are different signals. None should be relabeled as another. In particular, the absence of negative feedback does not mean success, and a task-wide rating is not automatically a rating of a particular site or tool.

### 5. Understand how people and agents collaborate

A user may intervene, correct an action, undo a change, or answer a confirmation. The [WebML CG discussion](https://www.w3.org/2026/11/12-webmachinelearning-minutes.html#cad0) also raised whether agent-caused UI changes were perceivable, including through accessibility announcements. These are useful questions for research and requirements gathering; this draft does not yet define additional fields for them. We should avoid turning human oversight or assistive-technology use into a negative quality score.

### Mapping questions to candidate signals

| Developer question | Candidate signal | Important limit |
| --- | --- | --- |
| Are tools used, fast, and completing? | `tool-call` timing and status | Measures execution, not whether the answer was useful. |
| Is a call from page script or an agent? | Coarse `caller` category | Only when the browser can establish provenance. |
| Are tools being retried within a turn? | Document-scoped `interactionId` | A retry alone does not prove misunderstanding. |
| Did the agent find a tool useful? | Optional `agentToolAssessment` | Agent-provided assessment, with additional privacy risk. |
| Where is interaction time spent? | `agent-interaction` and coarse steps | Only the portion attributable to the receiving document. |
| Did the experience go well? | Optional `agent-outcome` | Not proof of user intent or a site's causal contribution. |
| Were tools never discovered, or seen and not chosen? | No signal in this initial sketch | Discovery and rejection diagnostics need separate evaluation. |

## Goals and non-goals

### Goals

- Let first-party developers and RUM libraries observe tool execution without modifying each tool's JavaScript.
- Cover agent-mediated experiences that mix WebMCP and other forms of actuation.
- Distinguish execution results, agent assessments, and explicit user feedback.
- Support asynchronous observation and bounded buffering so measurement code can load late.
- Identify the minimum information needed from agents, without requiring bilateral analytics integrations.
- Respect document and origin boundaries and the user's choice about sharing agent-related information.
- Permit partial support: execution timing should not depend on outcomes being available.

### Non-goals

- Expose prompts, conversation history, chain of thought, tool inputs or outputs, DOM targets, screenshots, or the user's goal.
- Create stable user, agent-installation, conversation, or cross-site task identifiers.
- Authenticate agents, grant permission to act, attest that an action was authorized, or provide fraud detection.
- Produce a universal agent leaderboard or a single agent-experience score.
- Guarantee observation of every agent, including agents whose actions the browser cannot attribute.
- Replace existing loading, responsiveness, accessibility, business-outcome, or application instrumentation.
- Change tool execution, confirmation, cancellation, or event trust semantics.

## Proposed approach

Use the existing `PerformanceObserver` mechanism rather than introduce a separate observer API. The conceptual split is:

| Entry type | Unit of observation | Source of information |
| --- | --- | --- |
| `tool-call` | One WebMCP invocation | Browser tool-dispatch hooks; optional agent assessment. |
| `agent-interaction` | A document's participation in one agent turn | Browser actuation hooks and cooperating agent lifecycle signals. |
| `agent-outcome` | An optional assessment associated with local interactions | Explicit user feedback and/or a cooperating agent's assessment. |

This is a requirements and API sketch, not a commitment to expose every field to page script. The initial candidate for page-visible measurement is basic execution information. Caller classification still reveals agent use. Step breakdowns, assessments, brand/model identifiers, and feedback require additional review; some may belong only in DevTools or privacy-preserving reporting, or may not be exposed at all.

### Observing entries

The following code illustrates the API shape. The `record*` functions stand for application-defined local aggregation, not automatic transmission. Collection and any subsequent reporting must follow the applicable user choices and site policy.

```js
if ("PerformanceObserver" in globalThis) {
  const supported = new Set(PerformanceObserver.supportedEntryTypes);
  const observer = new PerformanceObserver(list => {
    for (const entry of list.getEntries()) {
      switch (entry.entryType) {
        case "tool-call":
          recordToolLatency(entry.name, entry.duration, entry.status);
          break;
        case "agent-interaction":
          recordInteractionLatency(entry.duration);
          break;
        case "agent-outcome":
          recordOutcomeCounts(entry.userFeedback, entry.agentTaskAssessment);
          break;
      }
    }
  });

  for (const type of ["tool-call", "agent-interaction", "agent-outcome"]) {
    if (supported.has(type)) {
      observer.observe({ type, buffered: true });
    }
  }
}
```

`buffered: true` requests retained entries of that type; it cannot recover entries that were never generated, were withheld, or were evicted. Support for an entry type does not establish that an agent is installed or active. A browser could support `tool-call` without supporting either of the other types. Applications must work when they receive no entries.

The tables below describe candidate additions to `PerformanceEntry`. Times are relative to the receiving document's `performance.timeOrigin`. Agent-provided times must be converted into that clock domain by trusted browser plumbing, not exposed as arbitrary remote timestamps.

## Tool calls

A `tool-call` entry describes one invocation of a WebMCP tool registered in the receiving document. Both imperative tools and a potential future declarative API should be representable.

| Field | Proposed meaning |
| --- | --- |
| `entryType` | `"tool-call"`. |
| `name` | The registered tool name, not its arguments or a generated description. |
| `startTime` | For an executed call, when the UA dispatches execution, after any pre-dispatch UA confirmation. |
| `duration` | Wall-clock time from dispatch until the execution callback settles, throws, or the invocation is cancelled. |
| `toolOrigin` | The origin of the document that registered the tool. |
| `declarative` | Whether the invocation used a declaratively registered tool. |
| `caller` | `"browser-agent"`, `"extension"`, `"page"`, or `"unknown"`, based on browser-established provenance. |
| `status` | `"success"`, `"error"`, `"invalid-input"`, or `"cancelled"`; details below. |
| `interactionId` | An opaque document-local interaction identifier, or `null` if no eligible interaction can be attributed. |

### Completion and timing semantics

`success` means the execution callback fulfilled; it does not mean the result was consumed, the tool accomplished its advertised effect, or the user was satisfied. `error` covers a throw or rejection, without exposing exception messages or stacks. `cancelled` records the browser's terminal cancellation of the invocation, not subsequent work a callback might continue despite cancellation. Cancellation must not generate a second entry if that callback later settles.

`invalid-input` is a candidate for UA-detected input validation failures. Some such failures occur before the tool's callback runs. The dispatch boundary, whether these attempts should produce entries at all, and whether they belong in script or reporting remain open. It must not become a general signal that an agent considered, rejected, or safety-blocked a tool.

Time spent in a UA confirmation before dispatch is outside the tool-call duration, but could be represented as `awaiting-user` in the interaction. If a tool requests user input after its callback has started, its promise can remain pending during the wait. The wall-clock duration includes that wait; an interaction step may identify it separately. We should not silently subtract waiting time or treat overlapping spans as additive.

Synchronous throws, asynchronous rejection, cancellation races, and navigation-triggering tools need well-defined terminal hooks. Declarative tools do not necessarily have an `execute` promise; their completion boundary must be coordinated with that API rather than invented here.

### Additional candidate fields

These fields were suggested in the original straw proposal, but are not assumed to be part of a minimally exposed page API:

| Field | Candidate meaning | Issue to resolve |
| --- | --- | --- |
| `callerOrigin` | The invoking document's origin, only for `caller: "page"`; otherwise absent or redacted. | Tool-execution permission does not automatically imply permission to disclose caller identity or timing. |
| `agentBrand` | Optional coarse brand for a browser agent. | A browser's UA string does not necessarily identify its agent; extra entropy and discrimination risks need justification. |
| `model` | Optional model identification. | Potentially high entropy and sensitive; model names or versions should not be presumed safe to expose. |
| `agentToolAssessment` | `"usable"`, `"problematic"`, or `"unknown"`. | Requires agent cooperation and a definition of usefulness; sites could probe tools to fingerprint agent behavior. |

A page-script caller remains `page` even if it describes itself as an agent. Extension-originated activity can be classified as `extension` only when the browser has an appropriate trusted execution path; an extension injecting ordinary page script may be indistinguishable from other script. This proposal does not expose extension IDs or enumerate installed agents.

### Relationship to execution events

Performance entries provide asynchronous, potentially buffered measurement. WebMCP's `toolactivated` and `toolcancel` events can also support functional UI or cancellation-related behavior. We should investigate whether measurement-only uses can migrate to the timeline, but should not remove or replace functional events merely because entries exist. An observer is not an execution-control mechanism.

## Agent interactions

An interaction is one agent turn: it starts when an agent begins acting on a user request and ends when the agent finishes, asks the user a new conversational question, or gives up. A subsequent user message starts a new interaction. A temporary confirmation or elicitation within an ongoing action pauses progress but does not, by itself, start another interaction.

An agent turn can touch multiple documents. A document receives only a view of its own participation, not the turn's history before the agent reached it, time spent on other sites, or activity after the agent left it. Defining that clipping boundary is necessary before exposing durations or step sequences. This sketch does not expose a global turn duration to every participating page.

| Field | Proposed meaning |
| --- | --- |
| `entryType` | `"agent-interaction"`. |
| `name` | The empty string; never the user's request or an agent-generated task label. |
| `interactionId` | An opaque identifier scoped to the receiving document. |
| `startTime` / `duration` | Start and elapsed time of the eligible document-local portion of the interaction. |
| `steps` | A bounded, coarsened array of `{ type, startTime, duration }` for eligible local activity. |

Candidate step types are `"tool-call"`, `"dom-actuation"`, `"navigation"`, `"thinking"`, and `"awaiting-user"`. `thinking` would mean only an interval of agent processing or waiting on inference attributable to this document; it would expose no reasoning content. DOM actuation would not identify nodes, text, event payloads, or selectors. Navigation would not disclose destination URLs or timing from another document.

An illustrative entry, assuming the step types have been approved for exposure:

```js
({
  entryType: "agent-interaction",
  name: "",
  interactionId: "local-interaction-7",
  startTime: 1000,
  duration: 4200,
  steps: [
    { type: "tool-call", startTime: 1000, duration: 400 },
    { type: "thinking", startTime: 1400, duration: 600 },
    { type: "awaiting-user", startTime: 2000, duration: 2000 },
    { type: "dom-actuation", startTime: 4000, duration: 1200 }
  ]
})
```

These numbers are illustrative, not a proposed precision requirement. Step boundaries and interaction boundaries need consistent coarsening. Parallel calls and user-input requests made by pending tools can produce overlapping spans; consumers must not sum step durations and call that total interaction latency. Nor should they interpret all non-tool time as avoidable agent overhead.

### Correlation without a cross-site task ID

Tool calls in the same eligible interaction share its document-local `interactionId`. This permits analysis of repeated calls and mixed actuation within that document. The identifier's representation is not settled; examples use strings and consumers should treat them as opaque.

Identifiers are not reused across documents, including across same-origin navigations. Two origins participating in the same task must not receive a shared identifier. An absent or redacted ID is not proof that a call was made by a human.

We should investigate the relationship with [Event Timing's `interactionId`](https://www.w3.org/TR/event-timing/#dom-performanceeventtiming-interactionid), but not equate them: one agent turn may contain several UI interactions and tool calls. Similarly, an `agentInvoked` indication on Event Timing entries might help distinguish browser-attributed agent actuation from human input, but would reveal agent use even on sites without WebMCP. It is a related, privacy-sensitive design question, not a prerequisite for all three entry types.

## Agent outcomes

An `agent-outcome` represents optional feedback about eligible local interactions. It is not a transcript summary, an explanation of the user's goal, or a guaranteed measurement of correctness.

| Field | Proposed meaning |
| --- | --- |
| `entryType` | `"agent-outcome"`, if exposed through PerformanceObserver. |
| `name` | The empty string. |
| `startTime` / `duration` | When the outcome becomes available to a live document, with duration `0`; not a retroactive disclosure of a cross-site task's duration. |
| `interactionIds` | A bounded array of relevant interaction IDs from the receiving document only. |
| `userFeedback` | `"positive"`, `"negative"`, `"none"`, or `"unknown"`. |
| `agentTaskAssessment` | `"completed"`, `"abandoned"`, `"failed"`, or `"unknown"`. |

`userFeedback` describes explicit feedback, not sentiment inferred from conversation text. `none` means an eligible feedback channel was available but supplied no feedback for the reported scope; `unknown` means it cannot be determined or disclosed. Withholding an outcome entirely must also be allowed. The exact distinction and release policy need privacy review so they do not expose additional user activity.

`agentTaskAssessment` is the agent's assessment, not a fact established by the browser. An agent may lack enough context or confidence to provide anything other than `unknown`. Neither callback completion nor navigation away should automatically become a positive or negative outcome.

A task can span several turns, pages, or origins. Only the receiving document's relevant IDs and safely attributable assessment may be disclosed. A global thumbs-up/down or final success on another site must not simply be copied to every page that participated. If an outcome cannot be meaningfully scoped without revealing off-site activity or the user's private intent, it should be withheld. Determining when useful local attribution is possible is an open research question.

### Reporting after the document is gone

A user may provide feedback after navigating away or closing a page, so a live-document observer cannot cover all outcomes. We should explore a [Reporting API](https://www.w3.org/TR/reporting/) delivery option to an endpoint configured by the receiving origin while the document was active.

The exact report type, endpoint group, retention rules, consent requirements, and body are not defined here. A Reporting API option would require new specification work; existing Reporting API support does not provide agent outcomes automatically. Retained records would need bounded storage and expiry, partitioning, and deletion when relevant browsing data or permission is cleared. Delivery would be best-effort, not a durable analytics queue.

Important alternatives are to make outcomes reporting-only, to report aggregates rather than individual outcomes, or not to expose them at all. A report-only design need not carry the same `interactionIds` as the observer sketch; omitting event-level joins may be essential to its privacy model.

Delayed or credential-less delivery is not inherently anonymous. The report destination, network metadata, URL fields in a report envelope, unique endpoint URLs, and joinable IDs can still link a report to a person or visit. Any reporting design must explicitly address those channels, not just omit cookies. A normal Reporting API endpoint is not an aggregation or anonymity service.

## Information needed from agents

| Information | Can browser hooks provide it? | What cooperation is needed? |
| --- | --- | --- |
| Tool name, origin, dispatch, completion, cancellation | Generally for browser-mediated WebMCP execution. | Well-defined hooks in WebMCP, including future declarative execution. |
| Caller category | Only for execution paths whose provenance the browser knows. | Browser/agent integration; extension-specific plumbing where applicable. |
| Interaction start, end, and attribution | Not reliably from tool calls or input events alone. | An agent lifecycle signal that the browser can scope to a document. |
| DOM actuation and navigation attribution | In part, if agent-originated actions are explicitly attributed. | Integration with actuation mechanisms; not inference from event timing patterns. |
| Coarse processing and awaiting-user intervals | Often not available to the page or browser today. | Optional agent state transitions, with no reasoning content. |
| Tool usefulness assessment | No. A fulfilled callback is not enough. | An optional, consistently defined agent assessment. |
| Explicit user feedback | Only the agent UI or other feedback collector knows it. | A permitted, narrowly scoped feedback signal; not access to the conversation. |
| Task assessment | No general browser-only ground truth. | Optional agent assessment and a safe attribution policy. |

For browser-integrated agents, this could use internal browser interfaces. Extension agents would need an appropriate extension API or another browser-mediated bridge. Without it, extension identity or outcomes may be unavailable. For in-page agents, applications can already instrument their own code and communicate with their provider. Whether they need a standardized way to contribute assessments, and how those assessments would be distinguished from browser-provided entries, is unresolved.

Remote or OS-level agents may appear as ordinary input or automation. We should not introduce behavioral fingerprinting to guess their identity. Broader agent identity and delegation work may supply trustworthy attribution primitives, but this measurement proposal does not define that protocol or confer authority on an agent.

## Delivery, buffering, and document lifecycle

- **Scope:** Begin with secure-context `Window` documents. Tool-call entries belong to the document that registered and executes the tool. There is no automatic parent-frame, caller-frame, or top-level aggregation, even when cross-origin execution is allowed.
- **Completion:** The simplest interaction model emits one completed entry for each eligible document-local interaction. It cannot promise delivery when a document disappears mid-interaction.
- **Visibility:** The straw proposal also suggested emission on visibility changes. Partial snapshots would need explicit finality or segmentation semantics to prevent double-counting. A visibility change must not itself mean task completion, abandonment, or a new turn. This remains open.
- **Buffering:** Specify finite per-type buffers and caps on steps, interaction references, and retained outcomes. Late observers see only retained, eligible completed entries. Polling or re-observation must not defeat privacy limits.
- **Lifecycle transitions:** Navigation, destruction, prerendering, freezing, and back/forward-cache restoration need explicit handling. Restoring a document must not replay already delivered measurements as new interactions. No cross-origin history may be reconstructed from lifecycle gaps.
- **Availability:** Agent support, user controls, browser policy, and privacy protections may suppress any optional signal. No entry is not a negative assessment. Optional signals will have selection bias and should not be treated as population-wide success rates.

## Security and privacy considerations

### Data minimization is necessary, not sufficient

Do not expose conversation content, user goals, arguments, results, exception text, DOM targets, or navigation destinations through these entries. Even enums can reveal sensitive information when linked to a particular page, visit, action, or time. A negative rating on a health or financial site may be sensitive without disclosing any text.

Tool names are supplied by the site and can themselves contain user-specific information. RUM collectors should use allowlisted names or local aggregation rather than blindly serialize entries. The API should not introduce additional agent-generated strings or unbounded metadata.

### Agent detection and fingerprinting

A coarse caller category or `agentInvoked` flag reveals use of an agent. Brand and model identifiers can add entropy beyond the browser's UA, and timing patterns may reveal model choice or operating mode. A site could register specially chosen tools and observe assessments, validation failures, or selection behavior to probe an agent.

Coarsening, sampling, rate limits, and restricting signals to actual local execution are candidate mitigations, not a completed privacy argument. Model identification, discovery, skipped-tool diagnostics, and safety-block reasons should not enter a minimal page-visible API by default. DevTools and genuinely privacy-preserving aggregate reporting should be evaluated as alternatives for richer diagnostics.

### User choice and agent feedback

Feedback given to an agent is not automatically feedback the user intended to share with a website. Agent processing intervals and task assessments can also disclose previously private behavior. Outcome sharing and other newly exposed agent data need a clear user-control and consent model, including any agent-provider privacy commitments. Merely configuring an analytics endpoint is not user consent.

This draft does not select a new permission prompt. Whether any signal needs explicit opt-in, a browser setting, a Permissions Policy, restricted reporting, or should not be released is part of the design work. A site can receive useful execution measurements even if outcomes never become available.

### Origin and document isolation

Cross-origin WebMCP exposure through `exposedTo` is permission to discover or execute a tool, not blanket permission to inspect an agent's timeline. Do not expose another document's interactions, agent processing, navigation history, or outcomes. Caller-origin disclosure needs its own justification and policy. Any future delegation or embedding controls must be enforced by the UA, not by a JavaScript convention.

Third-party script running with the receiving document's privileges can generally observe what that document can observe. The proposal cannot describe page-visible entries as accessible only to a site's preferred analytics library.

### Reporting and correlation

Document-local IDs prevent an API-provided global join key, but sites can still associate measurements with their own accounts, cookies, timestamps, and business events. Delayed reports can remain linkable without cookies. Evaluate minimum aggregation thresholds, partitioning, batching, bounded retention, removal of identifying envelope fields, and whether event-level correlation should be available at all.

The privacy model must also consider colluding origins, unique report endpoints, private browsing, permission revocation, and browsing-data deletion. No quantitative precision, retention, or aggregation threshold is settled in this draft.

### Integrity, authorization, and abuse

Only browser-established attribution may label a native measurement as browser- or extension-originated. An untrusted page must not be able to inject an assessment into another origin's timeline or impersonate a browser agent through a claimed brand string.

Nevertheless, ordinary RUM is not an attested record: clients can fabricate network submissions, agents can provide inaccurate assessments, and pages can manipulate their own behavior. Collection services still need appropriate server-side validation, authorization where relevant, rate limiting, and abuse handling. These signals must not be used as proof of identity, consent, payment authorization, or safe execution.

Measurement must not alter WebMCP execution permissions or expose internal safety decisions to the page. Bounded buffers and rate limits are also needed so an agent or a malicious page cannot create unbounded memory, CPU, or reporting work.

## Accessibility and internationalization

- Human confirmation, correction, and assistive-technology use must not be treated as failures or unnecessary latency. Assistive technology must not be classified as an agent merely because it synthesizes input.
- Timing and outcome signals cannot establish that a UI update was perceivable or that an accessibility announcement was delivered or understood. Investigate those needs with accessibility experts rather than infer them from visibility or speed.
- Observation itself introduces no required UI. Any eventual consent, feedback, or developer-facing controls must be accessible, localizable, and independent of a particular input modality.
- Enums are machine-readable values, not user-facing English labels. Feedback semantics, task boundaries, and the time needed for a user to respond may vary across languages, cultures, and workflows.
- The performance timeline uses monotonic relative times; measurement should not depend on a user's locale or wall-clock formatting.

## Alternatives considered

### Author instrumentation and User Timing only

Developers can wrap tools and create [User Timing](https://www.w3.org/TR/user-timing/) measures today. This is a useful baseline and fallback, and sites should continue using their own business-outcome data. However, it requires author integration, is difficult for independent RUM libraries to apply consistently, and lacks browser-established caller provenance or agent-side interaction information.

### Agent-provider dashboards and bilateral backend integrations

Providers can offer richer analytics through their own services, with explicit contractual and privacy arrangements. This avoids exposing some signals to page script. It also requires sites to integrate with each provider, excludes providers without such services, and does not give a provider-neutral browser measurement surface. It may remain appropriate for sensitive data that should not become a web API.

### Feedback tools or extra tool parameters

A site could register a `provideFeedback` tool or ask the agent to rate a tool through its arguments. This needs no new platform API, but changes the tool interface, depends on agent interpretation and extra calls, and can create the same privacy leakage as a native feedback field. It does not give third-party RUM a uniform observation mechanism or establish trustworthy provenance.

### A new AgenticPerformanceObserver or DOM events

A dedicated observer could separate agent telemetry from the general timeline, while events could support immediate UI reactions. PerformanceObserver already provides asynchronous delivery and per-type buffering for RUM. Reusing it avoids another observation abstraction, but does not solve the harder questions of exposure, attribution, or lifecycle. Functional tool events should be retained where needed.

### Reporting-only or aggregate-only measurement

Keeping sensitive signals out of live page script could reduce immediate adaptation and probing. Aggregates could also reduce linkability. However, ordinary delayed Reporting API delivery alone does not provide those guarantees, and aggregate designs limit document-local diagnosis and joins. We should compare a basic execution observer plus a separate restricted outcome channel against making all signals observer entries. This alternative remains open.

### DevTools-only diagnostics

Local developer tooling could expose richer traces for debugging without making the same data readable by every visited page. It does not by itself answer field-measurement questions across real users, but may be the right home for model details, rejected tools, and detailed agent processing. It complements rather than necessarily replaces a minimal RUM surface.

## Dependencies and related work

- **Performance Timeline and High Resolution Time:** Supply observation and time semantics. New entry definitions must specify buffering, serialization, clock conversion, and privacy treatment; using these APIs does not itself settle safe timing precision.
- **WebMCP:** Supplies the tool registration and execution hooks. It is an evolving proposal, and its current specification and declarative exploration must not be treated as equally settled. `tool-call` depends on this work; generic interaction measurement need not.
- **Declarative WebMCP exploration:** Would require separate dispatch, completion, navigation, and cancellation integration. See the [declarative explainer](https://github.com/webmachinelearning/webmcp/blob/main/declarative-api-explainer.md).
- **Event Timing:** Provides related user-interaction measurements. Correlation with agent turns and any `agentInvoked` extension need joint design, not a silent change to existing IDs or responsiveness metrics.
- **Reporting API:** A possible delivery mechanism for outcomes beyond document lifetime, not an existing agent-outcome reporting feature or privacy-preserving aggregation service.
- **Agent identity and delegation:** Related WebML CG work may establish attribution and capability boundaries. This proposal must coordinate without making telemetry a new authentication or authorization API.
- **On-device AI observability and OpenTelemetry:** The [TPAC observability breakout](https://github.com/w3c/tpac2026-breakouts/issues/63) discusses complementary needs. Page-owned inference, token accounting, and distributed trace propagation are not prerequisites here; adapters could consume safe measurements without exposing prompts or global trace IDs through this API.

## Stakeholder feedback and evidence

The evidence so far is public developer discussion, implementation anecdotes, and a CG request for an explainer, not systematic end-user research or browser implementation commitments.

- [Dominic Farolino](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-4792411089) argued for measuring agent-mediated experiences generally, not only WebMCP calls.
- [Developer experience from stacktr.ee](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-4919276328) identifies caller classification and tool discovery as gaps despite author instrumentation.
- [Ryan Townsend](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-5510686305) asks for measurement without wrapping author code and coverage of declarative tools.
- [Brandon Walderman](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-4929491673) is open to a standard frontend interface but emphasizes user opt-in and the risk of disclosing private intent through feedback. Earlier comments discuss backend alternatives and data integrity.
- [Fingerprinting feedback](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-6413798793) questions brand/model exposure, step timings, probing of assessments, and which signals should be page-visible versus restricted reports or DevTools.
- The [WebML CG resolution](https://www.w3.org/2026/11/12-webmachinelearning-minutes.html#cad0) requests use cases, a rough API, and review with agent providers and privacy reviewers. It does not approve this particular API or establish a venue transfer.
- [Related observability work](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-6875052690) and a [commerce use-case contribution offer](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-7100095284) provide opportunities for collaboration. This draft does not treat offered benchmark or adoption figures as independently validated evidence.

We need structured input from users, website and RUM developers, browser and extension implementers, agent providers, accessibility experts, and privacy reviewers. Individual comments should not be interpreted as their employers' formal positions.

## Open questions and next steps

### Exposure and user control

1. What is the smallest useful page-visible surface: execution timing only, timing plus coarse caller type, or more?
2. Which assessments or outcomes, if any, can be exposed without disclosing private user intent? What consent and revocation model is appropriate?
3. Should outcomes be observer entries, restricted reports, aggregate reports, DevTools-only, or not exposed? What actual unlinkability guarantees would reporting require?
4. Is agent brand or model information necessary for any use case that cannot be served by coarser diagnostics? Should those fields be omitted entirely?
5. What precision, rate limits, budgets, buffer caps, and retention rules balance utility with fingerprinting and resource-use risks?

### Attribution and semantics

6. How should document-local participation in a cross-page turn be bounded without revealing time spent elsewhere?
7. What constitutes tool completion for declarative or navigation-triggering tools, input validation, and cancellation races?
8. Can agents provide consistent usefulness and task assessments? How are explicit feedback, no feedback, unavailability, and withholding distinguished safely?
9. How should interaction entries relate to Event Timing, rendering updates, and human interventions? Are visibility snapshots necessary, and how would partial and final entries be distinguished?
10. What browser-mediated integration can extension agents use? Do in-page agents need a standard assessment channel, and how is its provenance represented?
11. Does discovery measurement justify a separate signal, or should tool enumeration, selection, and safety rejection remain outside page-visible telemetry?

### Collaboration plan

- Gather concrete examples, including mixed tool/DOM flows, consequential actions, and accessible human oversight. Separate data sites already hold from new information requested from agents.
- Ask agent providers which lifecycle and assessment signals they can supply reliably, optionally, and with user control.
- Review the threat model and exposure alternatives with privacy and security reviewers before treating sensitive fields as part of the API.
- Coordinate with the WebML CG, Web Performance WG, relevant horizontal reviewers, and WICG chairs on scope and incubation venue. The intended WICG proposal is a request, not an assertion that adoption has occurred.
- After those discussions, narrow the initial surface, write detailed lifecycle and permission rules, and identify implementers and interoperable tests. Prototype data must be synthetic or collected with appropriate user authorization.

## References and acknowledgements

- [Original WebMCP RUM issue](https://github.com/webmachinelearning/webmcp/issues/186) and [source straw proposal](https://github.com/webmachinelearning/webmcp/issues/186#issuecomment-6369314489).
- [WebML CG meeting, 12 November 2026](https://www.w3.org/2026/11/12-webmachinelearning-minutes.html#cad0).
- [Performance Timeline](https://www.w3.org/TR/performance-timeline/), [High Resolution Time](https://www.w3.org/TR/hr-time/), [Event Timing](https://www.w3.org/TR/event-timing/), [User Timing](https://www.w3.org/TR/user-timing/), and [Reporting API](https://www.w3.org/TR/reporting/).
- [WebMCP specification](https://webmachinelearning.github.io/webmcp/) and its discussion of [privacy leakage through over-parameterization](https://webmachinelearning.github.io/webmcp/#privacy-leakage-over-parameterization).
- [TAG explainer guidance](https://w3ctag.github.io/explainers/) and [WICG proposal process](https://github.com/WICG/admin#contributing-new-proposals).

Thanks to the participants in WebMCP issue #186, the WebML CG discussions, and the related observability work for motivating and challenging this proposal. Participation in those discussions does not imply endorsement of this draft or authorship of its new text.
