# CruzBot Evolution Roadmap — PRD

**Date:** 2026-02-23  
**Role:** Product Manager  
**Canon Phase:** PM  
**Input:** [Analyst Brief](cruzbot-evolution-analyst-brief.md)  
**Project:** CruzBot 2.0

---

## 1. Problem Statement

CruzBot averages **2.25 failures/day** across 8 days of operation (18 total Wolverine lessons). The failures cluster into two dominant patterns:

1. **Process erosion (30%)** — Canon skipping for "small" tasks, implementing before tracking, committing to main. The same lesson ("always follow Canon") has been "learned" 3+ times and keeps recurring. Behavioral discipline alone is insufficient.
2. **Autonomy miscalibration (25%)** — Oscillating between asking permission when it's not needed and going idle when it should be acting. Heartbeat-based work continuation is too coarse, creating multi-hour gaps.

Secondary issues include context/memory loss across sessions, subagent quality failures from missing shared context, and observability gaps.

**Core insight:** The top two failure categories are behavioral. They need structural enforcement (hard gates, automation), not more documentation.

---

## 2. Goals & Success Metrics

| Goal | Metric | Current | Target |
|------|--------|---------|--------|
| Reduce process failures | Canon violations per week | ~5/week | <1/week |
| Reduce idle time | Hours idle between completable work | ~2h gaps | <15min (event-driven) |
| Improve memory continuity | Sessions starting with stale/missing context | ~50% | <10% |
| Reduce subagent quality failures | PRs with shared infrastructure damage | ~1/week | 0 |
| Improve observability | Trace coverage of agent actions | 0% | >80% |

**North star:** CruzBot operates autonomously through Canon phases without human intervention for routine work, with <0.5 failures/day.

---

## 3. Prioritized Epic Breakdown

### Epic 1: Process Guardrails (Quick Wins + Plugin)
**Impact:** 🔴 Critical | **Effort:** S-M | **Ratio:** ★★★★★

Prevents the #1 failure pattern (Canon skipping, untracked work). Combines immediate skill improvements with a plugin-based enforcer.

| ID | Item | Type | Size | Shortcomings |
|----|------|------|------|--------------|
| QW-3 | Canon preflight skill | Skill | S | P1, P2, P4 |
| QW-6 | Linear issue template in Canon classify | Skill update | S | P2 |
| PL-1 | Canon Workflow Enforcer plugin | Plugin | M | P1, P2, P3, P4 |

### Epic 2: Memory & Context (Config + Skill)
**Impact:** 🔴 Critical | **Effort:** S | **Ratio:** ★★★★★

Addresses context loss across sessions. LanceDB is a config change; memory consolidation is a small skill.

| ID | Item | Type | Size | Shortcomings |
|----|------|------|------|--------------|
| QW-1 | Enable LanceDB memory plugin | Config | S | C1, C2 |
| QW-4 | Memory consolidation skill | Skill | S | C1 |
| QW-2 | Evaluate Cognee knowledge graph plugin | Config/eval | S | CRU-83 |

### Epic 3: Subagent Quality & Context
**Impact:** 🟡 Medium | **Effort:** S | **Ratio:** ★★★★

Prevents quality failures from subagents missing shared context. Includes better spawn templates and auto-logging.

| ID | Item | Type | Size | Shortcomings |
|----|------|------|------|--------------|
| QW-5 | Subagent shared infrastructure context | Skill update | S | Q1, C3, Q6 |
| PL-3 | Wolverine Auto-Logger plugin | Plugin | S | C2 |

### Epic 4: Event-Driven Workflow (Plugin)
**Impact:** 🔴 Critical | **Effort:** M-L | **Ratio:** ★★★

Highest architectural impact — replaces heartbeat-based continuation with event-driven flow. Eliminates idle periods. Depends on Epics 1-2 being stable first.

