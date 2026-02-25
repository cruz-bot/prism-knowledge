# Architecture Document — CRU-136: CruzBot OS

**Author:** CruzBot (Architect)  
**Date:** 2026-02-22  
**Status:** Complete  
**Inputs:** [Analyst Brief](cru-136-cruzbot-os-analyst-brief.md) · [PRD](cru-136-cruzbot-os-prd.md)

---

## 1. System Overview

CruzBot OS is a set of standalone CommonJS Node.js scripts that enforce four operating frameworks (Canon, Neo, Wolverine, Evolve). Scripts are invoked by agents during sessions — no daemons, no watchers, no build step, zero new dependencies.

```
┌─────────────────────────────────────────────────────────┐
│                     Agent Session                       │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌────────┐│
│  │  Canon   │  │   Neo    │  │ Wolverine │  │ Evolve ││
│  │ (gates)  │  │ (decide) │  │  (heal)   │  │(track) ││
│  └────┬─────┘  └────┬─────┘  └─────┬─────┘  └───┬────┘│
│       │              │              │             │     │
│       v              v              v             v     │
│  ┌─────────────────────────────────────────────────────┐│
│  │              logs/ (JSONL)                          ││
│  │  decisions.jsonl · lessons.jsonl · evolution.jsonl  ││
│  └─────────────────────────────────────────────────────┘│
│       │              │                                  │
│       v              v                                  │
│  ┌──────────┐  ┌───────────┐                           │
│  │ Linear   │  │   Git     │                           │
│  │  API     │  │  (repo)   │                           │
│  └──────────┘  └───────────┘                           │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Directory Structure

```
workspace/
├── scripts/
│   ├── os/                          # ← ALL new scripts live here
│   │   ├── lib/
│   │   │   ├── linear-client.js     # Shared Linear GraphQL helper
│   │   │   ├── log-writer.js        # Shared JSONL append + dir creation
│   │   │   ├── git-helpers.js       # Shared git commands (diff, branch)
│   │   │   └── schemas.js           # JSON schema validation (inline)
│   │   ├── canon-classify.js        # C-1  Workflow classifier
│   │   ├── canon-gate.js            # C-2  Phase transition gate
│   │   ├── canon-preflight.js       # C-3  Dev session pre-flight
│   │   ├── canon-postflight.js      # C-4  Post-flight check
│   │   ├── canon-audit.js           # C-5  Compliance audit
│   │   ├── neo-log-decision.js      # N-1  Decision logger
│   │   ├── neo-detect-destructive.js# N-3  Destructive action detector
│   │   ├── neo-summary.js           # N-4  Weekly decision summary
│   │   ├── wolverine-wrap.js        # W-1  Error capture wrapper
│   │   ├── wolverine-log-lesson.js  # W-2  Lesson logger
│   │   ├── wolverine-p1-gate.js     # W-3  P1 ops gate
│   │   ├── wolverine-patterns.js    # W-4  Recurring failure detector
│   │   ├── evolve-diff.js           # E-1  DNA file diff
│   │   ├── evolve-log.js            # E-2  Evolution changelog
│   │   ├── evolve-justify.js        # E-3  Change justification gate
│   │   └── evolve-digest.js         # E-4  Evolution digest
│   │
│   ├── start-dev-session.js         # Existing — will call canon-preflight
│   ├── create-github-pr.js          # Existing — will call canon-postflight
│   ├── create-ops-issue.js          # Existing — called by wolverine-wrap
│   ├── bmad-to-linear.js            # Existing — unchanged
│   └── ...
│
├── logs/                            # ← Runtime log directory
│   ├── decisions.jsonl              # Neo decisions
│   ├── lessons.jsonl                # Wolverine lessons
│   └── evolution.jsonl              # Evolve changelog
│
├── _bmad-output/                    # Existing planning artifacts
└── secrets/                         # Existing credentials
```

### Key Decisions

- **`scripts/os/lib/`** contains three shared modules extracted from duplicated patterns across existing scripts (Linear API calls, file writing, git commands). These are internal — not a framework.
- **`logs/`** lives in workspace root (not repo) — these are agent operational logs, not source code.
- All scripts are `#!/usr/bin/env node` CommonJS. No ESM (existing infra is mixed; new scripts standardize on CJS for consistency with `create-ops-issue.js` and `start-dev-session.js`).

