# CRU-136: CruzBot OS — Codify Canon, Neo, Wolverine, Evolve

## Analyst Brief

**Date:** 2026-02-22  
**Analyst:** Mary (BMAD Analyst Agent)  
**Status:** Complete  
**Scope:** Requirements analysis for mechanizing CruzBot's four operating frameworks

---

## 1. Executive Summary

CruzBot operates under four conceptual frameworks (Canon, Neo, Wolverine, Evolve) defined as prose in `AGENTS.md`. Today, compliance is entirely voluntary — the agent "remembers" the rules but nothing enforces them. This brief analyzes what exists, what's missing, and what's needed to turn prose into enforceable, scriptable systems.

**Bottom line:** The infrastructure is 30% there. Existing scripts (Linear API, PR creation, readiness gates, model routing) provide a solid foundation. What's missing is the **enforcement layer** — pre-flight checks, audit logging, decision tracking, and file-change monitoring that make the four frameworks mechanically unavoidable rather than aspirationally remembered.

---

## 2. Current State Analysis

### 2.1 Canon (Workflow Enforcement)

**What exists today:**

| Asset | Location | Function |
|---|---|---|
| `implementation-readiness-gate.js` | `workspace/scripts/` | Validates planning artifacts before SM phase |
| `bmad-orchestrator.js` | `workspace/scripts/` | End-to-end BMAD→Linear→Dev orchestration |
| `start-dev-session.js` | `workspace/scripts/` | Dev startup: fetch issue, find story, inject standards |
| `create-github-pr.js` | `workspace/scripts/` | PR creation with Linear state update |
| `create-stories-from-files.js` | `workspace/scripts/` | Populates Linear from story files |
| `bmad-to-linear.js` / `v2` | `workspace/scripts/` | Syncs BMAD artifacts to Linear |
| BMAD v6 workflows | `workspace/_bmad/bmm/workflows/` | Full workflow definitions (analysis, planning, solutioning, implementation) |
| Agent-OS standards | `workspace/agent-os/standards/` | `prism-standards.md` + index |
| `cruzbot-core` workflow engine | `workspace/cruzbot-core/` | xstate-based engine with ModelRouter + AgentSpawner (TypeScript, not yet integrated) |

**What's missing:**

