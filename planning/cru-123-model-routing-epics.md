# Epics & Stories — CRU-123: Model Routing Strategy & Subagent Architecture

**Author:** BMAD Scrum Master  
**Date:** 2026-02-22  
**PRD:** `cru-123-model-routing-prd.md`  
**Architecture:** `cru-123-model-routing-architecture.md`  
**Status:** Ready for Dev

---

## Epic 1: Config & Model Registration

**Goal:** Register all 12+ models in `openclaw.json`, configure main session (Heavy) and default subagent (Medium) fallback chains with corrected ordering per AD-1.

| Story | Title | Estimate |
|-------|-------|----------|
| 1.1 | Register missing models in OpenClaw config | Small |
| 1.2 | Configure main session Heavy fallback chain | Small |
| 1.3 | Configure default subagent Medium fallback chain (with AD-1 fix) | Small |

---

## Epic 2: Persona Routing & Documentation

**Goal:** Update all documentation with tier tables, persona mappings, decision tree, and corrected Medium chain ordering.

| Story | Title | Estimate |
|-------|-------|----------|
| 2.1 | Update AGENTS.md model routing section | Small |
| 2.2 | Update docs/subagent-architecture.md with corrected chains | Small |

---

## Epic 3: Validation & Rollout

**Goal:** Verify fallback chains work, per-spawn overrides resolve correctly, and apply config to production gateway.

| Story | Title | Estimate |
|-------|-------|----------|
| 3.1 | Validate fallback chains and per-spawn overrides | Small |
| 3.2 | Apply config to production gateway and verify | Small |

---

## Dependency Graph

```
Epic 1 (Config) → Epic 3 (Validation)
Epic 2 (Docs)   → Epic 3 (Validation)
Epic 1 and Epic 2 are independent — can run in parallel.
```

## Notes

- **Medium chain fix (AD-1):** Haiku must move from 2nd to last position in Medium fallback chain. Affects Stories 1.3, 2.1, and 2.2.
- **All work is config/docs** — no application code changes.
- **Gateway restart required** after config changes (Story 3.2). Check for running subagents first.