| ID | Item | Type | Size | Shortcomings |
|----|------|------|------|--------------|
| PL-2 | Linear Webhook Bridge plugin | Plugin | M | A3, CRU-81 |
| PL-4 | Session Context Injector plugin | Plugin | M | C1, C3, A3 |
| GW-2 | Event-driven subagent chaining | Gateway ext | M | A3, A4 |

### Epic 5: Observability
**Impact:** 🟡 Medium | **Effort:** M | **Ratio:** ★★

Important for debugging and cost tracking but doesn't prevent failures directly.

| ID | Item | Type | Size | Shortcomings |
|----|------|------|------|--------------|
| IN-1 | OTEL backend (Langfuse) | Infrastructure | M | T6, T7 |

---

## 4. Scope Decisions

### V1 (Epics 1-3): Ship in 2 weeks
- Canon preflight skill + Linear issue template
- LanceDB memory enabled
- Memory consolidation skill
- Subagent context improvements
- Wolverine auto-logger plugin
- Canon Workflow Enforcer plugin

### V2 (Epic 4): Ship in 4-6 weeks
- Linear Webhook Bridge
- Session Context Injector
- Event-driven subagent chaining

### Deferred
- **Cognee knowledge graph** — evaluate after LanceDB proves value; don't stack memory changes
- **OTEL/Langfuse** — nice to have, pursue when V1 is stable
- **CRU-84 (Dashboard)** — extend Prism UI when specific visibility needs arise
- **CRU-85 (VS Code Extension)** — cancel. Telegram + webchat workflow works. No clear value.
- **GW-1 (Subagent env consistency)** — investigate as config fix first, only fork if needed

---

## 5. Key Decisions

### Legacy Epic Disposition

| Epic | Decision | Rationale |
|------|----------|-----------|
| **CRU-81** (Event-Driven Integrations) | **Reimagine as PL-2** | Build as OpenClaw plugin, not custom gateway code. Linear webhooks → auto-spawn agents. |
| **CRU-82** (Vector Memory / Qdrant) | **Cancel — use LanceDB** | Built-in LanceDB plugin does vector memory with auto-recall/capture. Zero infrastructure needed. |
| **CRU-83** (Knowledge Graph / Neo4j) | **Defer — evaluate Cognee first** | Try the plugin before building custom Neo4j infrastructure. Likely sufficient. |
| **CRU-84** (Web Dashboard) | **Defer — extend Prism** | CRU-145 already built a web UI. Extend it rather than building standalone. |
| **CRU-85** (VS Code Extension) | **Cancel** | No clear value given current workflow. Copilot Proxy exists if needed. |

### Implementation Approach Per Item

| Item | Approach | Gateway Fork? |
|------|----------|---------------|
| LanceDB memory | Config change (`openclaw.json`) | No |
| Cognee evaluation | Plugin install + config | No |
| Canon preflight | New skill (SKILL.md) | No |
| Linear issue template | Skill update | No |
| Memory consolidation | New skill | No |
| Subagent context | Skill update (dev-discipline) | No |
| Canon Enforcer | Custom plugin (`before_agent_start` hook) | No — plugin API |
| Wolverine Auto-Logger | Custom plugin (lifecycle hooks) | No — plugin API |
| Linear Webhook Bridge | Custom plugin (HTTP handler + background service) | No — plugin API |
| Session Context Injector | Custom plugin (`before_agent_start` hook) | No — plugin API |
| Event-driven chaining | Gateway extension | **Yes** — fork change |
| OTEL backend | Docker deploy + config | No |

**Key insight:** Only 1 of 12 items requires a gateway fork. The plugin API handles almost everything.

---

## 6. Dependencies & Ordering

