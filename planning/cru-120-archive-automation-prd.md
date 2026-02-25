# PRD: Automate Issue Archiving (CRU-120)

## Problem Statement
The `archive-completed-issues.js` script (built in CRU-114) archives Done/Canceled Linear issues but requires manual execution. On Linear's free plan, issue limits matter — stale completed issues consume quota unnecessarily.

## Goal
Automate the archival of completed Linear issues with safety guardrails, so it runs without human intervention while being auditable and configurable.

## User Stories

### US-1: Automated Scheduled Archiving
**As** the team lead, **I want** completed issues to be archived automatically on a schedule, **so that** I don't have to remember to run the script manually.

**Acceptance Criteria:**
- Cron job runs weekly (configurable schedule)
- Results announced to the main session
- No manual intervention required

### US-2: Retention Period
**As** a developer, **I want** recently completed issues to remain visible for a configurable period, **so that** I can reference them before they're archived.

**Acceptance Criteria:**
- Configurable retention period (default: 7 days)
- Issues completed/canceled within the retention window are skipped
- Retention period overridable via environment variable

### US-3: Audit Trail
**As** the team lead, **I want** a log of what was archived and when, **so that** I can verify the automation is working correctly and trace any issues.

**Acceptance Criteria:**
- Each run writes a JSON audit log with issue identifiers, titles, states, and timestamps
- Logs stored in `workspace/logs/archive-YYYY-MM-DD.json`
- Failed archives are logged with error messages
- Logs are append-safe (multiple runs on the same day merge)

## Functional Requirements
- **FR-1:** Enhance existing script with retention period filtering
- **FR-2:** Add structured JSON audit logging
- **FR-3:** Create OpenClaw cron job for weekly execution
- **FR-4:** Dry-run mode preserved for manual testing

## Out of Scope
- Post-Done webhook triggers (over-engineered for current volume)
- UI dashboard for archive history
- Automatic unarchiving

## Technical Notes
- Script location: `workspace/scripts/archive-completed-issues.js`
- Linear API key sourced from `LINEAR_API_KEY` env var (set in openclaw.json)
- Cron runs as isolated agentTurn with announce delivery