---

## 3. Module Interfaces & Contracts

### 3.1 Universal Script Contract

Every script in `scripts/os/` follows this contract:

```
INPUT:   CLI args (primary) or JSON on stdin (for complex payloads)
OUTPUT:  Single JSON object to stdout (≤50 lines)
EXIT:    0 = success/pass, 1 = failure/blocked
STDERR:  Human-readable messages (warnings, progress)
DEPS:    Only Node.js built-ins (https, fs, path, child_process, crypto)
```

**JSON output envelope:**
```json
{
  "ok": true,
  "system": "canon|neo|wolverine|evolve",
  "action": "classify|gate|preflight|log-decision|...",
  "ts": "2026-02-22T20:00:00.000Z",
  "data": { ... }
}
```

On failure:
```json
{
  "ok": false,
  "system": "canon",
  "action": "gate",
  "ts": "...",
  "error": "Missing story file for CRU-136",
  "missing": ["story-file", "standards-index"]
}
```

### 3.2 Shared Library Interfaces

#### `lib/linear-client.js`

```js
// Extracted from existing scripts — same pattern as start-dev-session.js
module.exports = {
  gql(query, variables) → Promise<Object>,   // Raw GraphQL
  getIssue(issueId) → Promise<{id, title, state, labels[], project, description}>,
  updateIssueState(issueId, stateId) → Promise<void>,
  getOpenP1OpsIssues() → Promise<Issue[]>,
  getIssueHistory(issueId) → Promise<StateChange[]>
};
```

#### `lib/log-writer.js`

```js
module.exports = {
  append(logFile, entry) → void,   // Append JSON line, create dir if needed
  read(logFile, opts) → Object[],  // Read + parse, opts: {since, system, limit}
  hash(str) → string               // SHA-256 short hash for dedup
};
```

#### `lib/git-helpers.js`

```js
module.exports = {
  diff(files, base) → {changed[], diffs{}},  // git diff output
  currentBranch() → string,
  branchExists(name) → boolean,
  hasUncommittedChanges(files) → boolean
};
```

### 3.3 Inter-Module Contracts

Scripts call each other via `child_process.execSync` (not `require`). This keeps them independently runnable and testable.

```
canon-preflight.js
  ├── calls → canon-classify.js (get workflow type)
  └── calls → canon-gate.js (validate phase prereqs)

wolverine-wrap.js
  ├── calls → <target-script> (child process)
  ├── calls → wolverine-log-lesson.js (on failure)
  └── calls → create-ops-issue.js (on failure, existing script)

evolve-diff.js
  └── calls → evolve-log.js (when changes detected + justification provided)

start-dev-session.js (existing, modified)
  └── calls → canon-preflight.js (new integration point)

create-github-pr.js (existing, modified)
  └── calls → canon-postflight.js (new integration point)
```

**Cross-system interaction:** Systems are loosely coupled. The only cross-system call is `canon-gate.js --force` logging an override to `logs/decisions.jsonl` (via `neo-log-decision.js`). This is the single bridge between Canon enforcement and Neo audit trail.

---

## 4. Data Formats (JSON Schemas)

### 4.1 Base Log Entry

All JSONL entries share:

```json
{
  "ts": "string (ISO-8601)",
  "system": "enum: canon|neo|wolverine|evolve",
  "type": "string (event type)",
  "agent": "string (session label, optional)"
}
```

### 4.2 `logs/decisions.jsonl` — Neo

```json
{
  "ts": "2026-02-22T20:00:00.000Z",
  "system": "neo",
  "type": "decision",
  "decision": "Use lightweight canon for CRU-140",
  "reasoning": "Docs-only change, 2 files affected",
  "alternatives": ["full-canon", "off-canon"],
  "category": "enum: dev-sequence|technical-approach|model-routing|process-improvement|quality-gate|resource-allocation|escalation",
  "alignment": "Matches AGENTS.md guidance: ops/docs → lightweight",
  "agent": "cru-140-dev",
  "override": false
}
```

