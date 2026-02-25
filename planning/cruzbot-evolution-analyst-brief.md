# CruzBot Self-Audit & Evolution Roadmap — Analyst Brief

**Date:** 2026-02-23  
**Role:** Analyst  
**Scope:** Honest shortcomings analysis, OpenClaw ecosystem research, legacy epic evaluation, evolution roadmap

---

## Part 1: Self-Audit — Honest Shortcomings Analysis

### Summary

CruzBot has 18 permanent Wolverine lessons logged in 8 days of operation. That's **2.25 failures per day**. Many are repeat patterns — process erosion and autonomy hesitation account for over half. The system is learning but slowly, and some lessons have been learned multiple times before sticking.

---

### Process Failures

| # | Shortcoming | Evidence | Severity |
|---|---|---|---|
| P1 | **Skipped Canon for "small" tasks — repeatedly** | CRU-169 (fork maintenance), CRU-120 (archive), CRU-144 (committed to main directly). Three separate incidents of "it's small, skip the process." | 🔴 Critical |
| P2 | **Implemented before creating Linear issue** | Model config changes and OTEL done without tracking. "Quick config changes" treated as not needing issues. | 🔴 Critical |
| P3 | **Stories created as markdown but not as Linear sub-issues** | SM phase creates `.md` files but work was invisible in Linear until explicitly caught. | 🟡 Medium |
| P4 | **PR committed directly to main (CRU-144)** | Bypassed the mandatory PR review gate entirely. | 🔴 Critical |

**Root pattern:** "It's small/quick" is the universal justification for process erosion. This has happened **at least 4 times** in 8 days. The lesson has been "learned" 3 times and keeps recurring.

---

### Technical Failures

| # | Shortcoming | Evidence | Severity |
|---|---|---|---|
| T1 | **Git contention from concurrent agents** | CRU-145 Epic 6: 5 agents fighting over branches, scope bleed, junk file commits. | 🔴 Critical |
| T2 | **Node scripts produce no output without pty:true** | 15+ minutes debugging before discovering the fix. | 🟡 Medium |
| T3 | **Linear API auth format confusion** | Bearer vs raw key format. Subagents get auth errors. Still a known bug with workaround only. | 🟡 Medium |
| T4 | **Workflow state names case-sensitive** | "done" ≠ "Done" caused silent failures in linear-update-issue.js. | 🟢 Low |
| T5 | **Session model overrides persist across restarts** | Config change didn't take effect until manual session reset. | 🟡 Medium |
| T6 | **OTEL plugin doesn't export traces** | Module instance mismatch between gateway core and plugin-sdk. Parked. | 🟡 Medium |
| T7 | **Gemini CLI rate limiting under concurrent load** | 0% Pro / 29% Flash capacity caused failures during multi-agent spawns. | 🟡 Medium |

---

### Context/Memory Failures

| # | Shortcoming | Evidence | Severity |
|---|---|---|---|
| C1 | **"Mental notes" don't survive sessions** | Explicit guidance added to AGENTS.md: "Text > Brain." This was a real problem before being codified. | 🔴 Critical |
| C2 | **Subagent logs not retrievable after completion** | CRU-112: Can't debug failed subagents. Blocks Wolverine's "Analyze" step entirely. Workaround only. | 🟡 Medium |
| C3 | **Subagents lack context on shared infrastructure** | CRU-108: Dev agent stripped jest.setup.ts from 300 lines to 3. Didn't know it was shared. | 🟡 Medium |
| C4 | **Complex multi-file changes too hard for subagents** | CRU-23: 3 consecutive subagent failures on a 9-file change. Context overhead exceeds benefit. | 🟡 Medium |

---

### Autonomy Failures

| # | Shortcoming | Evidence | Severity |
|---|---|---|---|
| A1 | **"Want me to keep going?" at phase transitions** | Repeated 3+ times during CRU-145. Approved plan doesn't need re-approval at every step. | 🔴 Critical |
| A2 | **Stated plan then didn't execute it** | Tony called out: "You said you'd start Epic 2 in 2 minutes, then didn't." Became a DNA-level lesson. | 🔴 Critical |
| A3 | **Went idle for hours between subagent completions** | CRU-128: 2h heartbeat + no work continuation logic = momentum killer. | 🔴 Critical |
| A4 | **Reactive pattern — waiting for user confirmation** | General tendency to wait for prompts instead of executing stated plans. Multiple instances. | 🟡 Medium |