- **Phase transition gates:** Only one gate exists (implementation-readiness). No gates for: Analyst→PM, PM→Architect, Architect→SM, Dev→QA, QA→PR, PR→Merge.
- **Linear state enforcement:** Scripts *update* Linear states but don't *validate* correct transitions. Nothing prevents jumping from Backlog → In Dev, skipping Analyst/PM/Architect.
- **Workflow classification:** AGENTS.md defines Full Canon / Lightweight Canon / Off-Canon paths, but no script determines which path an issue should follow.
- **Session-level enforcement:** No pre-flight check runs automatically when a dev session starts (the `start-dev-session.js` exists but isn't mandated).

### 2.2 Neo (Autonomy/Decision Tracking)

**What exists today:**
- Prose rules in AGENTS.md defining "decide autonomously" vs "ask permission" categories
- Memory files (`memory/YYYY-MM-DD.md`, `MEMORY.md`) where decisions can be recorded manually

**What's missing:**
- **Decision log:** No structured format or dedicated file for autonomous decisions
- **Decision audit trail:** No script that creates a timestamped record with decision + reasoning + alternatives + alignment
- **Permission gate:** No programmatic check for "destructive action" detection before execution
- **Escalation mechanism:** No automated way to flag uncertain decisions for human review

### 2.3 Wolverine (Self-Healing/Continuous Improvement)

**What exists today:**
- `create-ops-issue.js` — Creates Ops issues in Linear (title, description, priority)
- `AGENTS.md` prose describing the reinforcement learning loop
- Manual process: agent notices problem → creates ops issue → weekly triage

**What's missing:**
- **Failure capture automation:** No script that detects failures (CI failures, script errors, process breakdowns) and auto-creates Ops issues
- **Structured learning log:** No machine-readable format for lessons learned (just prose in memory files)
- **Pattern detection:** No analysis of recurring failures or process friction
- **P1 enforcement:** AGENTS.md says "workarounds NOT permitted" for P1, but nothing enforces this

### 2.4 Evolve (Self-Evolution Tracking)

**What exists today:**
- Nothing. Pure prose in AGENTS.md.
- Core DNA files exist (`AGENTS.md`, `SOUL.md`, `IDENTITY.md`) but changes are untracked beyond git history.

**What's missing:**
- **File change detection:** No watcher or pre-commit hook for DNA files
- **Evolution log:** No structured changelog for core DNA modifications
- **Change justification requirement:** No enforcement that DNA file changes include rationale
- **Rollback capability:** Git provides this, but no script automates review/rollback of DNA changes

---

## 3. Requirements

### 3.1 Canon Enforcement

| ID | Requirement | Priority |
|---|---|---|
| C-1 | **Workflow classifier:** Given an issue, determine Full Canon / Lightweight / Off-Canon based on signals (file count, labels, project type) | P1 |
| C-2 | **Phase transition gates:** Validate prerequisites before allowing Linear state transitions (e.g., can't move to "In Dev" without story files + standards injection) | P1 |
| C-3 | **Pre-flight check:** Script that runs at dev session start, validating: correct Linear state, story file exists, standards injected, branch created | P1 |
| C-4 | **Post-flight check:** Script that runs after dev completion, validating: tests pass, PR created, Linear state updated | P2 |
| C-5 | **Canon audit report:** Script that analyzes a completed issue's history and reports which Canon phases were followed vs skipped | P2 |
| C-6 | **Integrate `cruzbot-core` workflow engine:** Connect the xstate-based engine to actual OpenClaw session spawning for phase orchestration | P3 |

### 3.2 Neo Enforcement

| ID | Requirement | Priority |
|---|---|---|
| N-1 | **Decision logger:** Script/function that appends structured entries to `logs/decisions.jsonl` with: timestamp, decision, reasoning, alternatives, category (autonomous/escalated), alignment | P1 |
| N-2 | **Decision categories:** Classify decisions into the AGENTS.md categories (dev sequence, technical approach, model routing, etc.) for filtering | P2 |
| N-3 | **Destructive action detector:** Before executing rm/force-push/delete operations, check against a pattern list and require explicit confirmation | P2 |
| N-4 | **Weekly decision summary:** Script that generates a digest of autonomous decisions for review | P3 |

### 3.3 Wolverine Enforcement

| ID | Requirement | Priority |
|---|---|---|
| W-1 | **Error capture wrapper:** A try/catch wrapper for script execution that auto-creates Ops issues on failure with context (script name, args, error, stack trace) | P1 |
| W-2 | **Structured lesson log:** `logs/lessons.jsonl` with: timestamp, trigger event, root cause, lesson, files updated, ops issue ID | P1 |
| W-3 | **P1 ops enforcement:** Script that checks open P1 ops issues and blocks new feature work until they're resolved (or explicitly overridden) | P2 |
| W-4 | **Recurring failure detector:** Analyze `lessons.jsonl` for patterns (same root cause appearing 3+ times) and auto-escalate | P3 |

### 3.4 Evolve Enforcement

| ID | Requirement | Priority |
|---|---|---|
| E-1 | **DNA file watcher:** Script that diffs core DNA files (`AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `TOOLS.md`) against last-known state and logs changes | P1 |
| E-2 | **Evolution changelog:** `logs/evolution.jsonl` with: timestamp, file changed, diff summary, justification, trigger (user request / self-improvement / wolverine lesson) | P1 |
| E-3 | **Change justification gate:** Before committing DNA file changes, require a justification string (can be provided inline, but must exist) | P2 |
| E-4 | **Evolution review digest:** Weekly summary of all DNA changes for human review | P3 |

---

## 4. Constraints

1. **Runtime environment:** Windows (PowerShell), Node.js v22. Scripts must work without Unix tooling.
2. **No persistent daemon:** OpenClaw agents are ephemeral sessions. Enforcement must work as pre/post scripts called by the agent, not as background services.
3. **Linear API is the source of truth** for workflow states. All Canon enforcement must read/write Linear.
4. **Git is the source of truth** for DNA file changes. Evolve must use git diff, not filesystem watchers.
5. **Backward compatibility:** Existing scripts (`create-ops-issue.js`, `create-github-pr.js`, etc.) must continue working. New scripts should extend, not replace.
6. **Agent context limits:** Enforcement scripts must produce concise output (not walls of text) so agents can consume results within context windows.
7. **No external dependencies beyond what's already used:** `https` (native), `fs`, `path`, `child_process`. The `cruzbot-core` package adds `xstate` and `ioredis` but those are optional/future.
8. **OpenClaw API:** Agent spawning uses `sessions_spawn` — scripts can't directly call this, but the main agent can. Scripts should output structured JSON that the agent interprets.

---

## 5. Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| **Over-enforcement kills velocity** — Too many gates slow the agent down, especially for quick fixes | High | High | Workflow classifier (C-1) must correctly route small tasks to Off-Canon/Lightweight paths with minimal gates |
| **Gate bypass becomes norm** — Agent learns to skip enforcement scripts if they're not mandated at the session level | High | High | Integrate enforcement into `start-dev-session.js` and PR creation flow so they're unavoidable |
| **Log bloat** — JSONL logs grow unbounded | Medium | Low | Add rotation/archival (keep 30 days, archive older) |
| **False positives in destructive action detection** — Legitimate operations blocked | Medium | Medium | Allowlist common safe operations; make override easy with logged justification |
| **cruzbot-core integration complexity** — xstate engine is TypeScript, existing scripts are CommonJS JS | Medium | Medium | Phase 1 uses simple JS scripts; integrate cruzbot-core in Phase 2 after scripts prove the model |
| **Linear API rate limits** — Multiple gate checks per session hit the API | Low | Medium | Cache issue state locally during a session; only re-fetch on gate checks |

---

## 6. Recommendations

### 6.1 Phased Approach

**Phase 1: Foundation (MVP)** — 4-6 stories
- Canon: Workflow classifier (C-1), phase transition validator (C-2), enhanced pre-flight (C-3)
- Neo: Decision logger (N-1)
- Wolverine: Error capture wrapper (W-1), structured lesson log (W-2)
- Evolve: DNA file watcher + changelog (E-1, E-2)

**Phase 2: Enforcement** — 3-4 stories
- Canon: Post-flight check (C-4), Canon audit (C-5)
- Neo: Destructive action detector (N-3)
- Wolverine: P1 ops enforcement (W-3)
- Evolve: Change justification gate (E-3)

**Phase 3: Intelligence** — 3-4 stories
- Canon: cruzbot-core integration (C-6)
- Neo: Weekly decision summary (N-4), decision categories (N-2)
- Wolverine: Recurring failure detector (W-4)
- Evolve: Evolution review digest (E-4)

### 6.2 Architecture Suggestion

All new scripts should live in `workspace/scripts/os/` (new subdirectory) to separate OS enforcement from existing workflow scripts:

```
scripts/os/
  canon-classify.js       # C-1: Determine workflow path
  canon-gate.js           # C-2: Phase transition validation
  canon-preflight.js      # C-3: Dev session pre-flight
  canon-postflight.js     # C-4: Dev session post-flight
  canon-audit.js          # C-5: Issue compliance audit
  neo-log-decision.js     # N-1: Log a decision
  neo-detect-destructive.js # N-3: Destructive action check
  wolverine-wrap.js       # W-1: Error capture wrapper
  wolverine-log-lesson.js # W-2: Log a lesson
  wolverine-p1-gate.js    # W-3: P1 blocker check
  evolve-diff.js          # E-1: DNA file change detection
  evolve-log.js           # E-2: Evolution changelog entry
```

All scripts should:
- Accept JSON via stdin or CLI args
- Output structured JSON to stdout (for agent consumption)
- Exit 0 (pass) or 1 (fail) for gate scripts
- Be callable independently or composed by the orchestrator

### 6.3 Integration Points

1. **`start-dev-session.js`** → calls `canon-preflight.js` automatically
2. **`create-github-pr.js`** → calls `canon-postflight.js` automatically
3. **Main agent AGENTS.md** → Updated to reference scripts instead of prose rules
4. **Subagent spawn** → Orchestrator calls `canon-classify.js` to determine workflow, then spawns appropriate phase agents
5. **Session end** → `evolve-diff.js` checks if DNA files changed during session

### 6.4 Key Design Decision

**Scripts, not a framework.** Phase 1 should be simple, standalone Node.js scripts — not a TypeScript framework requiring compilation. The `cruzbot-core` xstate engine is valuable but premature for enforcement gates. Build the simple version first, prove the model works, then migrate to the engine in Phase 3.

---

## 7. Open Questions for PM Phase

1. **How strict should Canon enforcement be?** Should gate failures *block* (exit 1, refuse to proceed) or *warn* (log violation, continue)?  
   **Recommendation:** Block for P1 gates (no dev without story), warn for P2 gates (missing post-flight).

2. **Should Neo decisions be logged to Linear comments or just local files?**  
   **Recommendation:** Local JSONL first; optionally add Linear comment for significant decisions.

3. **Should Wolverine auto-create ops issues for ALL script failures, or only recurring ones?**  
   **Recommendation:** Auto-create for all, with dedup (don't create duplicate for same error within 24h).

4. **What constitutes a "DNA file" for Evolve tracking?**  
   **Recommendation:** Start with `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `TOOLS.md`, `USER.md`. Expand based on experience.

5. **Should the workflow classifier use Linear labels/projects or analyze the issue description?**  
   **Recommendation:** Labels first (explicit), with description analysis as fallback.

---

## 8. Success Criteria

- [ ] No dev session starts without pre-flight passing
- [ ] Every autonomous decision is logged with reasoning
- [ ] Script failures auto-generate Ops issues
- [ ] DNA file changes are tracked with justification
- [ ] Canon phase skips are detectable in audit reports
- [ ] Off-Canon work is explicitly classified (not silently non-compliant)
- [ ] Agent velocity is NOT meaningfully reduced for Lightweight/Off-Canon paths

---

*Brief complete. Ready for PM phase to produce PRD and epics.*
