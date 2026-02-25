# Product Requirements Document — Observability Workflows & Operational Playbook (Langfuse)

**Author:** John (BMAD PM)
**Date:** 2026-02-22
**Linear Issue:** CRU-127
**Depends on:** CRU-126 (Done — technical integration), CRU-123 (model routing architecture)
**Status:** Draft

---

## 1. Problem Statement

CRU-126 connected the pipe: OpenClaw now exports OTEL traces to Langfuse Cloud. But a telemetry pipe without operational workflows is just noise. Nobody defined **when** to look at traces, **what** to look for, or **how** insights feed back into decisions.

Today's gaps (from CRU-123 AD-4):
- No way to verify which model a subagent actually ran on after fallback
- No automated cost tracking
- No fallback event alerting
- Debugging subagent failures relies on memory files and gateway logs — traces exist but aren't part of the workflow

Without defined workflows, the observability investment produces data that rots unread.

---

## 2. Goals

1. **Define when and why to review traces** — triggers, not schedules-for-the-sake-of-schedules
2. **Create a Langfuse playbook** — concrete steps for the 3-4 scenarios that actually matter
3. **Integrate observability into existing Canon workflows** — lightweight touches, not new bureaucracy
4. **Enable data-driven model routing decisions** — replace gut feelings with trace evidence
5. **Track costs against the free tier budget** — 50k observations/month awareness

## 3. Non-Goals

- **Building dashboards or custom tooling** — use Langfuse's built-in UI, filters, and views
- **Real-time alerting infrastructure** — no webhook/PagerDuty/Slack alerts; this is a solo dev setup
- **Modifying OpenClaw source** — we consume what OTEL exports, we don't instrument new spans
- **Comprehensive APM** — we care about LLM-specific signals (model attribution, cost, fallbacks), not generic infra metrics
- **Automated remediation** — humans (Tony + main agent) make decisions; traces inform, not act

---

## 4. User Stories

The "users" are Tony (human) and the main agent (Cruz).

### US-1: Subagent Failure Diagnosis
**As** Tony or Cruz, **when** a subagent fails or times out, **I want** a step-by-step playbook for checking Langfuse traces **so that** I can determine if the failure was model-related (fallback exhaustion, provider error) vs. task-related (bad prompt, context overflow).

**Acceptance Criteria:**
- Playbook covers: how to find the trace by session/label, what fields to check, decision tree for root cause
- Playbook is in `docs/observability-playbook.md`
- Main agent can follow the playbook autonomously (no Tony intervention needed for diagnosis)

### US-2: Weekly Model Routing Review
**As** Tony, **during** the weekly review, **I want** a checklist for reviewing Langfuse data **so that** I can spot fallback patterns, cost trends, and model performance issues in ≤10 minutes.

**Acceptance Criteria:**
- Checklist with specific Langfuse filters/views to check
- Covers: fallback frequency, cost-per-model, error rates, free-tier usage
- Produces a brief summary suitable for `memory/YYYY-MM-DD.md`

### US-3: Model Routing Change Validation
**As** Cruz, **before and after** changing model routing config (openclaw.json), **I want** to capture baseline trace metrics and compare post-change **so that** routing changes are evidence-based, not vibes-based.

**Acceptance Criteria:**
- Pre-change: document current fallback rate, primary model success rate, avg latency
- Post-change: compare same metrics after 24-48h of usage
- Decision: keep, revert, or adjust

### US-4: Cost & Quota Awareness
**As** Tony, **I want** to know when Langfuse free-tier usage is approaching limits **so that** I can adjust before hitting the wall.

**Acceptance Criteria:**
- Monthly usage check is part of the weekly review checklist
- Threshold: flag when >70% of 50k observations consumed before month's 70% mark
- Action plan: reduce trace verbosity or upgrade tier

### US-5: PR Review Trace Context (Lightweight)
**As** a PR Reviewer agent, **when** reviewing a subagent's dev work, **I want** optional trace context available **so that** if code quality seems off, I can check whether the agent fell back to a weaker model.

**Acceptance Criteria:**
- PR review skill updated with a single optional step: "If quality concerns, check Langfuse for session traces"
- NOT mandatory for every PR — only when something seems wrong
- Keeps PR review fast; this is an escape hatch, not a gate

---

## 5. Functional Requirements

### FR-1: Observability Playbook Document

Create `docs/observability-playbook.md` containing:

| Section | Content |
|---|---|
| **Quick Reference** | Langfuse URL, service name (`cruzbot`), how to filter by session/label |
| **Scenario 1: Subagent Failure** | Step-by-step trace diagnosis (US-1) |
| **Scenario 2: Weekly Review** | Checklist with specific filters (US-2) |
| **Scenario 3: Routing Validation** | Before/after comparison protocol (US-3) |
| **Scenario 4: Quota Check** | How to check usage, thresholds, actions (US-4) |
| **Model Cost Reference** | Per-token costs for each of the 14 configured models (for Langfuse cost tracking config) |

### FR-2: Langfuse Cost Configuration