### 4.3 `logs/lessons.jsonl` — Wolverine

```json
{
  "ts": "2026-02-22T20:00:00.000Z",
  "system": "wolverine",
  "type": "lesson",
  "trigger": "wolverine-wrap caught script failure",
  "script": "scripts/os/canon-gate.js",
  "args": ["CRU-136", "In Dev"],
  "exitCode": 1,
  "errorHash": "a1b2c3d4",
  "rootCause": "Linear API returned 401 — token expired",
  "lesson": "Add token validation at script startup",
  "filesUpdated": [],
  "opsIssueId": "CRU-141",
  "dedup": false
}
```

### 4.4 `logs/evolution.jsonl` — Evolve

```json
{
  "ts": "2026-02-22T20:00:00.000Z",
  "system": "evolve",
  "type": "dna-change",
  "file": "AGENTS.md",
  "diffSummary": "Added P1 ops enforcement to Wolverine section (+12 -3 lines)",
  "justification": "Codifying lesson from CRU-138: P1 workarounds were slipping through",
  "trigger": "enum: user-request|self-improvement|wolverine-lesson|feedback",
  "agent": "main"
}
```

### 4.5 Canon Gate Output

```json
{
  "ok": true,
  "system": "canon",
  "action": "gate",
  "ts": "...",
  "data": {
    "issueId": "CRU-136",
    "workflow": "full-canon",
    "targetState": "In Dev",
    "gate": "pass",
    "checks": {
      "story-file": { "pass": true, "path": "_bmad-output/stories/cru-136-story-1.md" },
      "architecture-doc": { "pass": true, "path": "_bmad-output/planning-artifacts/cru-136-...-architecture.md" },
      "standards-index": { "pass": true },
      "linear-state": { "pass": true, "current": "Sprint Ready" },
      "git-branch": { "pass": true, "branch": "feature/cru-136-cruzbot-os" }
    },
    "missing": []
  }
}
```

### 4.6 Canon Classifier Output

```json
{
  "ok": true,
  "system": "canon",
  "action": "classify",
  "ts": "...",
  "data": {
    "issueId": "CRU-136",
    "classification": "full-canon",
    "reason": "Label: full-canon",
    "override": false,
    "signals": {
      "labels": ["full-canon", "cruzbot-os"],
      "project": "CruzBot",
      "estimatedFiles": null
    }
  }
}
```

---

## 5. Canon State Machine

### 5.1 Phase Transitions

```
                    ┌──────────────────────────────────────────────┐
  FULL CANON:       │                                              │
                    │  Backlog → Analyst → PM → Architect → SM    │
                    │           → Sprint Ready → In Dev → In QA   │
                    │           → In Review → Done                 │
                    └──────────────────────────────────────────────┘

                    ┌──────────────────────────────────────────────┐
  LIGHTWEIGHT:      │                                              │
                    │  Backlog → PM → SM → Sprint Ready → In Dev  │
                    │           → In Review → Done                 │
                    └──────────────────────────────────────────────┘

                    ┌──────────────────────────────────────────────┐
  OFF-CANON:        │                                              │
                    │  Backlog → In Dev → In Review → Done         │
                    └──────────────────────────────────────────────┘
```

### 5.2 Gate Prerequisites Per Transition

| From → To | Full Canon | Lightweight | Off-Canon |
|---|---|---|---|
| → Analyst | Issue exists | n/a | n/a |
| Analyst → PM | Analyst brief exists in `_bmad-output/` | n/a | n/a |
| PM → Architect | PRD exists in `_bmad-output/` | n/a | n/a |
| Architect → SM | Architecture doc exists | n/a | n/a |
| → PM (lightweight) | n/a | Issue exists | n/a |
| PM → SM | PRD or spec exists | PRD or spec exists | n/a |
| SM → Sprint Ready | Story files exist + Linear sub-issues created | Story files exist | n/a |
| Sprint Ready → In Dev | Standards index exists, git branch | Standards index, git branch | Standards index, git branch |
| In Dev → In QA | Tests pass (exit 0 provided) | n/a | n/a |
| In Dev → In Review | PR created | PR created | PR created |
| In Review → Done | PR merged | PR merged | PR merged |

