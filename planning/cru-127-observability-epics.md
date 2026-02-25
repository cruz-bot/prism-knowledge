# CRU-127 — Epics: Observability Workflows & Operational Playbook (Langfuse)

**Issue:** CRU-127
**PRD:** `_bmad-output/planning-artifacts/cru-127-observability-playbook-prd.md`
**Date:** 2026-02-22
**Depends on:** CRU-126 (Done), CRU-123 (Done)

---

## Epic 1: Trace Schema Discovery & Validation

**Goal:** Understand what OpenClaw actually sends to Langfuse before building anything on assumptions.

| Story | Title | Complexity |
|-------|-------|------------|
| 1.1 | Trace Schema Inspection & Open Question Resolution | M |

---

## Epic 2: Observability Playbook & Cost Configuration

**Goal:** Create the operational playbook and configure Langfuse for cost tracking and saved views.

| Story | Title | Complexity |
|-------|-------|------------|
| 2.1 | Model Cost Configuration in Langfuse | M |
| 2.2 | Observability Playbook Document | L |
| 2.3 | Langfuse Saved Views & Filters | S |

---

## Epic 3: Workspace Integration & Weekly Review

**Goal:** Wire observability into existing workflows (AGENTS.md, TOOLS.md, subagent-architecture.md, weekly review).

| Story | Title | Complexity |
|-------|-------|------------|
| 3.1 | Workspace Documentation Updates | M |
| 3.2 | Weekly Review Observability Checklist | S |

---

## Dependency Graph

```
1.1 (Schema Discovery)
 ├── 2.1 (Cost Config)
 ├── 2.2 (Playbook) ← also depends on 2.1
 ├── 2.3 (Saved Views)
 └── 3.1 (Workspace Docs)
      └── 3.2 (Weekly Review) ← also depends on 2.2
```

**Total: 6 stories** | S: 2, M: 3, L: 1
