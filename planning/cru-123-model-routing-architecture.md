# Architecture Decision Document — Model Routing Strategy & Subagent Architecture

**Author:** Winston (BMAD Architect)  
**Date:** 2026-02-22  
**Linear Issue:** CRU-123  
**PRD:** `cru-123-model-routing-prd.md`  
**Status:** Draft

---

## 1. Overview

This document captures the key architecture decisions for CRU-123: implementing tiered model routing with multi-provider fallback chains in OpenClaw. We configure OpenClaw — we don't modify its source. All decisions constrain to what `openclaw.json` config and `sessions_spawn({ model })` support.

---

## 2. Architecture Decisions

### AD-1: Fallback Chain Ordering — Same-Model-Cross-Provider Before Tier Downgrade

**Decision:** Yes, the PRD's ordering is correct. Same model on a different provider BEFORE downgrading to a weaker model.

**Rationale:**
- **Quality preservation:** Opus on Antigravity is still Opus. Sonnet on Antigravity is still Sonnet. Downgrading from Opus to Gemini 2.5 Pro is a capability loss; switching Opus providers is not.
- **Provider quotas are independent:** Anthropic rate limit doesn't affect Antigravity's Claude quota. Cross-provider is essentially free retry capacity.
- **User expectation:** When Tony spawns a PR reviewer with `opus`, he expects Opus-quality output. Getting Gemini 2.5 Pro instead is a surprise; getting Opus via Antigravity is seamless.

**Risk flagged — Medium chain anomaly:** The PRD's Medium chain puts `anthropic/claude-haiku-4-5` (Fast tier!) as the second entry, before `google-antigravity/claude-sonnet-4-5` (same tier). This violates the stated philosophy.

**Recommendation — fix the Medium chain to:**
```
anthropic/claude-sonnet-4-6           ← primary
→ google-antigravity/claude-sonnet-4-5  ← same capability, different provider
→ google-gemini-cli/gemini-3-flash-preview  ← comparable Medium model
→ google-gemini-cli/gemini-2.5-flash   ← tier downgrade (acceptable last resort)
→ anthropic/claude-haiku-4-5           ← final fallback (fast but reliable)
```

This keeps Haiku as the last resort, not the second choice. The existing architecture doc (`docs/subagent-architecture.md`) has the same ordering error — both should be corrected together.

---

### AD-2: Config Schema Design

**Decision:** Use the existing `model` + `subagents.model` two-level schema. No custom tier abstractions.