### 5.3 Implementation

`canon-gate.js` encodes this as a lookup table:

```js
const GATES = {
  'full-canon': {
    'Analyst':      { requires: ['issue-exists'] },
    'PM':           { requires: ['analyst-brief'] },
    'Architect':    { requires: ['prd'] },
    'SM':           { requires: ['architecture-doc'] },  // was 'Sprint Ready'
    'Sprint Ready': { requires: ['story-files', 'linear-sub-issues'] },
    'In Dev':       { requires: ['standards-index', 'git-branch'] },
    'In QA':        { requires: ['tests-pass'] },
    'In Review':    { requires: ['pr-created'] },
    'Done':         { requires: ['pr-merged'] }
  },
  'lightweight': {
    'PM':           { requires: ['issue-exists'] },
    'SM':           { requires: ['prd-or-spec'] },
    'Sprint Ready': { requires: ['story-files'] },
    'In Dev':       { requires: ['standards-index', 'git-branch'] },
    'In Review':    { requires: ['pr-created'] },
    'Done':         { requires: ['pr-merged'] }
  },
  'off-canon': {
    'In Dev':       { requires: ['standards-index', 'git-branch'] },
    'In Review':    { requires: ['pr-created'] },
    'Done':         { requires: ['pr-merged'] }
  }
};
```

Each `requires` key maps to a checker function that returns `{pass: bool, detail: string}`. Checkers are file-existence checks, Linear API queries, or git commands.

### 5.4 Override Mechanism

Any gate can be bypassed with `--force`. When used:
1. Gate logs override to `logs/decisions.jsonl` via `neo-log-decision.js`
2. Gate exits 0 (pass) but output includes `"override": true`
3. `canon-audit.js` flags overrides in compliance reports

---

## 6. Integration Points

### 6.1 `start-dev-session.js` Integration

**Change:** Add `canon-preflight.js` call as first step.

```js
// At top of main() in start-dev-session.js:
const { execSync } = require('child_process');
const preflightResult = JSON.parse(
  execSync(`node scripts/os/canon-preflight.js ${issueId}`, { encoding: 'utf8' })
);
if (!preflightResult.ok) {
  console.error('❌ Pre-flight failed:', preflightResult.missing);
  process.exit(1);
}
// ... existing logic continues
```

**Note:** `create-github-pr.js` is ESM (`import`). The new OS scripts are CJS. Integration uses `child_process.execSync` (works across module systems). Alternatively, `create-github-pr.js` can be gradually migrated to CJS, but this is not required for Phase 1.

### 6.2 `create-github-pr.js` Integration (Phase 2)

Post-flight call added after PR creation succeeds:

```js
execSync(`node scripts/os/canon-postflight.js ${issueId}`, { encoding: 'utf8' });
```

### 6.3 `create-ops-issue.js` Integration

`wolverine-wrap.js` calls the existing `create-ops-issue.js` as a child process:

```js
execSync(`node scripts/create-ops-issue.js "${title}" "${description}" ${priority}`);
```

No changes to `create-ops-issue.js` needed.

### 6.4 OpenClaw `sessions_spawn` Integration

Scripts don't call `sessions_spawn` directly — they output structured JSON that the main agent interprets. The agent reads classifier output and spawns the appropriate phase agent:

```
Agent reads: { "classification": "full-canon" }
Agent spawns: Analyst → PM → Architect → SM → Dev (sequentially, gated)
```

Between spawns, the agent runs `canon-gate.js` to validate the prior phase's output before spawning the next.

### 6.5 `bmad-to-linear.js` — No Changes

Remains independent. Canon gates check for its *output* (Linear sub-issues exist) but don't call it directly.

---

## 7. Storage Strategy