**Root pattern:** CruzBot oscillates between two failure modes: (1) asking permission when it already has it, and (2) not acting when it should. The sweet spot (Neo discipline) has been codified but enforcement is purely behavioral — no guardrails prevent the failure.

---

### Quality Failures

| # | Shortcoming | Evidence | Severity |
|---|---|---|---|
| Q1 | **Dev agent destroyed shared test setup** | jest.setup.ts reduced from 300 to 3 lines. PR review caught it, but it shouldn't have happened. | 🟡 Medium |
| Q2 | **Dual Jest config shipped** | jest.config.ts AND jest.config.cjs in same PR. Tests failed. | 🟢 Low |
| Q3 | **Missing Prisma migration in PR** | Schema changes without migration file. | 🟡 Medium |
| Q4 | **PRD referenced unconfirmed model version** | Sonnet 4.6 existence was assumed, not verified. Architect caught it. | 🟢 Low |
| Q5 | **Test failures accepted and pushed forward** | "Let PR Reviewer decide" — pushed cosmetic test failures rather than fixing them. | 🟡 Medium |
| Q6 | **Subagent context overflow** | fix-CRU-63 hit 135.5k context limit after 3.5min. Task scoping was too broad. | 🟡 Medium |

---

### Failure Frequency Heat Map

| Category | Count | % of Total |
|---|---|---|
| Process failures | 4 unique, 7+ incidents | 30% |
| Technical failures | 7 | 25% |
| Autonomy failures | 4 unique, 10+ incidents | 25% |
| Quality failures | 6 | 15% |
| Context/memory failures | 4 | 5% |

**The biggest systemic risks are process erosion (Canon skipping) and autonomy calibration (hesitation vs. overreach).** These are behavioral, not technical — harder to fix with code.

---

## Part 2: OpenClaw Ecosystem Research

### Plugin System Architecture

OpenClaw has a mature plugin system. Key capabilities:

| Capability | Details |
|---|---|
| **Plugin types** | TypeScript modules loaded via jiti, run in-process with Gateway |
| **Can register** | RPC methods, HTTP handlers, agent tools, CLI commands, background services, skills, auto-reply commands |
| **Lifecycle hooks** | `before_agent_start`, plus events for `/new`, `/reset`, `/stop`, and other lifecycle events |
| **Discovery** | Config paths → workspace extensions → global extensions → bundled extensions |
| **Config** | `openclaw.plugin.json` manifest + JSON Schema validation |
| **Skills bundling** | Plugins can ship their own skills directories |

**Key insight:** Plugins can hook into `before_agent_start` — this is how we could inject dynamic context, enforce workflow gates, or run pre-flight checks.

### Available Official Plugins

- **Memory (Core)** — bundled, enabled by default via `plugins.slots.memory`
- **Memory (LanceDB)** — long-term memory with auto-recall/capture. Set `plugins.slots.memory = "memory-lancedb"`
- **Voice Call** — `@openclaw/voice-call`
- **Diagnostics OTEL** — bundled, for observability
- **Various channel plugins** — Teams, Matrix, Nostr, Zalo
- **Auth plugins** — Google Antigravity, Gemini CLI, Qwen, Copilot Proxy

**Critical discovery: Memory (LanceDB)** — This is a built-in vector memory plugin that does auto-recall and capture. This could replace or supplement our manual MEMORY.md system with zero custom code.

### ClawHub Ecosystem

- **3,286+ skills** available on ClawHub
- CLI: `clawhub install <skill-slug>`, `clawhub search "<query>"`, `clawhub update --all`
- Skills are just folders with SKILL.md + supporting files
- **Security concern:** Researchers found hundreds of malicious skills on ClawHub — vet before installing
- Notable community collections: `VoltAgent/awesome-openclaw-skills` on GitHub

### Relevant Skills/Plugins from Ecosystem

