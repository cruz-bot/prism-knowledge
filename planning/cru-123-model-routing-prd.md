# Product Requirements Document — Model Routing Strategy & Subagent Architecture

**Author:** John (BMAD PM)  
**Date:** 2026-02-22  
**Linear Issue:** CRU-123  
**Priority:** P0  
**Status:** Draft

---

## 1. Executive Summary

OpenClaw spawns subagents for dev, review, QA, research, and planning work. Today, model assignment is ad-hoc — agents get whatever default is configured, rate limits cause hard failures, and only 2-3 of 12+ available models are used. This PRD defines a tiered model routing system with persona-based assignment, multi-provider fallback chains, and config-driven control — so the right model handles the right task, failures gracefully degrade, and cost is optimized.

---

## 2. Problem Statement

| Problem | Impact |
|---------|--------|
| No systematic model assignment | PR reviews run on cheap models; simple fixes waste expensive ones |
| Rate limits = hard failures | Subagent dies, work lost, manual restart required |
| 12+ models available, 2-3 used | Paying for provider access that sits idle |
| No persona-based routing | All agents treated identically regardless of task complexity |
| No provider load balancing | Single provider saturates while others have capacity |
| Config changes require code edits | Tony can't adjust routing without touching source |

**Root cause:** OpenClaw lacks a model routing layer between "spawn subagent" and "pick a model." Every agent gets the same default with no intelligence about task requirements or provider availability.

---

## 3. User & Context

**User:** Tony — solo developer, intermediate skill level.  
**Platform:** OpenClaw running on Windows, used as AI assistant for software development.  
**Providers:** Anthropic (direct API), Google Gemini CLI (OAuth), Google Antigravity (OAuth).  
**Workflow:** Canon-based (Analyst → PM → Architect → SM → Dev → QA → PR → Merge).

**What Tony values:**
- Autonomy: system should make smart routing decisions without his intervention
- Efficiency: don't burn Opus tokens on formatting tasks
- Reliability: never fail due to rate limits when fallback capacity exists
- Visibility: know what model each agent used and why

---

## 4. Goals & Success Metrics

| Goal | Metric | Target |
|------|--------|--------|
| Right model for right task | % of subagents using tier-appropriate model | 100% |
| Zero hard failures from rate limits | Rate-limit-induced agent deaths per week | 0 |
| Full inventory utilization | Models with >0 usage per month | ≥10 of 12+ |
| Cost optimization | Monthly model spend vs. all-Opus baseline | ≤60% |
| Config-driven changes | Time to change model routing | <2 min (edit + restart) |
| Observability | Subagent runs with model attribution | 100% |

---

## 5. Functional Requirements

### FR-1: Tiered Model Classification

Three tiers based on task complexity:

| Tier | Primary Models | Use Cases |
|------|---------------|-----------|
| **Heavy** | Opus 4.6, Gemini 3 Pro | PR reviews, BMAD planning, architecture, ambiguous problems |
| **Medium** | Sonnet 4.6, Gemini 3 Flash | Dev work, code fixes, QA, docs, research |
| **Fast** | Haiku 4.5, Gemini 2.5 Flash | Simple fixes, formatting, lookups, bulk ops |

### FR-2: Fallback Chains (Never Fail)

Each tier has a fallback chain that:
1. Tries the same model on a different provider before downgrading tier
2. Crosses all 3 providers before giving up on a tier
3. Degrades to the next tier down as last resort
4. Chain depth: 4-5 models minimum per tier

**Heavy chain:**
```
anthropic/claude-opus-4-6
→ google-antigravity/claude-opus-4-6
→ google-gemini-cli/gemini-3-pro-preview
→ google-gemini-cli/gemini-2.5-pro
```

**Medium chain:**
```
anthropic/claude-sonnet-4-6
→ anthropic/claude-haiku-4-5
→ google-antigravity/claude-sonnet-4-5
→ google-gemini-cli/gemini-3-flash-preview
→ google-gemini-cli/gemini-2.5-flash
```

**Fast chain:**
```
anthropic/claude-haiku-4-5
→ google-gemini-cli/gemini-2.5-flash
→ google-gemini-cli/gemini-2.0-flash
```

### FR-3: Subagent Persona → Tier Mapping