### 7.1 Log Files

| File | Location | Format | Rotation |
|---|---|---|---|
| `decisions.jsonl` | `workspace/logs/` | JSONL | None in v1; Phase 2 adds 30-day archive |
| `lessons.jsonl` | `workspace/logs/` | JSONL | None in v1 |
| `evolution.jsonl` | `workspace/logs/` | JSONL | None in v1 |

**Why JSONL:**
- Append-only (no read-modify-write race conditions)
- Line-per-entry (greppable, streamable)
- No dependency (native `fs.appendFileSync`)
- Easy to query: `read + filter + JSON.parse` per line

**Why workspace (not repo):**
- Operational logs ≠ source code
- Avoids polluting git history with every decision/lesson
- Stays with the agent's workspace, not the project

### 7.2 Artifact Checks

Canon gates check for planning artifacts in:
- `workspace/_bmad-output/planning-artifacts/` — briefs, PRDs, arch docs
- `workspace/_bmad-output/stories/` — story files (pattern: `*<issue-id>*`)
- `repo/` (prism) — source code, tests, branches

### 7.3 Credentials

All scripts read Linear API key from `workspace/secrets/linear-token.txt` (existing pattern). No new credential files.

### 7.4 Git as Source of Truth

- **Evolve** uses `git diff` for DNA file change detection — not filesystem comparison
- **Canon audit** uses `git log` for branch/PR/merge history
- No local state files for tracking "last known" — git is the single source

---

## 8. Error Handling Patterns

### 8.1 Script-Level Error Handling

```js
async function main() {
  try {
    // ... script logic
    console.log(JSON.stringify({ ok: true, system: 'canon', action: '...', ts: new Date().toISOString(), data: result }));
    process.exit(0);
  } catch (err) {
    console.log(JSON.stringify({ ok: false, system: 'canon', action: '...', ts: new Date().toISOString(), error: err.message }));
    process.exit(1);
  }
}
main();
```

### 8.2 Linear API Errors

- **401 Unauthorized:** Log error, suggest token refresh, exit 1
- **Rate limited (429):** Retry once after 1s delay, then fail
- **Network error:** Fail with clear message, no retry (agent can retry manually)
- **Issue not found:** Fail with `"error": "Issue CRU-XXX not found"`

### 8.3 Git Command Errors

- **Not in repo:** Detect via `git rev-parse --git-dir`, fail with clear message
- **Branch doesn't exist:** Return `{pass: false}` in gate check, not a crash
- **Dirty working tree:** Report state, don't block (informational)

### 8.4 Wolverine Wrapper Error Handling

```
wolverine-wrap.js runs <script>
  ├── Script exits 0 → pass through stdout, exit 0
  ├── Script exits non-0 →
  │     1. Capture stderr + stdout
  │     2. Compute error hash (script + first line of error)
  │     3. Check lessons.jsonl for same hash in last 24h
  │     4. If new: log lesson + create ops issue
  │     5. If duplicate: log lesson only (skip ops issue)
  │     6. Output combined JSON (original error + wrapper metadata)
  │     7. Exit 1
  └── Script crashes (signal) → same as non-0 exit
```

### 8.5 Graceful Degradation

