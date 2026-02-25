# Product Requirements Document — CRU-136: CruzBot OS

**Author:** CruzBot (PM)  
**Date:** 2026-02-22  
**Status:** Draft  
**Linear Issue:** CRU-136  
**Input:** [Analyst Brief](cru-136-cruzbot-os-analyst-brief.md)

---

## 1. Executive Summary

CruzBot's four operating systems (Canon, Neo, Wolverine, Evolve) exist as prose in AGENTS.md. Compliance is aspirational. This project turns them into enforceable, scriptable systems — lightweight Node.js scripts in `scripts/os/` that gate, log, audit, and track the behaviors these frameworks define.

**Core bet:** Enforcement accelerates velocity by eliminating rework, not slowing it down. The workflow classifier ensures small tasks stay fast while big features get the rigor they need.

---

## 2. Goals & Success Criteria

| Goal | Metric |
|---|---|
| No dev session starts without pre-flight validation | 100% of dev sessions run `canon-preflight` |
| Every autonomous decision is logged with reasoning | `logs/decisions.jsonl` entries for all non-trivial decisions |
| Script failures auto-generate Ops issues | Error wrapper catches all script failures |
| DNA file changes are tracked with justification | `logs/evolution.jsonl` has entry for every DNA diff |
| Canon compliance is auditable | Audit script can report phase coverage for any issue |
| Off-Canon work is explicitly classified, not silently non-compliant | Classifier runs before workflow selection |
| Lightweight/Off-Canon paths have ≤30s overhead | Measured from classification to dev start |

---

## 3. User Journeys

### 3.1 Full Canon Feature (Main Agent)
1. Agent receives feature request → runs `canon-classify` → returns "Full Canon"
2. Agent spawns Analyst → PM → Architect → SM, each gated by `canon-gate` checking prerequisites
3. SM creates stories, syncs to Linear → `canon-gate` validates story files + Linear states
4. Agent spawns Dev → `canon-preflight` validates: correct Linear state, story exists, standards injected, branch created
5. Dev completes → `canon-postflight` validates: tests pass, PR created, Linear updated
6. PR Reviewer merges → `canon-audit` confirms all phases were hit

### 3.2 Quick Bug Fix (Main Agent)
1. Agent receives bug report → runs `canon-classify` → returns "Off-Canon"
2. Pre-flight is minimal: just standards injection + branch creation
3. Dev fixes → PR → merge. Total enforcement overhead: <15 seconds.

### 3.3 Autonomous Decision (Any Agent)
1. Agent makes a decision (e.g., choosing technical approach)
2. Calls `neo-log-decision` with: decision, reasoning, alternatives, category
3. Entry appended to `logs/decisions.jsonl`
4. Weekly digest script summarizes for human review

### 3.4 Script Failure (Any Agent)
1. Script throws an error inside `wolverine-wrap` wrapper
2. Wrapper catches error, logs to `logs/lessons.jsonl`
3. Wrapper calls `create-ops-issue.js` with error context
4. Agent receives structured error output, can retry or escalate

### 3.5 DNA File Change (Main Agent)
1. Agent modifies `AGENTS.md` during session
2. At session checkpoint (or manually), runs `evolve-diff`
3. Script detects changes via `git diff`, prompts for justification
4. Entry logged to `logs/evolution.jsonl` with diff summary + rationale

---

## 4. Scope

### In Scope
- Node.js scripts in `scripts/os/` (CommonJS, no build step)
- JSONL log files in `logs/` (decisions, lessons, evolution)
- Integration with existing scripts (`start-dev-session.js`, `create-github-pr.js`, `create-ops-issue.js`)
- Linear API for state validation and transitions
- Git for DNA file diffing

### Out of Scope
- `cruzbot-core` xstate engine integration (Phase 3, separate issue)
- Background daemons or file watchers (agents are ephemeral)
- UI/dashboard for logs (JSONL + CLI scripts suffice)
- Modifying Linear workflow states themselves

---

## 5. Functional Requirements