```
Week 1 (parallel):
  ├── QW-1: Enable LanceDB (no deps)
  ├── QW-3: Canon preflight skill (no deps)
  ├── QW-5: Subagent context improvements (no deps)
  └── QW-6: Linear issue template (no deps)

Week 2:
  ├── QW-4: Memory consolidation skill (after QW-1 — needs LanceDB working)
  ├── PL-3: Wolverine Auto-Logger plugin (no deps)
  └── PL-1: Canon Enforcer plugin (after QW-3 — skill validates approach)

Week 3-4:
  ├── QW-2: Cognee evaluation (after QW-1 settled)
  └── PL-4: Session Context Injector (after QW-1 + PL-3)

Week 4-6:
  ├── PL-2: Linear Webhook Bridge (after PL-1 — enforcer validates plugin pattern)
  └── GW-2: Event-driven chaining (after PL-2 — needs webhook bridge)
```

**Critical path:** QW-1 → QW-4 → PL-4 (memory stack)  
**Critical path:** QW-3 → PL-1 → PL-2 → GW-2 (workflow automation stack)

---

## 7. Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| LanceDB memory conflicts with manual MEMORY.md workflow | Medium | Medium | Run both in parallel for 1 week, compare quality. Deprecate manual only when LanceDB proves reliable. |
| Canon Enforcer too restrictive, blocks legitimate quick fixes | Medium | High | Start with "warn" mode (log + remind), graduate to "block" mode after tuning. |
| Plugin API insufficient for enforcer hooks | Low | High | Analyst confirmed `before_agent_start` hook exists. Test with minimal plugin first. |
| Linear webhook reliability / security | Medium | Medium | Validate webhook signatures, implement retry logic, fall back to polling. |
| Gateway fork divergence makes upgrades painful | Medium | High | Only 1 item needs fork. Isolate changes, upstream if possible. |
| Cognee plugin immature or abandoned | Medium | Low | It's a deferred evaluation — no investment until tested. |
| Concurrent work on CruzBot config while CruzBot is running | High | Medium | Schedule config changes during low-activity windows. Test in staging session first. |

---

## Appendix A: Linear-Ready Issue Definitions

All issues target **Project: CruzBot 2.0** (`f549908c-3906-4a3d-82dc-29b4b82765a3`)

---

### Epic 1: Process Guardrails

#### CRU-XXX: Create Canon Preflight Skill
**Priority:** Urgent  
**Description:**  
Create a new skill (`skills/canon-preflight/SKILL.md`) that activates at session start and before any work begins. The skill should:
- Check if the agent has an active Linear issue for current work
- Remind of Canon classification requirement (Full/Lightweight/Off-Canon)
- Output the classification decision before any implementation begins
- Include explicit "STOP: Create Linear issue first" gate for new work

This addresses Wolverine lessons P1, P2, P4 — Canon skipping for "small" tasks has occurred 7+ times in 8 days.

**Acceptance Criteria:**
- Skill file exists and is referenced in agent context
- Session start includes Canon classification prompt
- New work cannot begin without issue reference

---

#### CRU-XXX: Add Linear Issue Template to Canon Classify
**Priority:** Urgent  
**Description:**  
Update the existing Canon classification flow to output a ready-to-execute Linear issue creation command as step 1. When classifying work as any tier (Full/Lightweight/Off-Canon), the first output should be a `node scripts/linear-create-issue.js` command with pre-filled title and description.

This makes it structurally harder to skip issue creation (P2).

**Acceptance Criteria:**
- Canon classification outputs Linear issue creation command
- Command includes suggested title, description, and priority

---

#### CRU-XXX: Build Canon Workflow Enforcer Plugin
**Priority:** High  
**Description:**  
Build a custom OpenClaw plugin that enforces Canon workflow gates programmatically. The plugin should:
1. Hook `before_agent_start` to check if active work has a Linear issue
2. In "warn" mode (default): log warning + inject reminder into agent context
3. In "block" mode (future): prevent agent turn if no issue context found
4. Track Canon phase transitions and verify correct ordering
5. Register as background service to monitor workflow compliance

This is the structural solution to process erosion — behavioral discipline has failed 3+ times.