If a shared library (`lib/linear-client.js`) can't connect to Linear:
- Gate scripts that need Linear → fail (exit 1)
- Logger scripts (neo, wolverine, evolve) → succeed locally (don't need Linear)
- Classifier → fall back to default `lightweight` classification

---

## 9. Testing Strategy

### 9.1 Approach

Unit tests per script, co-located in `scripts/os/__tests__/`. Tests use Node.js built-in `node:test` and `node:assert` (available in Node 22, zero dependencies).

### 9.2 Test Structure

```
scripts/os/__tests__/
  canon-classify.test.js
  canon-gate.test.js
  canon-preflight.test.js
  neo-log-decision.test.js
  wolverine-wrap.test.js
  wolverine-log-lesson.test.js
  evolve-diff.test.js
  evolve-log.test.js
  lib/
    log-writer.test.js
    linear-client.test.js
```

### 9.3 Test Patterns

**Mock strategy:** Inject dependencies via environment variables or function arguments.

```js
// linear-client.js supports mock mode:
const MOCK = process.env.CRUZBOT_OS_MOCK === '1';
// In mock mode, returns fixture data instead of calling Linear API
```

**Filesystem tests:** Use `os.tmpdir()` for log files, clean up in `after()`.

**Git tests:** Mock `child_process.execSync` by extracting git calls into `lib/git-helpers.js` and injecting a test double.

### 9.4 Test Runner

```bash
node --test scripts/os/__tests__/*.test.js
```

No test framework dependency. Run from workspace root.

### 9.5 What to Test Per Script

| Script | Key Test Cases |
|---|---|
| `canon-classify` | Each classification path (label, project, description), override, default |
| `canon-gate` | Pass/fail per transition per workflow, --force override, missing prereqs |
| `canon-preflight` | Full/lightweight/off-canon paths, integration with classify + gate |
| `neo-log-decision` | Valid entry, invalid category rejection, file creation, JSON format |
| `wolverine-wrap` | Success passthrough, failure capture, dedup logic, ops issue creation |
| `wolverine-log-lesson` | Valid entry, file creation |
| `evolve-diff` | Changed/unchanged detection, diff truncation, mock git output |
| `evolve-log` | Valid entry, missing justification rejection |
| `log-writer` | Append, directory creation, read with filters, hash function |

### 9.6 Integration Tests (Phase 2)

After Phase 1 scripts are stable:
- End-to-end: classify → gate → preflight → (dev) → postflight against a real Linear issue
- Requires a dedicated test issue in Linear (created once, reused)

---

## 10. Security Considerations

- **Credentials:** Linear API key read from `secrets/linear-token.txt` — same as existing scripts. Never logged or output.
- **Log content:** Decision and lesson logs may contain issue descriptions. Logs stay in workspace (not committed to repo). `.gitignore` should include `logs/`.
- **Git commands:** Only read-only git operations (`diff`, `log`, `rev-parse`, `branch --list`). No writes.
- **Child process execution:** `wolverine-wrap.js` executes arbitrary scripts — same trust model as existing agent execution. Scripts are always within the workspace.

---

## 11. Migration & Rollout

### Phase 1 Implementation Order

1. **`lib/` shared modules** — foundation for everything else
2. **`neo-log-decision.js`** — simplest script, proves the pattern
3. **`wolverine-log-lesson.js`** — same pattern, different schema
4. **`evolve-diff.js` + `evolve-log.js`** — git-based, self-contained
5. **`canon-classify.js`** — requires Linear API, but no gating
6. **`canon-gate.js`** — the core enforcement piece
7. **`canon-preflight.js`** — composes classify + gate
8. **`wolverine-wrap.js`** — composes lesson logger + ops issue creation

### Integration Sequence

1. New scripts deployed and tested independently
2. `start-dev-session.js` updated to call `canon-preflight.js`
3. Agents updated to call `neo-log-decision.js` for autonomous decisions
4. `AGENTS.md` updated to reference scripts instead of pure prose

---

## 12. Constraints Verification

| Constraint | Satisfied | How |
|---|---|---|
| Standalone CommonJS Node.js | ✅ | All scripts are `#!/usr/bin/env node`, `require()` |
| Zero new dependencies | ✅ | Only `https`, `fs`, `path`, `child_process`, `crypto` |
| Output ≤50 lines JSON | ✅ | Envelope pattern, truncated diffs (500 char limit) |
| Linear API integration | ✅ | Shared `lib/linear-client.js`, same auth as existing |
| GitHub CLI integration | ✅ | Via `create-github-pr.js` (existing), git commands |
| OpenClaw sessions_spawn | ✅ | Scripts output JSON; agent interprets and spawns |
| Windows compatible | ✅ | No Unix-only commands; `child_process` for git |
| Backward compatible | ✅ | Existing scripts unchanged; new scripts extend via child_process |

---

*Architecture complete. Ready for Scrum Master phase to produce story files.*