| Persona | Label Pattern | Tier | Model Override | Rationale |
|---------|--------------|------|----------------|-----------|
| **Dev Agent** | `dev-{ID}` | Medium | *(default)* | Standard implementation work |
| **PR Reviewer** | `pr-review-{ID}` | Heavy | `opus` | Quality gate — no shortcuts |
| **Fix Agent** | `fix-{ID}` | Medium | *(default)* | Targeted corrections |
| **Planning Agent** | `plan-{PHASE}-{ID}` | Heavy | `opus` | Strategic thinking, artifact production |
| **Research Agent** | `research-{TOPIC}` | Medium | `gemini` | Large context window advantage |
| **QA Agent** | `qa-{ID}` | Medium | *(default)* | Systematic testing |

### FR-4: Config-Driven Routing

All routing configuration lives in `openclaw.json`:

```json
{
  "model": {
    "primary": "anthropic/claude-opus-4-6",
    "fallbacks": [
      "google-antigravity/claude-opus-4-6",
      "google-gemini-cli/gemini-3-pro-preview",
      "google-gemini-cli/gemini-2.5-pro",
      "google-gemini-cli/gemini-2.5-flash"
    ]
  },
  "subagents": {
    "model": {
      "primary": "anthropic/claude-sonnet-4-6",
      "fallbacks": [
        "anthropic/claude-haiku-4-5",
        "google-antigravity/claude-sonnet-4-5",
        "google-gemini-cli/gemini-3-flash-preview",
        "google-gemini-cli/gemini-2.5-flash"
      ]
    }
  }
}
```

Per-spawn overrides use the `model` parameter in `sessions_spawn()` to select a different primary (e.g., `opus` for PR reviewers). Fallbacks inherit from config.

### FR-5: Provider Priority Order

1. **Anthropic direct** (`anthropic:default`) — primary, lowest latency
2. **Google Gemini CLI** (`google-gemini-cli`) — secondary, large context advantage
3. **Google Antigravity** (`google-antigravity`) — tertiary, same Claude models via different quota

Rationale: Exhaust Anthropic quota first (best quality for Claude models), then Google's Claude quota, then Gemini native models.

### FR-6: Model Inventory Registration

All 12+ models must be registered in `openclaw.json` models section so they're available for fallback chains. Currently missing:
- `anthropic/claude-sonnet-4-6` (new)
- `anthropic/claude-haiku-4-5` (new)
- `google-gemini-cli/gemini-3-pro-preview` (new)
- `google-gemini-cli/gemini-3-flash-preview` (new)
- `google-antigravity/claude-opus-4-6-thinking` (thinking variant)

---

## 6. Non-Functional Requirements

### NFR-1: Zero-Downtime Model Switching
- Change routing by editing `openclaw.json` + `openclaw gateway restart`
- No code changes, no redeployment
- Gateway restart completes in <5 seconds

### NFR-2: Observability
- Every subagent log must include: model requested, model actually used, provider
- If fallback triggered: log which models were tried and why they failed
- Queryable after the fact (log files or memory notes)

### NFR-3: Cost Awareness
- Default to cheapest model that meets quality requirements for task type
- Never use Heavy tier for tasks that Medium handles equivalently
- Track approximate spend per tier per week (manual review acceptable for v1)

### NFR-4: Latency
- Fallback chain traversal should add <5s per hop
- Fast tier should resolve to a working model in <3s total

### NFR-5: Resilience
- Total provider outage (all 3 down) is the ONLY acceptable failure mode
- Partial outages (1-2 providers down) must be fully transparent to the user

---

## 7. Scope

### In Scope (v1)
- Tiered model classification (Heavy/Medium/Fast)
- Fallback chain configuration in `openclaw.json`
- Per-spawn model overrides for persona types
- Full model inventory registration
- AGENTS.md documentation update
- Subagent architecture doc (`docs/subagent-architecture.md`)

### Out of Scope (v1)
- Automatic rate limit detection and proactive provider switching (OpenClaw handles fallback internally)
- Cost tracking dashboard or automated reporting
- Dynamic tier assignment based on task analysis (tiers are manually assigned per persona)
- Token budget enforcement per subagent
- A/B testing models against each other

### Future Considerations (v2+)
- Automatic task complexity scoring → tier assignment
- Real-time cost tracking with alerts
- Model performance benchmarking per task type
- Token budget caps per agent type

---

## 8. Decision Tree (Routing Logic)