### 5.1 Canon — Workflow Enforcement

| ID | Requirement | Priority | Notes |
|---|---|---|---|
| C-1 | **Workflow Classifier** — Given a Linear issue, return `full-canon`, `lightweight`, or `off-canon` based on: Linear labels, project, file-count estimate, explicit override | P1 | Most critical piece. Default to `lightweight` when ambiguous. |
| C-2 | **Phase Transition Gate** — Validate prerequisites before Linear state transitions. Each phase has a checklist (e.g., "In Dev" requires: story file exists, standards index exists, branch created) | P1 | Exit 0/1. Blocks on failure. |
| C-3 | **Enhanced Pre-flight** — Wraps into `start-dev-session.js`. Validates: correct Linear state for the workflow path, story file exists (if full/lightweight), standards injected, git branch created | P1 | Must not add >10s to dev startup |
| C-4 | **Post-flight Check** — Validates after dev: tests pass (exit code), PR created, Linear state updated to In Review | P2 | Warn on failure, don't block |
| C-5 | **Canon Audit** — Given an issue ID, trace Linear state history + git history to report which Canon phases were completed vs skipped | P2 | Output: JSON report |

### 5.2 Neo — Autonomy Engine

| ID | Requirement | Priority | Notes |
|---|---|---|---|
| N-1 | **Decision Logger** — Append structured entry to `logs/decisions.jsonl`: timestamp, decision, reasoning, alternatives, category, alignment, agent session | P1 | Callable as script or importable function |
| N-2 | **Decision Categories** — Validate category against AGENTS.md taxonomy: `dev-sequence`, `technical-approach`, `model-routing`, `process-improvement`, `quality-gate`, `resource-allocation`, `escalation` | P2 | Reject unknown categories |
| N-3 | **Destructive Action Detector** — Pattern-match commands against destructive patterns (`rm -rf`, `git push --force`, `DROP TABLE`, etc.). Return warn/block with override mechanism | P2 | Allowlist for safe variants |
| N-4 | **Weekly Decision Summary** — Parse `decisions.jsonl`, group by category, output markdown digest | P3 | For human review |

### 5.3 Wolverine — Self-Healing

| ID | Requirement | Priority | Notes |
|---|---|---|---|
| W-1 | **Error Capture Wrapper** — `wolverine-wrap.js <script> [args]` — runs child script, catches non-zero exit / thrown errors, auto-creates Ops issue via `create-ops-issue.js`, logs to `lessons.jsonl` | P1 | Dedup: skip if same error+script within 24h |
| W-2 | **Structured Lesson Log** — Append to `logs/lessons.jsonl`: timestamp, trigger, root cause, lesson, files updated, ops issue ID | P1 | Can be called independently of wrapper |
| W-3 | **P1 Ops Gate** — Check Linear for open P1 Ops issues. If any exist, warn before allowing new feature work (overridable with `--force`) | P2 | Query Linear API filtered by priority + state |
| W-4 | **Recurring Failure Detector** — Scan `lessons.jsonl` for same root cause appearing 3+ times in 30 days. Output escalation report | P3 | Run weekly |

### 5.4 Evolve — Self-Evolution

