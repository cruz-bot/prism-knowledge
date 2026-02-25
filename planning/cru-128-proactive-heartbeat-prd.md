# PRD: Proactive Heartbeat — Autonomous Work Continuation (CRU-128)

## Problem Statement
The agent goes idle between user messages. The heartbeat fires every 2 hours but only monitors for problems — it doesn't continue active work. This creates multi-hour gaps where subagent work completes but the next step doesn't kick off until the user checks Telegram.

The root cause is twofold:
1. **Heartbeat interval too long** — 2h is appropriate for monitoring but not for driving active dev sprints
2. **HEARTBEAT.md is monitoring-only** — it checks resources and status but doesn't include "continue current work" as a standing task
3. **No documented principle** — AGENTS.md doesn't codify the expectation that the agent should chain work proactively

## Goal
Make the agent autonomously continue sprint work during active hours. When the user kicks off work and walks away, they should come back to progress — not an idle agent waiting to be poked.

## User Stories

### US-1: Faster Heartbeat During Active Hours
**As** the team lead, **I want** the heartbeat to fire more frequently during active dev work, **so that** pending next-steps get picked up within minutes, not hours.

**Acceptance Criteria:**
- Heartbeat interval reduced to 30min during active hours (8am–11pm CST)
- 2h interval preserved for overnight/quiet hours (or no heartbeat at all)

### US-2: Work Continuation in Heartbeat
**As** the team lead, **I want** the heartbeat to check for and continue pending work, **so that** sprint momentum isn't lost between interactions.

**Acceptance Criteria:**
- HEARTBEAT.md includes a "Continue Active Sprint" standing task
- Heartbeat checks: pending PR reviews, queued dev fixes, next wave items
- Agent spawns appropriate subagents when work is ready

### US-3: Proactive Chaining Principle
**As** the team lead, **I want** the autonomous work chaining behavior documented as a core principle, **so that** it persists across sessions and isn't forgotten.

**Acceptance Criteria:**
- AGENTS.md updated with a "Proactive Execution" section under Neo
- Principle: when a subagent completes, immediately evaluate and spawn the next step
- Principle: don't wait for user input when the plan is already clear

## Functional Requirements
- **FR-1:** Update heartbeat config to 30min interval
- **FR-2:** Update HEARTBEAT.md with work continuation checklist
- **FR-3:** Update AGENTS.md with proactive chaining principle
- **FR-4:** Wolverine lesson documented

## Out of Scope
- Background daemon/persistent process (OpenClaw architectural limitation)
- Custom heartbeat intervals per-project
- Auto-detecting "active sprint" vs "idle" mode