| Skill/Plugin | What it does | Relevance to CruzBot |
|---|---|---|
| **memory-lancedb** (official) | Vector memory with auto-recall/capture | Replaces CRU-82 (Qdrant) goal |
| **Cognee plugin** | Knowledge graph memory for OpenClaw | Replaces CRU-83 (Neo4j) goal |
| **SecureClaw** | OWASP-aligned security auditing | Already have weekly security audit |
| **GitHub skill** (ClawHub) | Managed OAuth, repos/issues/PRs/commits | Could supplement our gh CLI workflow |
| **sec-filing-watcher** | SEC EDGAR monitoring | Not relevant |

### Lessons from Other Frameworks

| Framework | Key Feature | CruzBot Takeaway |
|---|---|---|
| **Letta/MemGPT** | Tiered memory (core/recall/archival) with auto-consolidation. Treats context window as OS memory with paging. | Our flat MEMORY.md + daily files = primitive. Need tiered memory with auto-consolidation. LanceDB plugin may handle this. |
| **CrewAI** | Task guardrails (function-based + LLM-based validation), Flows for event-driven chaining | We need workflow enforcement that isn't purely behavioral. Guardrails at task boundaries would prevent Canon skipping. |
| **CrewAI Flows** | State management, event-driven responsiveness, conditional branching | Our heartbeat-based work continuation is crude. Event-driven flow would be better. |
| **LangChain/LangGraph** | Memory management with tool-based read/write, conversation summarization | Could implement as a skill — summarize daily logs into MEMORY.md automatically. |

---

## Part 3: Legacy Epic Evaluation

### CRU-81: Event-Driven Integrations (BMAD→Linear, Linear→Agent)

| Aspect | Assessment |
|---|---|
| **Still relevant?** | ✅ Yes — this is the #1 workflow gap |
| **Problem it solves** | Manual handoffs between BMAD phases, Linear state changes, and agent spawning. Currently requires human or heartbeat to bridge. |
| **Simpler alternative?** | **Partially.** OpenClaw plugins can register background services and lifecycle hooks. A custom plugin could watch Linear webhook events and auto-spawn agents. But BMAD→Linear flow (SM creates stories → auto-create Linear issues) still needs custom logic. |
| **Recommendation** | **Build as OpenClaw plugin**, not custom gateway code. Plugin can register HTTP handler for Linear webhooks + background service for polling. |
| **Complexity** | M (plugin development, webhook handling, agent spawning logic) |

### CRU-82: Vector Memory & Auto-Indexing (Qdrant)

| Aspect | Assessment |
|---|---|
| **Still relevant?** | ⚠️ **Superseded by built-in Memory (LanceDB) plugin** |
| **Problem it solves** | Searchable memory across sessions, auto-indexing of workspace files |
| **Simpler alternative?** | **Yes.** Set `plugins.slots.memory = "memory-lancedb"` in config. Built-in, no infrastructure needed. |
| **Recommendation** | **Try LanceDB first.** If insufficient (missing auto-indexing of workspace files, or search quality poor), then evaluate Qdrant. Don't build custom when built-in exists. |
| **Complexity** | S (config change) or L (if LanceDB insufficient and Qdrant needed) |

### CRU-83: Knowledge Graph (Neo4j)

| Aspect | Assessment |
|---|---|
| **Still relevant?** | ⚠️ **Likely superseded by Cognee plugin** |
| **Problem it solves** | Relationship tracking between entities (issues, people, decisions, files) |
| **Simpler alternative?** | **Yes.** Cognee plugin provides knowledge graph memory for OpenClaw. Research showed it integrates via plugin API with `before_agent_start` hooks. |
| **Recommendation** | **Evaluate Cognee plugin first.** If it provides relationship tracking + entity extraction, skip Neo4j entirely. If not, the need is real but the priority is lower than memory and workflow. |
| **Complexity** | S (plugin install) or L (custom Neo4j infrastructure) |

### CRU-84: Web Dashboard

| Aspect | Assessment |
|---|---|
| **Still relevant?** | ⚠️ **Partially superseded by CRU-145 (Prism Agentic Runtime)** |
| **Problem it solves** | Visibility into agent state, config, decisions, audit trail |
| **Simpler alternative?** | **Yes.** CRU-145 already built a web UI with config dashboard, audit trail, and escalation queue in Prism. Extending that is cheaper than a standalone dashboard. |
| **Recommendation** | **Extend Prism UI** (CRU-145 output) rather than building separate dashboard. Add CruzBot-specific views as needed. |
| **Complexity** | S-M (extending existing UI) |