**Validated schema:**

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
        "google-antigravity/claude-sonnet-4-5",
        "google-gemini-cli/gemini-3-flash-preview",
        "google-gemini-cli/gemini-2.5-flash",
        "anthropic/claude-haiku-4-5"
      ]
    }
  }
}
```

**What this means:**
- `model` = main session (Tony's direct chat). Heavy tier.
- `subagents.model` = default for all spawned agents. Medium tier.
- Per-spawn overrides via `sessions_spawn({ model: 'opus' })` replace primary only; fallbacks come from config.

**What we're NOT doing:** Named tier definitions in config (e.g., `tiers.heavy`, `tiers.medium`). OpenClaw doesn't support that. Tiers are a conceptual layer we enforce through documentation + spawn conventions, not config structure.

**FR-4 validation:** The PRD's proposed schema is correct but has the Medium chain ordering issue noted in AD-1. Corrected version above.

---

### AD-3: Per-Spawn Override Mechanism

**Decision:** `model` parameter on `sessions_spawn()` selects primary model. Fallbacks are always inherited from `subagents.model.fallbacks` in config.

**How persona overrides work:**

| Spawn call | Effective primary | Fallback chain |
|---|---|---|
| `sessions_spawn({ model: 'opus' })` | `anthropic/claude-opus-4-6` | Config's `subagents.model.fallbacks` |
| `sessions_spawn({ model: 'gemini' })` | `google-gemini-cli/gemini-2.5-pro` | Config's `subagents.model.fallbacks` |
| `sessions_spawn({})` (no model) | Config's `subagents.model.primary` (Sonnet 4.6) | Config's `subagents.model.fallbacks` |

**Gap identified:** When a PR Reviewer gets `opus` as primary but falls back to the subagent chain, it hits Antigravity Sonnet and Gemini Flash — a tier downgrade. Ideally, an `opus` override would fall back to Antigravity Opus → Gemini Pro first. But OpenClaw doesn't support per-model-alias fallback chains.

**Mitigation:** This is acceptable for v1. The fallback scenario means the Anthropic Opus endpoint is down AND the primary just failed — at that point, getting Sonnet or Gemini Flash is better than failing entirely. The alternative (no fallback at all for overrides) is worse.

**Recommendation:** Document this as a known limitation. If OpenClaw ever supports named fallback chains, refactor to tier-specific chains.

---

### AD-4: Observability Approach (v1)

**Decision:** Log-based observability with structured memory notes. No dashboards, no metrics infrastructure.

**Implementation:**

1. **Subagent spawn logging:** When main agent spawns a subagent, log to daily memory file:
   ```
   [09:15] Spawned pr-review-CRU-45 | model: opus | label: pr-review-CRU-45
   ```

2. **Subagent self-reporting:** Each subagent's final message (auto-reported to main) should include which model it believes it's running as. This is implicit — the agent's response style and capabilities indicate the model, but there's no programmatic way for the agent to know its own model ID.

3. **Fallback detection:** OpenClaw logs fallback events internally. For v1, review gateway logs (`openclaw gateway logs`) when investigating issues.

4. **Weekly review pattern:** Tony reviews `memory/` files for the week, spots patterns:
   - Are certain agents always falling back? (Provider quota issue)
   - Are Fast-tier tasks getting routed to Heavy? (Misconfigured persona)

**What we're NOT doing (v1):**
- No `model_used` field in spawn response (OpenClaw doesn't expose this)
- No automated cost tracking
- No fallback event alerting

**Gap:** There is no reliable way for the spawning agent to know which model the subagent actually ran on after fallback. This is an OpenClaw platform limitation. For v1, we infer from behavior and gateway logs.

---

### AD-5: Thinking Model Usage

**Decision:** `claude-opus-4-6-thinking` is NOT a separate tier. It's the same Heavy tier with extended thinking enabled. Use it as a manual override for specific high-stakes decisions, not as a default.

**When to use thinking mode:**
- Architecture decisions with multiple competing tradeoffs
- Debugging complex, multi-file issues where root cause is unclear
- Security reviews or sensitive code analysis
- When a regular Opus attempt produced a wrong or shallow answer

**When NOT to use:**
- PR reviews (thorough but not ambiguous — Opus standard is sufficient)
- Planning artifacts (structured output, not deep reasoning)
- Any task where latency matters (thinking adds 10-30s)

**Config placement:** Include in the Heavy chain but AFTER standard Opus, not as primary:
```
anthropic/claude-opus-4-6
→ google-antigravity/claude-opus-4-6
→ google-antigravity/claude-opus-4-6-thinking  ← only reached if both Opus endpoints fail
→ google-gemini-cli/gemini-3-pro-preview
```

**Manual override:** For deliberate thinking-mode usage, spawn with:
```javascript
sessions_spawn({ model: 'google-antigravity/claude-opus-4-6-thinking' })
```

**Rationale for NOT making it default Heavy primary:**
- Higher latency (extended thinking adds processing time)
- Higher token usage (thinking tokens count against quota)
- Standard Opus already has strong reasoning for typical tasks
- Thinking mode is a precision tool, not a blunt instrument

---

## 3. Resolved Architecture

### Config (final recommended `openclaw.json` routing section)

```json
{
  "model": {
    "primary": "anthropic/claude-opus-4-6",
    "fallbacks": [
      "google-antigravity/claude-opus-4-6",
      "google-antigravity/claude-opus-4-6-thinking",
      "google-gemini-cli/gemini-3-pro-preview",
      "google-gemini-cli/gemini-2.5-pro",
      "google-gemini-cli/gemini-2.5-flash"
    ]
  },
  "subagents": {
    "model": {
      "primary": "anthropic/claude-sonnet-4-6",
      "fallbacks": [
        "google-antigravity/claude-sonnet-4-5",
        "google-gemini-cli/gemini-3-flash-preview",
        "google-gemini-cli/gemini-2.5-flash",
        "anthropic/claude-haiku-4-5"
      ]
    }
  }
}
```

### Persona → Model Mapping (final)

| Persona | Label | Tier | `model` param | Primary resolves to |
|---|---|---|---|---|
| Main session | — | Heavy | *(config)* | `anthropic/claude-opus-4-6` |
| Dev Agent | `dev-{ID}` | Medium | *(default)* | `anthropic/claude-sonnet-4-6` |
| PR Reviewer | `pr-review-{ID}` | Heavy | `opus` | `anthropic/claude-opus-4-6` |
| Fix Agent | `fix-{ID}` | Medium | *(default)* | `anthropic/claude-sonnet-4-6` |
| Planning Agent | `plan-{PHASE}-{ID}` | Heavy | `opus` | `anthropic/claude-opus-4-6` |
| Research Agent | `research-{TOPIC}` | Medium | `gemini` | `google-gemini-cli/gemini-2.5-pro` |
| QA Agent | `qa-{ID}` | Medium | *(default)* | `anthropic/claude-sonnet-4-6` |

### Decision Tree (unchanged from PRD — validated)

```
1. Quality gate (PR review)? → Heavy: opus
2. Planning/architecture?    → Heavy: opus  
3. Research + large context? → Medium: gemini override
4. Standard dev/fix/QA?      → Medium: default subagent chain
5. Simple/bulk task?          → Fast: haiku or flash override
6. Main session?              → Heavy: opus (config-level)
```

---

## 4. Gaps & Risks in the PRD

| # | Issue | Severity | Recommendation |
|---|---|---|---|
| 1 | **Medium chain ordering wrong** — Haiku (Fast) placed 2nd instead of after Antigravity Sonnet | High | Fix per AD-1. Both PRD and existing arch doc have this error. |
| 2 | **No per-override fallback chains** — `opus` override falls back to Medium chain | Medium | Accept for v1. Document as known limitation. |
| 3 | **No model introspection** — Can't verify which model a subagent actually used | Medium | Accept for v1. Use gateway logs for debugging. |
| 4 | **Gemini 3 preview stability unknown** — Preview models may break or be removed | Medium | Place after stable models in chains. Monitor Google announcements. |
| 5 | **Sonnet 4.6 availability unconfirmed** — PRD assumes it exists in OpenClaw config | High | Verify model exists before deploying. Fall back to Sonnet 4.5 if not. |
| 6 | **Cost metrics are aspirational** — "≤60% of all-Opus baseline" has no measurement mechanism in v1 | Low | Accept. Rough manual estimates are fine for v1. |
| 7 | **Fast tier underspecified** — No persona currently defaults to Fast; it's only used via manual override | Low | Acceptable. Fast tier exists for ad-hoc use. Will get personas in v2. |

---

## 5. Implementation Notes

1. **First step:** Verify all model IDs are valid in OpenClaw. Run `openclaw config` or check provider endpoints. Invalid model IDs in fallback chains cause silent failures.

2. **Testing fallback:** There's no way to simulate rate limits. Test by temporarily setting primary to an invalid model ID and verifying the fallback engages.

3. **Gateway restart:** Required after any `openclaw.json` change. Check `subagents list` before restarting (per AGENTS.md protocol).

4. **Rollback:** Keep a copy of current `openclaw.json` before applying changes. If new config causes issues, restore and restart.

---

## 6. Approval

- [ ] PRD author (John/PM) confirms gap fixes
- [ ] Tony approves config schema
- [ ] Config deployed and tested