| ID | Requirement | Priority | Notes |
|---|---|---|---|
| E-1 | **DNA Diff** — Run `git diff` on DNA files (`AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `TOOLS.md`, `USER.md`). Report changes as structured JSON | P1 | Works against HEAD or specified commit |
| E-2 | **Evolution Changelog** — Append to `logs/evolution.jsonl`: timestamp, file, diff summary, justification, trigger type | P1 | Justification is required param |
| E-3 | **Change Justification Gate** — Before committing DNA files, require justification string. If missing, block commit and prompt | P2 | Integrates with git workflow |
| E-4 | **Evolution Digest** — Weekly summary of all DNA changes for human review | P3 | Parse `evolution.jsonl` → markdown |

---

## 6. Non-Functional Requirements

| Requirement | Target |
|---|---|
| **Performance** | No script adds >10s to agent workflow. Classification <3s. |
| **Portability** | Windows (PowerShell) + Node.js v22. No Unix-only tooling. |
| **Dependencies** | Zero new npm packages. Use `https`, `fs`, `path`, `child_process` only. |
| **Output format** | All scripts output JSON to stdout. Gates exit 0 (pass) or 1 (fail). |
| **Log retention** | JSONL files. No auto-rotation in v1 (add in Phase 2+). |
| **Backward compat** | Existing scripts unchanged. New scripts extend, not replace. |
| **Context efficiency** | Script output ≤50 lines. Agents have limited context windows. |

---

## 7. Design Decisions

### 7.1 Scripts, Not a Framework
Phase 1 uses standalone CommonJS scripts. No TypeScript, no build step, no xstate. Prove the enforcement model works before adding complexity. `cruzbot-core` integration is Phase 3.

### 7.2 Block vs Warn
- **P1 requirements (C-1, C-2, C-3, N-1, W-1, W-2, E-1, E-2):** Gates BLOCK (exit 1).
- **P2 requirements (C-4, C-5, N-3, W-3, E-3):** Gates WARN (exit 0, stderr warning).
- Override with `--force` flag, which itself gets logged.

### 7.3 Log Format
All JSONL files share a base schema:
```json
{"ts": "ISO-8601", "system": "canon|neo|wolverine|evolve", "type": "event-type", ...payload}
```

### 7.4 Classifier Defaults
When the classifier can't determine workflow type, default to `lightweight` (not `full-canon`). Rationale: false-positive enforcement kills velocity; false-negative is caught in audit.

### 7.5 Neo Decisions — Local First
Log decisions to local JSONL, not Linear comments. Linear comments are noisy and non-queryable. Add Linear integration later if needed.

### 7.6 Wolverine — Auto-Create All, Dedup
Create Ops issues for ALL script failures (not just recurring). Dedup by script+error hash within 24h window to prevent spam.

---

## 8. Epics & Stories

### Epic 1: Canon Enforcement (Phase 1 — Foundation)

**Goal:** Classify workflows and gate phase transitions so Canon is mechanically enforced.

#### Story 1.1: Workflow Classifier (`canon-classify.js`)
**As** the main agent, **I want** to classify a Linear issue into Full Canon / Lightweight / Off-Canon **so that** the correct workflow path is followed without manual judgment each time.

**Acceptance Criteria:**
- [ ] Script accepts Linear issue ID as argument
- [ ] Fetches issue from Linear API (title, labels, project, description)
- [ ] Classification logic:
  - Labels containing `full-canon` → `full-canon`
  - Labels containing `off-canon` or `hotfix` → `off-canon`
  - Project = Ops or Docs → `lightweight`
  - Description mentions "bug fix" or ≤3 files estimated → `off-canon`
  - Default → `lightweight`
- [ ] Supports `--override <type>` flag to force classification
- [ ] Outputs JSON: `{ "issueId": "CRU-XX", "classification": "full-canon|lightweight|off-canon", "reason": "...", "override": bool }`
- [ ] Exit 0 always (classification never fails, it defaults)
- [ ] Tests: unit tests for each classification path + override

#### Story 1.2: Phase Transition Gate (`canon-gate.js`)
**As** the main agent, **I want** to validate prerequisites before transitioning a Linear issue between phases **so that** no phase is skipped accidentally.

**Acceptance Criteria:**
- [ ] Script accepts: issue ID, target state name
- [ ] Defines prerequisite map per workflow type:
  - **Full Canon:** Analyst→PM (brief exists), PM→Architect (PRD exists), Architect→SM (arch doc exists), SM→Dev (story files exist + Linear sub-issues created), Dev→QA (tests pass + PR exists), QA→Done (PR merged)
  - **Lightweight:** PM→SM (PRD or spec exists), SM→Dev (story files exist), Dev→Done (PR merged)
  - **Off-Canon:** Dev→Done (PR merged)
- [ ] Checks file existence in `_bmad-output/` and Linear API for sub-issue states
- [ ] Output JSON: `{ "gate": "pass|fail", "target": "In Dev", "missing": [...], "workflow": "full-canon" }`
- [ ] Exit 0 on pass, exit 1 on fail
- [ ] `--force` flag allows bypass (logged to `logs/decisions.jsonl` as override)
- [ ] Tests: unit tests for each transition + missing prereq scenarios

#### Story 1.3: Enhanced Pre-flight (`canon-preflight.js`)
**As** a dev agent starting a session, **I want** pre-flight checks to validate my setup **so that** I don't start coding without proper context.

**Acceptance Criteria:**
- [ ] Script accepts Linear issue ID
- [ ] Calls `canon-classify.js` to determine workflow type
- [ ] Checks based on workflow type:
  - **All paths:** Git branch exists or is created, standards index file exists
  - **Full/Lightweight:** Story file exists in `_bmad-output/`, Linear state is "In Dev" or "Sprint Ready"
  - **Full Canon only:** Architecture doc exists
- [ ] Output JSON: `{ "preflight": "pass|fail", "workflow": "...", "checks": { "branch": true, "story": true, ... } }`
- [ ] Exit 0 pass, exit 1 fail
- [ ] Integrates into `start-dev-session.js` (called as first step)
- [ ] Total execution time <10s
- [ ] Tests: mock Linear API + filesystem checks

### Epic 2: Neo Decision Tracking (Phase 1 — Foundation)

**Goal:** Create a structured, queryable log of all autonomous decisions.

#### Story 2.1: Decision Logger (`neo-log-decision.js`)
**As** any agent, **I want** to log my autonomous decisions in a structured format **so that** decisions are auditable and patterns are visible.

**Acceptance Criteria:**
- [ ] Script accepts via CLI args or JSON stdin: `decision`, `reasoning`, `alternatives` (optional), `category`, `alignment` (optional), `agent` (optional)
- [ ] Valid categories: `dev-sequence`, `technical-approach`, `model-routing`, `process-improvement`, `quality-gate`, `resource-allocation`, `escalation`
- [ ] Rejects unknown categories with helpful error
- [ ] Appends to `logs/decisions.jsonl` with ISO timestamp
- [ ] Creates `logs/` directory if it doesn't exist
- [ ] Output JSON: `{ "logged": true, "file": "logs/decisions.jsonl", "entry": {...} }`
- [ ] Exit 0 on success
- [ ] Tests: valid entry, invalid category, file creation

### Epic 3: Wolverine Self-Healing (Phase 1 — Foundation)

**Goal:** Automatically capture failures and lessons so nothing is silently lost.

#### Story 3.1: Error Capture Wrapper (`wolverine-wrap.js`)
**As** the main agent, **I want** to wrap script execution so failures are automatically captured as Ops issues **so that** no failure goes untracked.

**Acceptance Criteria:**
- [ ] Usage: `node wolverine-wrap.js <script-path> [args...]`
- [ ] Runs target script as child process, captures stdout/stderr/exit code
- [ ] On non-zero exit:
  - Logs to `logs/lessons.jsonl`: timestamp, script, args, error output, exit code
  - Calls `create-ops-issue.js` with: title = `Script failure: <script>`, description = error context
  - Dedup: checks `lessons.jsonl` for same script+error hash in last 24h, skips Ops issue if duplicate
- [ ] On success: passes through stdout, exits 0
- [ ] Output on failure includes original script output + wrapper metadata
- [ ] Tests: success passthrough, failure capture, dedup logic

#### Story 3.2: Structured Lesson Logger (`wolverine-log-lesson.js`)
**As** any agent, **I want** to log lessons learned in a structured format **so that** patterns of failure are detectable.

**Acceptance Criteria:**
- [ ] Accepts via CLI args or JSON stdin: `trigger`, `rootCause`, `lesson`, `filesUpdated` (optional), `opsIssueId` (optional)
- [ ] Appends to `logs/lessons.jsonl` with ISO timestamp
- [ ] Creates `logs/` directory if needed
- [ ] Output JSON: `{ "logged": true, "entry": {...} }`
- [ ] Tests: valid entry, file creation

### Epic 4: Evolve Self-Evolution Tracking (Phase 1 — Foundation)

**Goal:** Track all changes to core DNA files with justification.

#### Story 4.1: DNA Diff Detector (`evolve-diff.js`)
**As** the main agent, **I want** to detect changes to DNA files since last commit **so that** no evolution goes unnoticed.

**Acceptance Criteria:**
- [ ] DNA files: `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `TOOLS.md`, `USER.md`
- [ ] Runs `git diff HEAD -- <files>` to detect uncommitted changes
- [ ] Also supports `git diff <commit> HEAD -- <files>` for committed range
- [ ] Output JSON: `{ "changed": ["AGENTS.md"], "unchanged": [...], "diffs": { "AGENTS.md": "<summary>" } }`
- [ ] Diff summary truncated to 500 chars per file (context-friendly)
- [ ] Exit 0 if no changes, exit 1 if changes detected (so it can be used as a gate)
- [ ] Tests: mock git output, changed/unchanged scenarios

#### Story 4.2: Evolution Changelog Logger (`evolve-log.js`)
**As** the main agent, **I want** to log DNA file changes with justification **so that** evolution is traceable.

**Acceptance Criteria:**
- [ ] Accepts: `file`, `diffSummary`, `justification` (required), `trigger` (one of: `user-request`, `self-improvement`, `wolverine-lesson`, `feedback`)
- [ ] Rejects if justification is empty/missing
- [ ] Appends to `logs/evolution.jsonl` with ISO timestamp
- [ ] Output JSON: `{ "logged": true, "entry": {...} }`
- [ ] Tests: valid entry, missing justification rejection

---

### Epic 5: Enforcement Layer (Phase 2)

**Goal:** Add post-flight validation, destructive action detection, P1 blocking, and change justification gates.

#### Story 5.1: Post-flight Check (`canon-postflight.js`)
**AC:**
- [ ] Validates after dev: tests passed (accepts exit code as arg), PR created (checks GitHub API or local git), Linear state updated
- [ ] Warns on failure (exit 0 with warnings), does not block
- [ ] Integrates with `create-github-pr.js` flow

#### Story 5.2: Canon Audit Report (`canon-audit.js`)
**AC:**
- [ ] Accepts issue ID, fetches Linear state history via API
- [ ] Cross-references with git history (branch, PRs, commits)
- [ ] Outputs compliance report: which phases hit, which skipped, workflow classification
- [ ] JSON + markdown output modes

#### Story 5.3: Destructive Action Detector (`neo-detect-destructive.js`)
**AC:**
- [ ] Accepts a command string, checks against pattern list (rm -rf, force push, drop, delete repo, etc.)
- [ ] Returns `{ "destructive": true/false, "pattern": "...", "severity": "warn|block" }`
- [ ] Allowlist for safe variants (e.g., `rm` single file is warn, `rm -rf /` is block)
- [ ] `--override` with logged justification

#### Story 5.4: P1 Ops Gate (`wolverine-p1-gate.js`)
**AC:**
- [ ] Queries Linear for open Ops issues with priority P1/Urgent
- [ ] If any found, warns before feature work proceeds
- [ ] `--force` allows bypass, logged to `decisions.jsonl`
- [ ] JSON output with issue list

#### Story 5.5: Change Justification Gate (`evolve-justify.js`)
**AC:**
- [ ] Checks staged git files for DNA file changes
- [ ] If DNA files modified, requires justification argument
- [ ] Blocks commit (exit 1) if justification missing
- [ ] Logs to `evolution.jsonl` on success

---

### Epic 6: Intelligence Layer (Phase 3)

**Goal:** Add summaries, pattern detection, and cruzbot-core integration.

#### Story 6.1: Weekly Decision Summary (`neo-summary.js`)
**AC:**
- [ ] Parses `decisions.jsonl` for last 7 days
- [ ] Groups by category, counts, highlights significant decisions
- [ ] Outputs markdown digest

#### Story 6.2: Recurring Failure Detector (`wolverine-patterns.js`)
**AC:**
- [ ] Scans `lessons.jsonl` for same root cause 3+ times in 30 days
- [ ] Outputs escalation report with pattern details
- [ ] Suggests remediation based on frequency/severity

#### Story 6.3: Evolution Review Digest (`evolve-digest.js`)
**AC:**
- [ ] Parses `evolution.jsonl` for last 7 days
- [ ] Summarizes all DNA changes with justifications
- [ ] Outputs markdown digest

#### Story 6.4: cruzbot-core Workflow Engine Integration
**AC:**
- [ ] Connect xstate-based workflow engine to Canon enforcement scripts
- [ ] Engine manages phase transitions, calls gate scripts at each step
- [ ] Agent spawning via OpenClaw `sessions_spawn` driven by engine state
- [ ] Requires separate architecture doc (deferred to Architect phase)

---

## 9. Phased Delivery Plan

### Phase 1: Foundation (MVP) — Epics 1-4
**Stories:** 1.1, 1.2, 1.3, 2.1, 3.1, 3.2, 4.1, 4.2 (8 stories)  
**Delivers:** Workflow classification, phase gates, pre-flight, decision logging, error capture, DNA tracking  
**Risk:** Low — standalone scripts, no integration complexity  
**Validation:** Run classifier + preflight on 3 real issues, verify correct classification and gating

### Phase 2: Enforcement — Epic 5
**Stories:** 5.1–5.5 (5 stories)  
**Delivers:** Post-flight, audit, destructive action detection, P1 blocking, justification gates  
**Risk:** Medium — deeper integration with existing scripts  
**Depends on:** Phase 1 complete + at least 1 week of real-world usage data

### Phase 3: Intelligence — Epic 6
**Stories:** 6.1–6.4 (4 stories)  
**Delivers:** Weekly digests, pattern detection, cruzbot-core integration  
**Risk:** Medium-High — cruzbot-core integration is the most complex piece  
**Depends on:** Phase 2 complete + JSONL logs have enough data for meaningful analysis

---

## 10. File Structure

```
scripts/os/
  canon-classify.js       # C-1: Workflow classifier
  canon-gate.js           # C-2: Phase transition gate
  canon-preflight.js      # C-3: Pre-flight checks
  canon-postflight.js     # C-4: Post-flight checks
  canon-audit.js          # C-5: Compliance audit
  neo-log-decision.js     # N-1: Decision logger
  neo-detect-destructive.js # N-3: Destructive action detector
  neo-summary.js          # N-4: Weekly decision summary
  wolverine-wrap.js       # W-1: Error capture wrapper
  wolverine-log-lesson.js # W-2: Lesson logger
  wolverine-p1-gate.js    # W-3: P1 ops gate
  wolverine-patterns.js   # W-4: Recurring failure detector
  evolve-diff.js          # E-1: DNA file diff
  evolve-log.js           # E-2: Evolution changelog
  evolve-justify.js       # E-3: Change justification gate
  evolve-digest.js        # E-4: Evolution digest

logs/
  decisions.jsonl         # Neo decision log
  lessons.jsonl           # Wolverine lesson log
  evolution.jsonl         # Evolve changelog
```

---

## 11. Open Questions (Resolved)

| Question | Resolution |
|---|---|
| How strict should gates be? | P1 gates block, P2 gates warn. Override with `--force` + logged justification. |
| Neo decisions: Linear or local? | Local JSONL first. Linear comments are noisy. |
| Wolverine: all failures or recurring only? | All failures, with 24h dedup. |
| What are DNA files? | `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `TOOLS.md`, `USER.md` |
| Classifier signal source? | Labels first, project second, description analysis as fallback. Default: lightweight. |

---

*PRD complete. Ready for Architecture phase.*
