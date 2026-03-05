# PM Spec: CRU-431 — Live State Aggregator

**Project:** Ops | **Canon Tier:** Lightweight | **PM:** CruzBot

---

## 1. Problem Statement

CruzBot's operational awareness is fundamentally impaired by its reliance on stale, file-based state. Agents read from `MEMORY.md` and other markdown files that become outdated the moment they are written, leading to incorrect status reports, redundant agent spawns, and a general lack of trust in the system's perception of reality. To operate effectively, agents need access to a live, authoritative source of truth for active work, session status, and code changes, rather than a snapshot of the past.

## 2. Goals & Non-Goals

### Goals
- **Provide a single, authoritative script (`get-state.js`)** for querying the live operational state of the CruzBot system.
- **Eliminate agent reliance on stale markdown files** for tracking issue status, active sessions, and PRs.
- **Improve observability** by correctly wiring session identifiers to Langfuse for built-in session tracking.
- **Increase system reliability** by adding Langfuse to the standard `auth-status` health check.
- **Formalize the process change** by updating core documentation to deprecate file-based state tracking.

### Non-Goals
- **Build a new database or state store.** The goal is to query existing authoritative sources (Linear, Langfuse, GitHub), not create a new one.
- **Implement a full context-injection plugin in the initial phase.** Automatic injection is a future enhancement, not a day-one requirement.
- **Integrate third-party semantic memory tools** like Mem0 or Zep. These solve a different problem (conversational memory) and are out of scope.
- **Create a REST API on the OpenClaw gateway.** The analyst report confirms no such API exists, and we will not build one for this project.

## 3. User Stories

- As CruzBot, I need a single script to call (`get-state.js`) so that I can get a real-time summary of all active work, sessions, and PRs.
- As CruzBot, I need `get-state.js` to query live APIs so that I am not making decisions based on stale file-based information.
- As CruzBot, I need my session key to be mapped to the Langfuse `sessionId` so that I can use Langfuse's native session grouping for better observability and cost tracking.
- As CruzBot, I need `auth-status` to check Langfuse so that I can quickly diagnose if my telemetry pipeline is broken.
- As CruzBot, I need core documentation (`AGENTS.md`) updated so that my future evolution avoids the anti-pattern of using files for state management.

## 4. Phased Delivery

This project will be delivered in three distinct phases, prioritizing the highest-value, lowest-risk items first.

### Phase 1: The Core Script & Health Check
*(Corresponds to AC #1 & #3)*
The immediate goal is to create the core tool and improve diagnostics.
- **Deliverable:** A functional `scripts/ops/get-state.js` that queries Linear, GitHub, and Langfuse for live state and outputs a clean summary.
- **Deliverable:** The `scripts/ops/auth-status.js` script is updated to include a health check for the Langfuse connection.

### Phase 2: Langfuse Session Correlation
*(Corresponds to AC #2)*
With the core script in place, we will improve the underlying data quality in Langfuse.
- **Deliverable:** Investigate and implement the mapping of OpenClaw's `sessionKey` to the Langfuse `sessionId`. This may be a configuration change or a small, targeted plugin modification.
- **Benefit:** This unlocks powerful session-level analytics in Langfuse and simplifies the `get-state.js` logic.

### Phase 3: Process Integration & Documentation
*(Corresponds to AC #4)*
Finally, we will formalize the new operational standard and update our internal processes.
- **Deliverable:** Update `AGENTS.md` and `HEARTBEAT.md` to reflect the new standard: use `get-state.js` instead of reading files for operational state.

## 5. Success Metrics

- **Metric:** The `get-state.js` script successfully returns live, accurate data from at least two of the three sources (Linear, GitHub, Langfuse) within 5 seconds.
- **Metric:** After Phase 2, the Langfuse `/api/public/sessions` endpoint returns a non-zero number of sessions, and traces in the UI are grouped by session.
- **Metric:** After Phase 1, `auth-status` correctly reports the health of the Langfuse connection.
- **Metric:** Within one week of Phase 3 completion, agent error logs show a >50% reduction in errors related to stale state ("issue already closed", "PR already merged", etc.).
- **Metric:** Manual spot-checks of agent status reports (`/status`) show they are citing live data from `get-state.js` instead of `MEMORY.md`.

## 6. Risks

- **Medium Risk (Phase 2):** The mechanism for mapping `sessionKey` to Langfuse's `sessionId` is unknown. It may require a modification to the OpenClaw gateway source if it's not exposed via configuration or a simple plugin, which would increase complexity.
- **Low Risk (API Changes):** The external APIs (Linear, Langfuse, GitHub) could change, but they are stable, and our usage is straightforward. The script will be designed to handle individual source failures gracefully.
- **Low Risk (Performance):** The `get-state.js` script relies on multiple network requests. A slow response from any one service could delay the output, but the impact is minimal for an on-demand CLI tool.

## 7. Scope Decisions

This table clarifies what was recommended by the analyst and what is being included or deferred in this project's scope.

| Analyst Recommendation / Finding | PM Decision | Justification |
| --- | --- | --- |
| **Gateway has NO REST API** for live session state. | **ACCEPT & ELIMINATE** | We will not attempt to query the gateway directly. `get-state.js` will rely on Langfuse for session information. |
| **Langfuse `sessionId` is `null`** on all traces. | **ACCEPT - PHASE 2** | This is a critical fix for observability and is the core of Phase 2. We will not ship the final solution without it. |
| **Three live sources available:** Linear, GitHub, Langfuse. | **ACCEPT** | These are the exact three sources `get-state.js` will query. |
| Build `get-state.js` as the core deliverable. | **ACCEPT - PHASE 1** | This is the central piece of the project and the primary focus of Phase 1. |
| Add Langfuse to `auth-status.js`. | **ACCEPT - PHASE 1** | A low-effort, high-value task that fits perfectly into the initial phase. |
| Build a **Context-Injector Extension**. | **DEFER** | While a valuable enhancement, it adds complexity (caching, latency, TS compilation). It is deferred to a future project after the core script is proven stable and reliable. |
| Integrate **Mem0/Zep/Letta**. | **REJECT** | These are semantic memory tools and do not solve the problem of live operational state tracking. They are out of scope. |
| Build a **SQLite state store**. | **REJECT** | This would recreate the stale data problem we are trying to solve. Querying live APIs is the correct approach. |

---
*Spec complete. Ready for Architecture review.*