```
1. Is it the main session (direct chat with Tony)?
   → Heavy: opus (full Heavy fallback chain)

2. Is it a quality gate (PR review)?
   → Heavy: opus override

3. Is it planning/architecture (BMAD phases)?
   → Heavy: opus override

4. Is it research needing large context?
   → Medium: gemini override (Gemini 2.5 Pro primary)

5. Is it standard dev work, fixes, or QA?
   → Medium: default subagent chain

6. Is it a simple/bulk task (formatting, lookups)?
   → Fast: haiku or flash override
```

---

## 9. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| New models (Sonnet 4.6, Haiku 4.5) not yet in OpenClaw config | High | Blocks deployment | Add to config as first implementation step |
| Gemini 3 preview models unstable | Medium | Fallback chain breaks at that hop | Place after stable models in chain; chain continues past failures |
| Provider auth expires (OAuth tokens) | Low | Provider unavailable | Fallback chain covers; re-auth is quick manual step |
| Wrong tier assigned to persona | Medium | Quality issues or wasted spend | Document rationale; easy to adjust via config |

---

## 10. Implementation Epics

### Epic 1: Config & Model Registration
- Register all 12+ models in `openclaw.json` models section
- Configure main session fallback chain (Heavy)
- Configure default subagent fallback chain (Medium)
- Verify all providers authenticate successfully

### Epic 2: Persona Routing & Documentation
- Update AGENTS.md model routing section with tier table and persona mappings
- Update `docs/subagent-architecture.md` as canonical reference
- Document per-spawn override patterns for each persona type
- Add decision tree to AGENTS.md for quick reference

### Epic 3: Validation & Rollout
- Test each fallback chain (simulate rate limit on primary)
- Verify per-spawn overrides work (spawn test agent with `opus`, verify model used)
- Verify observability (check logs for model attribution)
- Apply config patch to production gateway

---

## 11. Acceptance Criteria

1. ✅ All 12+ models registered in `openclaw.json`
2. ✅ Main session uses Heavy tier with full fallback chain
3. ✅ Default subagent uses Medium tier with full fallback chain
4. ✅ PR Reviewer spawned with `opus` override uses Heavy chain
5. ✅ Research Agent spawned with `gemini` override uses Gemini-primary chain
6. ✅ Rate limit on primary model triggers fallback (not failure)
7. ✅ Model used by each subagent visible in logs
8. ✅ Routing changes require only `openclaw.json` edit + restart
9. ✅ AGENTS.md and architecture doc updated with final routing tables
10. ✅ All 3 providers used in at least one fallback chain

---

## Appendix A: Full Model Inventory

| Model | Provider | ID | Tier | Cost (in/out MTok) |
|-------|----------|----|------|-------------------|
| Claude Opus 4.6 | Anthropic | `anthropic/claude-opus-4-6` | Heavy | $5/$25 |
| Claude Sonnet 4.6 | Anthropic | `anthropic/claude-sonnet-4-6` | Medium | $3/$15 |
| Claude Sonnet 4.5 | Anthropic | `anthropic/claude-sonnet-4-5` | Medium | $3/$15 |
| Claude Haiku 4.5 | Anthropic | `anthropic/claude-haiku-4-5` | Fast | $1/$5 |
| Claude Opus 4.6 | Antigravity | `google-antigravity/claude-opus-4-6` | Heavy | (Google quota) |
| Claude Opus 4.6 Thinking | Antigravity | `google-antigravity/claude-opus-4-6-thinking` | Heavy+ | (Google quota) |
| Claude Sonnet 4.5 | Antigravity | `google-antigravity/claude-sonnet-4-5` | Medium | (Google quota) |
| Gemini 3 Pro | Gemini CLI | `google-gemini-cli/gemini-3-pro-preview` | Heavy | (Google quota) |
| Gemini 2.5 Pro | Gemini CLI | `google-gemini-cli/gemini-2.5-pro` | Medium-Heavy | (Google quota) |
| Gemini 3 Flash | Gemini CLI | `google-gemini-cli/gemini-3-flash-preview` | Medium | (Google quota) |
| Gemini 2.5 Flash | Gemini CLI | `google-gemini-cli/gemini-2.5-flash` | Fast | (Google quota) |
| Gemini 2.0 Flash | Gemini CLI | `google-gemini-cli/gemini-2.0-flash` | Fast | (Google quota) |