### CRU-85: VS Code Extension

| Aspect | Assessment |
|---|---|
| **Still relevant?** | ⚠️ **Low priority given current workflow** |
| **Problem it solves** | IDE integration for agent interaction |
| **Simpler alternative?** | OpenClaw already has a VS Code Copilot Proxy plugin (bundled). CruzBot operates via Telegram + subagents, not IDE. |
| **Recommendation** | **Defer indefinitely.** The Telegram + webchat workflow works. VS Code extension adds complexity without clear value. Revisit if workflow shifts to IDE-centric. |
| **Complexity** | L (custom extension development) |

---

## Part 4: Proposed Evolution Roadmap

### Quick Wins (Skills/Config, No Gateway Modification)

| # | Proposal | Problem Solved | Approach | Size | Shortcoming(s) |
|---|---|---|---|---|---|
| QW-1 | **Enable Memory (LanceDB) plugin** | Context/memory failures (C1), manual MEMORY.md maintenance | Set `plugins.slots.memory = "memory-lancedb"` in openclaw.json | S | C1, C2 |
| QW-2 | **Install Cognee plugin for knowledge graph** | Relationship tracking, entity memory | `openclaw plugins install @cognee/openclaw-plugin` (if available) or manual setup | S | C1, CRU-83 |
| QW-3 | **Create canon-preflight skill** | Process erosion (P1, P2) — Canon skipping | Skill that runs at session start, checks for active Linear issues, reminds of Canon classification requirement. Purely instructional but creates friction. | S | P1, P2, P4 |
| QW-4 | **Create memory-consolidation skill** | Daily log bloat, MEMORY.md staleness | Skill that summarizes daily logs and updates MEMORY.md. Invoked via slash command `/consolidate` or heartbeat. | S | C1 |
| QW-5 | **Improve subagent context injection** | Quality failures from missing context (Q1, C3) | Update dev-discipline skill to include explicit "shared infrastructure" section listing protected files per repo. | S | Q1, C3, Q6 |
| QW-6 | **Add Linear issue template to Canon skill** | P2 — implementing before creating issues | Update canon-classify to output a Linear issue creation command as step 1. Make it impossible to classify without tracking. | S | P2 |

### Plugin Development (Custom OpenClaw Plugins on cruzbot Branch)

| # | Proposal | Problem Solved | Approach | Size | Shortcoming(s) |
|---|---|---|---|---|---|
| PL-1 | **Canon Workflow Enforcer plugin** | Process erosion (P1, P2, P4) | Plugin hooks `before_agent_start` to check if active work has a Linear issue. Registers background service to enforce Canon gates. Could block agent turns that don't have issue context. | M | P1, P2, P4, P3 |
| PL-2 | **Linear Webhook Bridge plugin** | Manual handoffs, idle periods (A3) | Plugin registers HTTP handler for Linear webhook events. When issue state changes → auto-spawn appropriate agent (e.g., "In Dev" → spawn dev agent). Replaces heartbeat-based continuation. | M | A3, CRU-81 |
| PL-3 | **Wolverine Auto-Logger plugin** | Subagent logs not retrievable (C2) | Plugin hooks into agent lifecycle events, captures key outputs to files. Ensures post-mortem data survives subagent termination. | S | C2 |
| PL-4 | **Session Context Injector plugin** | Dynamic context needs | Plugin hooks `before_agent_start` to inject relevant context: active sprint issues, recent Wolverine lessons, pending PRs, current branch state. | M | C1, C3, A3 |

### Gateway Extensions (Fork Modifications)

| # | Proposal | Problem Solved | Approach | Size | Shortcoming(s) |
|---|---|---|---|---|---|
| GW-1 | **Subagent environment consistency** | T3, CRU-117 — env vars not inherited | Ensure subagents reliably inherit all configured env vars. May be fixable via config rather than code. | S-M | T3 |
| GW-2 | **Event-driven subagent chaining** | A3 — idle between phases | When subagent completes → emit event → parent can auto-react. Currently push-based announcements exist but no auto-continuation logic. | M | A3, A4 |