**Plugin location:** `plugins/canon-enforcer/`  
**Config:** `openclaw.plugin.json` manifest with warn/block mode toggle

**Acceptance Criteria:**
- Plugin loads successfully in OpenClaw
- `before_agent_start` hook fires and checks for Linear issue context
- Warn mode logs violations to daily memory file
- Plugin has tests

---

### Epic 2: Memory & Context

#### CRU-XXX: Enable LanceDB Memory Plugin
**Priority:** Urgent  
**Description:**  
Enable the built-in LanceDB memory plugin by setting `plugins.slots.memory = "memory-lancedb"` in `openclaw.json`. This replaces our manual MEMORY.md maintenance with vector-based auto-recall and capture.

**Steps:**
1. Update `openclaw.json` to set memory slot to `memory-lancedb`
2. Run one session to verify memory capture and recall work
3. Keep manual MEMORY.md as fallback for 1 week
4. Document any differences in `memory/` daily logs

This addresses C1 (mental notes don't survive sessions) and supersedes CRU-82 (Qdrant).

**Acceptance Criteria:**
- LanceDB plugin active and capturing memories
- Auto-recall returns relevant context at session start
- No regression in session continuity quality

---

#### CRU-XXX: Create Memory Consolidation Skill
**Priority:** High  
**Description:**  
Create a skill (`skills/memory-consolidation/SKILL.md`) that summarizes daily log files and updates long-term memory. Can be invoked via `/consolidate` or as part of end-of-day routine.

**The skill should:**
- Read today's `memory/YYYY-MM-DD.md`
- Extract key decisions, lessons, and state changes
- Update MEMORY.md with consolidated entries
- Archive verbose daily details
- Work alongside LanceDB (complementary, not competing)

**Acceptance Criteria:**
- Skill file exists with clear instructions
- Running consolidation produces meaningful MEMORY.md updates
- Daily logs remain intact (non-destructive)

---

#### CRU-XXX: Evaluate Cognee Knowledge Graph Plugin
**Priority:** Normal (Deferred to after V1)  
**Description:**  
Evaluate the Cognee plugin for OpenClaw as a potential replacement for CRU-83 (Neo4j Knowledge Graph). Install, configure, and test with CruzBot's workspace to determine if it provides sufficient relationship tracking and entity extraction.

**Evaluation criteria:**
- Can it track relationships between Linear issues, files, and decisions?
- Does it integrate via `before_agent_start` hook?
- Is the plugin actively maintained?
- Does it add meaningful context beyond LanceDB vector search?

**Acceptance Criteria:**
- Written evaluation with recommendation (adopt/defer/reject)
- If adopted: configuration documented and tested

---

### Epic 3: Subagent Quality & Context

#### CRU-XXX: Improve Subagent Shared Infrastructure Context
**Priority:** High  
**Description:**  
Update the `dev-discipline` skill to include an explicit "shared infrastructure" section. Each repo should have a `.shared-infra.md` file (or equivalent) listing:
- Protected files that must not be reduced/removed (e.g., `jest.setup.ts`)
- Shared configuration files and their consumers
- Cross-cutting concerns (test utilities, mock factories, etc.)

Subagent spawn instructions should reference this file. This prevents incidents like CRU-108 (jest.setup.ts stripped from 300 to 3 lines).

**Acceptance Criteria:**
- dev-discipline skill includes shared infrastructure check
- Prism repo has `.shared-infra.md` listing protected files
- Subagent spawn template references the file

---

#### CRU-XXX: Build Wolverine Auto-Logger Plugin
**Priority:** High  
**Description:**  
Build an OpenClaw plugin that hooks into agent lifecycle events to capture key outputs, ensuring post-mortem data survives subagent termination.

**The plugin should:**
1. Hook subagent completion/termination events
2. Write key outputs (final message, errors, tool calls summary) to `logs/subagent-{id}-{timestamp}.md`
3. Tag entries with Linear issue ID if available
4. Retain last 50 subagent logs (auto-prune older)

This addresses C2 (subagent logs not retrievable after completion).

**Plugin location:** `plugins/wolverine-logger/`

**Acceptance Criteria:**
- Plugin captures subagent completion data
- Log files are readable and include useful context
- Auto-prune keeps storage bounded

---

### Epic 4: Event-Driven Workflow

#### CRU-XXX: Build Linear Webhook Bridge Plugin
**Priority:** High  
**Description:**  
Build an OpenClaw plugin that receives Linear webhook events and auto-spawns appropriate agents based on issue state changes. This replaces heartbeat-based work continuation with event-driven flow.

**Webhook events to handle:**
- Issue moved to "In Dev" → spawn dev subagent with issue context
- Issue moved to "In QA" → spawn QA subagent
- Issue moved to "Sprint Ready" → notify main session
- Comment added with `@cruzbot` mention → surface to main session

**Plugin registers:**
- HTTP handler at `/webhooks/linear` for incoming events
- Background service for webhook signature validation
- Agent tool for manual event replay

This reimagines CRU-81 (Event-Driven Integrations) as a plugin instead of gateway code.

**Acceptance Criteria:**
- Plugin receives and validates Linear webhooks
- State change → correct subagent spawned with issue context
- Fallback to polling if webhook delivery fails
- Security: webhook signature validation

---

#### CRU-XXX: Build Session Context Injector Plugin
**Priority:** Normal  
**Description:**  
Build an OpenClaw plugin that hooks `before_agent_start` to inject dynamic context into every agent session:
- Active sprint issues (from Linear API)
- Recent Wolverine lessons (last 3 days)
- Pending PRs requiring review
- Current branch state of active repos

This replaces manual context gathering at session start.

**Acceptance Criteria:**
- Plugin injects relevant context at session start
- Context is concise (<2000 tokens)
- Configurable sections (enable/disable per context type)

---

#### CRU-XXX: Event-Driven Subagent Chaining (Gateway)
**Priority:** Normal  
**Description:**  
Extend the OpenClaw gateway (fork) to support auto-continuation when subagents complete. When a subagent finishes → emit structured event → parent session can define auto-reactions (e.g., "when dev completes, spawn PR reviewer").

**This is the only item requiring a gateway fork change.**

Depends on PL-2 (Linear Webhook Bridge) being stable first to validate the event-driven pattern.

**Acceptance Criteria:**
- Subagent completion emits structured event
- Parent can define reaction rules (config or plugin)
- Existing push-based announcements still work (backward compatible)

---

### Epic 5: Observability (Deferred)

#### CRU-XXX: Deploy OTEL Backend (Langfuse)
**Priority:** Low  
**Description:**  
Deploy Langfuse via Docker as the OTEL trace backend. Configure the bundled `diagnostics-otel` plugin to export traces. Provides trace visualization, cost tracking, and model attribution.

**Note:** The existing OTEL plugin has a module instance mismatch (T6). This issue may need to be resolved upstream or worked around.

**Acceptance Criteria:**
- Langfuse running and accessible
- Traces flowing from OpenClaw → Langfuse
- Agent actions visible in trace UI

---

## Appendix B: Legacy Epic Cross-Reference

| Legacy Epic | New Item | Status |
|-------------|----------|--------|
| CRU-81 (Event-Driven Integrations) | PL-2: Linear Webhook Bridge | Reimagined as plugin |
| CRU-82 (Vector Memory / Qdrant) | QW-1: Enable LanceDB | Canceled — built-in solution |
| CRU-83 (Knowledge Graph / Neo4j) | QW-2: Evaluate Cognee | Deferred — evaluate plugin first |
| CRU-84 (Web Dashboard) | — | Deferred — extend Prism UI when needed |
| CRU-85 (VS Code Extension) | — | Canceled — no clear value |