Configure model costs in Langfuse so the dashboard shows dollar amounts, not just token counts.

Models to configure (all 14):
- `anthropic/claude-opus-4-6`
- `anthropic/claude-sonnet-4-6` (verify existence; fallback `sonnet-4-5`)
- `anthropic/claude-haiku-4-5`
- `google-antigravity/claude-opus-4-6`
- `google-antigravity/claude-opus-4-6-thinking`
- `google-antigravity/claude-sonnet-4-5`
- `google-gemini-cli/gemini-3-pro-preview`
- `google-gemini-cli/gemini-3-flash-preview`
- `google-gemini-cli/gemini-2.5-pro`
- `google-gemini-cli/gemini-2.5-flash`

Source: provider pricing pages (fetch at implementation time; prices change).

### FR-3: Workspace Documentation Updates

| File | Change |
|---|---|
| `AGENTS.md` | Add "Observability" subsection under Wolverine protocol: when to check traces, link to playbook |
| `TOOLS.md` | Add Langfuse credentials section (URL, project, how to access) |
| `docs/subagent-architecture.md` | Add observability section replacing AD-4's "log-based" placeholder with Langfuse workflow references |

### FR-4: Saved Views / Filters in Langfuse

Create and document these saved views in Langfuse UI (manual setup, documented in playbook):

1. **Fallback Traces** — filter where model ≠ expected primary (indicates fallback occurred)
2. **Error Traces** — filter on error/failure status
3. **By Model** — group by model name, show count + avg latency + cost
4. **High Cost Sessions** — sort by total cost descending
5. **Recent 24h** — time-filtered view for post-change validation

### FR-5: Weekly Review Integration

Add to the existing weekly review process:
- A 5-item Langfuse checklist (see US-2)
- Output: 3-5 line summary appended to that week's memory file
- Time budget: ≤10 minutes

---

## 6. Success Metrics

| Metric | Target | How Measured |
|---|---|---|
| **Time to diagnose subagent failure** | <5 min with playbook (vs. ~20 min digging through logs today) | Qualitative — Tony's assessment after 2-3 incidents |
| **Weekly review completion** | Happens ≥3 of 4 weeks/month | Memory file entries |
| **Model routing decisions backed by data** | 100% of routing config changes reference trace data | PR/commit descriptions |
| **Free tier budget awareness** | Never hit 50k limit unexpectedly | Monthly check in weekly review |
| **Playbook usability** | Main agent can execute all scenarios without asking Tony for help | Observed in practice |

---

## 7. Risks & Mitigations

| Risk | Severity | Mitigation |
|---|---|---|
| **OTEL traces don't capture model-after-fallback** — OpenClaw may not include which model actually served the request in the span | High | Investigate trace schema in Langfuse. If model attribution is missing, file as OpenClaw feature request. Workaround: infer from latency/token patterns. |
| **Free tier exhaustion** — 50k obs/month may not be enough with active subagent usage | Medium | Weekly quota check (FR-5). If consistently >80%, either reduce trace sampling or upgrade to Langfuse Pro ($59/mo). |
| **Playbook goes stale** — model names, Langfuse UI, or workflows change | Low | Playbook is a living doc. Wolverine protocol: when you hit friction, update the playbook. |
| **Over-engineering** — spending more time on observability workflows than they save | Medium | Keep it lightweight. The playbook is ~4 scenarios. Weekly review is ≤10 min. If it's not earning its keep after a month, simplify. |
| **Langfuse Cloud reliability** — third-party SaaS dependency | Low | Traces are fire-and-forget (async OTEL export). Langfuse being down doesn't affect agent operation, only review capability. |

---

## 8. Implementation Scope

**Estimated effort:** Small feature (Lightweight Canon: PM → SM → Dev → PR → Merge)

**Epics:**

### Epic 1: Observability Playbook & Cost Config
- Write `docs/observability-playbook.md` (FR-1)
- Configure model costs in Langfuse UI (FR-2)
- Set up saved views/filters in Langfuse (FR-4)

### Epic 2: Workspace Integration
- Update `AGENTS.md` with observability section (FR-3)
- Update `TOOLS.md` with Langfuse credentials (FR-3)
- Update `docs/subagent-architecture.md` observability section (FR-3)
- Add weekly review checklist (FR-5)

**Total stories:** ~4-6 (to be broken down by SM)

---

## 9. Open Questions

1. **What does the OTEL trace schema actually look like in Langfuse?** — We need to inspect real traces to know what fields are available (model name, token counts, latency, error codes). This should be the first task: explore existing data.
2. **Does OpenClaw tag traces with the subagent label?** — If yes, filtering by `pr-review-CRU-45` is trivial. If not, we need another correlation strategy.
3. **Langfuse API access** — Can we query Langfuse programmatically (e.g., for automated weekly summaries), or is it UI-only on Hobby tier?

---

## 10. Approval

- [ ] Tony approves PRD scope
- [ ] Architecture review (if needed — likely skip for this scope)
- [ ] SM breaks into stories