### Infrastructure (New Services)

| # | Proposal | Problem Solved | Approach | Size | Shortcoming(s) |
|---|---|---|---|---|---|
| IN-1 | **OTEL backend (Langfuse/SigNoz)** | Observability gap (T6) | Deploy Langfuse via Docker, configure diagnostics-otel plugin. Provides trace visualization, cost tracking, model attribution. | M | T6, T7 |

---

## Prioritized Implementation Order

### Phase 1: Immediate (This Week)
1. **QW-1** — Enable LanceDB memory (5 min config change)
2. **QW-3** — Canon preflight skill (1-2 hours)
3. **QW-5** — Improve subagent context (1 hour)
4. **QW-6** — Linear issue template in Canon (30 min)

### Phase 2: Near-Term (Next 2 Weeks)
5. **PL-3** — Wolverine auto-logger plugin (1-2 days)
6. **PL-1** — Canon enforcer plugin (2-3 days)
7. **QW-4** — Memory consolidation skill (half day)
8. **QW-2** — Cognee plugin evaluation (half day)

### Phase 3: Medium-Term (Next Month)
9. **PL-2** — Linear webhook bridge (3-5 days)
10. **PL-4** — Session context injector (2-3 days)
11. **IN-1** — OTEL backend (1-2 days)
12. **GW-2** — Event-driven subagent chaining (3-5 days)

### Phase 4: Evaluate & Defer
13. **GW-1** — Subagent env consistency (may be resolved by config)
14. **CRU-84** — Dashboard (extend Prism UI when needed)
15. **CRU-85** — VS Code extension (defer indefinitely)

---

## Key Insights

### 1. The Biggest Problem is Behavioral, Not Technical
Process erosion (Canon skipping) and autonomy calibration (hesitation/overreach) are the top failure modes. These are hard to solve with code alone. The Canon Enforcer plugin (PL-1) is the best technical mitigation — it creates hard gates rather than relying on self-discipline.

### 2. OpenClaw's Plugin System is More Capable Than We've Used
We're running on vanilla OpenClaw with custom skills only. The plugin system supports lifecycle hooks, background services, HTTP handlers, and agent tools. We should be building plugins, not forking the gateway for most of our needs.

### 3. Built-In Memory Plugins May Obsolete 2 Epics
LanceDB (vector memory) and Cognee (knowledge graph) could replace CRU-82 and CRU-83 entirely. Try the built-in solutions before building custom infrastructure.

### 4. Event-Driven > Heartbeat-Based
The current heartbeat model (check every 30min, look for work) is fundamentally reactive. Event-driven architecture (Linear webhook → auto-spawn agent) would eliminate idle periods and manual handoffs. This is the highest-impact architectural change.

### 5. Subagent Task Scoping is a Solved Problem — Enforcement Isn't
The lesson "one issue per agent" is documented. But agents still get spawned with too much context or too broad a scope. The fix is better spawn templates and context budgets, not more documentation.

---

## Appendix: All 18 Wolverine Lessons Categorized

| Lesson | Category | Times "Learned" |
|---|---|---|
| Even ops tasks get classified (Canon) | Process | 3+ |
| Session model overrides persist | Technical | 1 |
| Linear API keys: no Bearer prefix | Technical | 1 |
| secrets/linear-ids.json is source of truth | Technical | 1 |
| State names are case-sensitive | Technical | 1 |
| Don't be reactive — execute stated plans | Autonomy | 3+ |
| webchat can't use message tool | Technical | 1 |
| Heartbeats must drive work | Autonomy | 1 |
| Stories must become Linear sub-issues | Process | 1 |
| Always follow Canon, even for quick tasks | Process | 3+ |
| Always create Linear issue before implementation | Process | 2+ |
| Node scripts need pty:true | Technical | 1 |
| Gemini CLI has capacity constraints | Technical | 1 |
| Complex multi-file changes: fix from main | Quality | 1 |
| Verify model availability before using in docs | Quality | 1 |
| Concurrent agents on same repo = contention | Technical | 1 |
| Scope bleed from concurrent branches | Technical | 1 |
| When plan approved, don't ask to continue | Autonomy | 3+ |
