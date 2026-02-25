# CRU-170: Four Pillars Framework — Architecture Document

**Date:** 2026-02-23
**Author:** CruzBot (Architect subagent)
**Issue:** CRU-170
**Status:** Final
**Phase:** Architect (after Analyst + PM)

---

## 1. Directory Structure

```
workspace/
├── os/                                    # Four Pillars operating system root
│   ├── README.md                          # Always-injected summary (~40 lines, <2KB)
│   ├── config.yaml                        # Global OS config (injection rules, defaults)
│   │
│   ├── canon/                             # Pillar 1: The Workflow
│   │   ├── CANON.md                       # Definition: workflow tiers, classification rules
│   │   ├── config.yaml                    # Canon-specific config: signals, thresholds
│   │   └── log/                           # Runtime classification logs
│   │       └── YYYY-MM.jsonl              # Monthly rotation (e.g., 2026-02.jsonl)
│   │
│   ├── neo/                               # Pillar 2: The Autonomy Engine
│   │   ├── NEO.md                         # Definition: decision framework, boundaries
│   │   ├── config.yaml                    # Neo-specific: permission levels, categories
│   │   └── log/                           # Runtime decision logs
│   │       └── YYYY-MM.jsonl              # Monthly rotation
│   │
│   ├── wolverine/                         # Pillar 3: The Self-Healing Protocol
│   │   ├── WOLVERINE.md                   # Definition: improvement protocol, RL approach
│   │   ├── config.yaml                    # Wolverine-specific: severity levels, triggers
│   │   ├── lessons.jsonl                  # PERMANENT — never rotated, append-only
│   │   └── log/                           # Runtime incident/improvement logs
│   │       └── YYYY-MM.jsonl              # Monthly rotation
│   │
│   └── evolve/                            # Pillar 4: The Self-Evolution Engine
│       ├── EVOLVE.md                      # Definition: evolution tracking, DNA management
│       ├── config.yaml                    # Evolve-specific: tracked files, triggers
│       ├── changelog.jsonl                # PERMANENT — DNA change audit trail
│       └── log/                           # Runtime diff/change logs
│           └── YYYY-MM.jsonl              # Monthly rotation
│
├── scripts/os/                            # Tools (existing, updated paths)
│   ├── canon-classify.js                  # Classify issues → os/canon/log/
│   ├── neo-log-decision.js                # Log decisions → os/neo/log/
│   ├── wolverine-log-lesson.js            # Log lessons → os/wolverine/log/ + lessons.jsonl
│   ├── evolve-diff.js                     # Detect DNA changes (unchanged behavior)
│   ├── evolve-log.js                      # Log changes → os/evolve/log/ + changelog.jsonl
│   ├── os-query.js                        # NEW: search/filter across all pillar logs
│   ├── os-summary.js                      # NEW: generate context-efficient summaries
│   ├── lib/
│   │   ├── log-writer.js                  # Updated: path resolution via config
│   │   ├── schemas.js                     # Updated: JSONL validation schemas
│   │   ├── config-loader.js               # NEW: YAML config loader
│   │   ├── git-helpers.js                 # Unchanged
│   │   ├── linear-client.js               # Unchanged
│   │   └── update-linear.js               # Unchanged
│   └── __tests__/                         # Updated tests
│
├── AGENTS.md                              # Slimmed to ~150 lines
├── SOUL.md                                # Unchanged
├── IDENTITY.md                            # Slimmed: remove framework implementation details
├── MEMORY.md                              # Slimmed: remove framework state
├── USER.md                                # Unchanged
└── memory/                                # Unchanged
    └── YYYY-MM-DD.md
```

### Permanent vs Rotated Files

| File | Rotation | Rationale |
|------|----------|-----------|
| `os/wolverine/lessons.jsonl` | **Never** | Lessons are permanent institutional knowledge |
| `os/evolve/changelog.jsonl` | **Never** | DNA change audit trail must be complete |
| `os/{pillar}/log/YYYY-MM.jsonl` | **Monthly** | Operational logs, queryable by date range |

---

## 2. Data Schemas — JSONL Records

All JSONL records share a common envelope, then pillar-specific fields.

### 2.1 Common Envelope

Every record contains:

```typescript
{
  ts: string;          // ISO 8601 timestamp, e.g., "2026-02-23T21:11:44.123Z"
  system: string;      // "canon" | "neo" | "wolverine" | "evolve"
  type: string;        // Pillar-specific record type
  agent?: string;      // Agent session that wrote this (optional)
}
```

### 2.2 Canon — Classification Record

**File:** `os/canon/log/YYYY-MM.jsonl`

```typescript
{
  ts: string;
  system: "canon";
  type: "classify";
  issueId: string;           // e.g., "CRU-170"
  title: string;             // Issue title
  classification: string;    // "full-canon" | "lightweight" | "off-canon"
  reason: string;            // Why this classification
  override: boolean;         // Was this manually overridden?
  signals: {
    labels: string[];        // Linear labels found
    project: string | null;  // Linear project name
    estimatedFiles: number | null;
  };
  agent?: string;
}
```

**Example:**
```json
{"ts":"2026-02-23T21:11:44Z","system":"canon","type":"classify","issueId":"CRU-170","title":"Four Pillars Framework","classification":"full-canon","reason":"Label: full-canon","override":false,"signals":{"labels":["full-canon"],"project":"Cruz World","estimatedFiles":null}}
```

### 2.3 Neo — Decision Record

**File:** `os/neo/log/YYYY-MM.jsonl`

```typescript
{
  ts: string;
  system: "neo";
  type: "decision";
  decision: string;          // What was decided
  reasoning: string;         // Why
  category: string;          // "dev-sequence" | "technical-approach" | "model-routing" |
                             // "process-improvement" | "quality-gate" | "resource-allocation" |
                             // "escalation"
  alternatives?: string[];   // What else was considered
  alignment?: string;        // How it serves project goals
  override?: boolean;        // Was permission asked/granted?
  agent?: string;
}
```

**Example:**
```json
{"ts":"2026-02-23T15:00:00Z","system":"neo","type":"decision","decision":"Start CRU-170 with Full Canon","reasoning":"Core infrastructure, affects all sessions","category":"dev-sequence","alternatives":["Lightweight Canon","Defer to next sprint"],"alignment":"Reduces token waste, enables queryable state"}
```

### 2.4 Wolverine — Lesson Record

**Files:** `os/wolverine/lessons.jsonl` (permanent) + `os/wolverine/log/YYYY-MM.jsonl` (monthly)

Both files get the same record. The `lessons.jsonl` copy is the permanent archive.

```typescript
{
  ts: string;
  system: "wolverine";
  type: "lesson";
  trigger: string;           // What caused this lesson
  rootCause: string;         // Root cause analysis
  lesson: string;            // The lesson learned
  severity?: string;         // "process" | "technical" | "critical" (default: "process")
  tags?: string[];           // Categorization tags
  filesUpdated?: string[];   // Files modified as a result
  opsIssueId?: string;       // Associated Ops issue in Linear
  agent?: string;
}
```

**Example:**
```json
{"ts":"2026-02-23T10:00:00Z","system":"wolverine","type":"lesson","trigger":"CRU-145 Epic 6 branch contention","rootCause":"3 dev agents on same repo caused git conflicts","lesson":"Isolate concurrent agents into separate clones","severity":"critical","tags":["concurrency","git","subagents"],"filesUpdated":["AGENTS.md"],"opsIssueId":"CRU-168"}
```

### 2.5 Wolverine — Improvement Record (monthly log only)

**File:** `os/wolverine/log/YYYY-MM.jsonl`

```typescript
{
  ts: string;
  system: "wolverine";
  type: "improvement";
  category: string;          // "process-bug" | "efficiency" | "documentation" | "tool"
  description: string;       // What needs improving
  status: string;            // "identified" | "in-progress" | "resolved"
  opsIssueId?: string;       // Linear issue if created
  agent?: string;
}
```

### 2.6 Evolve — DNA Change Record

**Files:** `os/evolve/changelog.jsonl` (permanent) + `os/evolve/log/YYYY-MM.jsonl` (monthly)

```typescript
{
  ts: string;
  system: "evolve";
  type: "dna-change";
  file: string;              // Which DNA file changed (e.g., "AGENTS.md")
  diffSummary: string;       // What changed (max 500 chars)
  justification: string;     // Why it changed
  trigger: string;           // "user-request" | "self-improvement" | "wolverine-lesson" | "feedback"
  section?: string;          // Section of the file affected
  agent?: string;
}
```

**Example:**
```json
{"ts":"2026-02-23T12:00:00Z","system":"evolve","type":"dna-change","file":"AGENTS.md","diffSummary":"Added Concurrent Subagent Isolation section","justification":"CRU-168 lesson: concurrent agents cause branch contention","trigger":"wolverine-lesson","section":"Concurrent Subagent Isolation"}
```

---

## 3. YAML Config Schemas

### 3.1 Global Config — `os/config.yaml`

```yaml
# os/config.yaml — Global Four Pillars configuration
version: 1

# Context injection rules
injection:
  always:
    - os/README.md                    # Framework summary (~2KB)
  on_demand:
    - os/canon/CANON.md               # When doing workflow classification
    - os/neo/NEO.md                    # When making autonomous decisions
    - os/wolverine/WOLVERINE.md        # When analyzing failures
    - os/evolve/EVOLVE.md              # When modifying DNA files

# Log settings
logging:
  rotation: monthly                    # YYYY-MM.jsonl
  permanent_files:                     # Never rotated
    - os/wolverine/lessons.jsonl
    - os/evolve/changelog.jsonl
  max_monthly_size_mb: 10              # Warn if exceeded (informational)

# DNA files tracked by Evolve
dna_files:
  - AGENTS.md
  - SOUL.md
  - IDENTITY.md
  - TOOLS.md
  - USER.md
```

### 3.2 Canon Config — `os/canon/config.yaml`

```yaml
# os/canon/config.yaml — Canon workflow classification
version: 1

# Classification signals (priority order)
signals:
  - type: label
    match: "full-canon"
    result: full-canon
    
  - type: label
    match: ["off-canon", "hotfix"]
    result: off-canon
    
  - type: project
    match: ["ops", "docs"]
    result: lightweight
    
  - type: description
    match: "bug fix"
    result: off-canon
    
  - type: estimated_files
    max: 3
    result: off-canon

# Default when no signal matches
default: lightweight

# Valid classifications
classifications:
  - full-canon       # Analyst → PM → Architect → SM → Dev → QA → PR → Merge
  - lightweight      # PM → SM → Dev → PR → Merge
  - off-canon        # inject-standards → code → tests → PR → merge
```

### 3.3 Neo Config — `os/neo/config.yaml`

```yaml
# os/neo/config.yaml — Neo autonomy boundaries
version: 1

# What the agent decides autonomously
autonomous:
  - dev-sequence
  - technical-approach
  - model-routing
  - process-improvement
  - quality-gate
  - resource-allocation

# What requires explicit permission
requires_permission:
  - destructive-actions       # Delete repos, force push
  - external-communications   # Public posts, emails
  - budget-decisions          # Paid services, infra costs
  - policy-changes            # Safety rules, access controls

# Valid decision categories (for schema validation)
categories:
  - dev-sequence
  - technical-approach
  - model-routing
  - process-improvement
  - quality-gate
  - resource-allocation
  - escalation
```

### 3.4 Wolverine Config — `os/wolverine/config.yaml`

```yaml
# os/wolverine/config.yaml — Wolverine self-healing
version: 1

# Severity levels
severities:
  - process       # Workflow/convention issues
  - technical     # Code/tooling issues  
  - critical      # P1 — no workarounds, root-cause fix only

# Improvement categories
improvement_categories:
  - process-bug
  - efficiency
  - documentation
  - tool

# P1 policy
critical_policy:
  workarounds_allowed: false
  requires_permanent_fix: true
```

### 3.5 Evolve Config — `os/evolve/config.yaml`

```yaml
# os/evolve/config.yaml — Evolve self-evolution
version: 1

# Valid triggers for DNA changes
triggers:
  - user-request
  - self-improvement
  - wolverine-lesson
  - feedback

# Max diff summary length
max_diff_length: 500
```

---

## 4. Markdown Definition File Templates

### 4.1 `os/README.md` — Always-Injected Summary

This is the critical file — injected every session. Must be concise.

```markdown
# Agent OS — Four Pillars

Four frameworks define how CruzBot operates. Definitions in `os/{pillar}/`,
logs in `os/{pillar}/log/`, config in `os/{pillar}/config.yaml`.

## Canon (The Workflow)
The canonical way to build software. Full / Lightweight / Off-Canon.
→ `os/canon/CANON.md` | Query: `node scripts/os/os-query.js --system canon`

## Neo (The Autonomy Engine)
Decide and execute. Log decisions with reasoning + alternatives.
→ `os/neo/NEO.md` | Query: `node scripts/os/os-query.js --system neo`

## Wolverine (The Self-Healing Protocol)
Learn from failures. Try → Observe → Analyze → Learn → Update.
→ `os/wolverine/WOLVERINE.md` | Lessons: `os/wolverine/lessons.jsonl`

## Evolve (The Self-Evolution Engine)
Track DNA changes. Every edit to core files gets logged with justification.
→ `os/evolve/EVOLVE.md` | Changelog: `os/evolve/changelog.jsonl`

## Quick Reference
- Log a decision: `node scripts/os/neo-log-decision.js --decision "..." --reasoning "..." --category "..."`
- Log a lesson: `node scripts/os/wolverine-log-lesson.js --trigger "..." --root-cause "..." --lesson "..."`
- Classify an issue: `node scripts/os/canon-classify.js CRU-XXX`
- Log a DNA change: `node scripts/os/evolve-log.js --file "..." --diff-summary "..." --justification "..." --trigger "..."`
- Query logs: `node scripts/os/os-query.js --system neo --since 7d --limit 10`
- Session summary: `node scripts/os/os-summary.js`
```

**Target:** ~35 lines, ~1.5KB, ~400 tokens.

### 4.2 `os/canon/CANON.md`

```markdown
# Canon — The Workflow

Canon is the canonical workflow for building software. It exists to prevent
cutting corners on complex work while allowing speed on simple tasks.

## Workflow Tiers

### Full Canon
Analyst → PM → Architect → SM → Dev → QA → PR Review → Merge

**Use for:** Medium/large features, architecture changes, core infrastructure.

### Lightweight Canon
PM → SM → Dev → PR Review → Merge

**Use for:** Small features, ops tasks, documentation, anything that doesn't
need deep analysis or architecture.

### Off-Canon
`/inject-standards` → Code → Tests → PR Review → Merge

**Use for:** Bug fixes (< 3 files), hotfixes, emergencies, quick config changes.

## Classification
Issues are classified by `canon-classify.js` using signals from Linear
(labels, project, description). Config: `os/canon/config.yaml`.

## Checkpoints
- [ ] After PM: Epics document exists
- [ ] After Architect: Architecture document exists
- [ ] After SM: Story files exist, Linear sub-issues created
- [ ] Before dev: Linear issues exist
- [ ] Every dev session: `/inject-standards` run first
- [ ] Every PR: References Linear issue ID

## Non-Negotiable
- Never write code without a story file OR `/inject-standards`
- Standards FIRST, code SECOND
- Stories are authoritative — read before writing
- Tests before marking done
- Fresh sessions — planning and dev agents never share sessions
```

### 4.3 `os/neo/NEO.md`

```markdown
# Neo — The Autonomy Engine

Neo governs autonomous decision-making. Make decisions and execute.
Don't ask permission you've already been given.

## Decision Framework

### Decide Autonomously
1. Development sequence — which story/epic next
2. Technical approach — how to implement (within standards)
3. Model routing — which model for which task
4. Process improvements — workflow optimizations
5. Quality gates — when to approve/reject PRs
6. Resource allocation — spawning agents, parallel work

### Require Permission
1. Destructive actions — deleting repos, force pushing
2. External communications — public posts, emails
3. Budget decisions — paid services, infrastructure
4. Policy changes — safety rules, access controls
5. When uncertain — if not confident, ask

## Decision Logging
Every significant decision gets logged via `neo-log-decision.js` with:
- **Decision:** What was chosen
- **Reasoning:** Why
- **Alternatives:** What else was considered
- **Category:** dev-sequence, technical-approach, etc.

## Execution Discipline
- If you state a plan, execute it. Period.
- Hesitation after a decision is asking permission twice.
- Context switching kills momentum — finish current work first.
- Trust is earned through follow-through.

## Proactive Execution
- When a subagent completes → spawn the next step, don't wait
- During heartbeats → check for pending work before monitoring
- The plan was approved — execution doesn't need re-approval
```

### 4.4 `os/wolverine/WOLVERINE.md`

```markdown
# Wolverine — The Self-Healing Protocol

Wolverine is the continuous improvement engine. Learn from failures,
fix process bugs, self-heal through reinforcement learning.

## Protocol
Try → Observe → Analyze → Learn → Update → Create Ops Issue

## Improvement Tracking
Track as Ops issues in Linear:
- 🐛 Process bugs (workflow breaks)
- ⚡ Efficiency gains (automation opportunities)
- 📚 Documentation gaps
- 🧠 Learning moments (RL-style insights)
- 🔧 Tool improvements

## Reinforcement Learning
1. **Document** the experience
2. **Analyze** root cause
3. **Learn** the lesson
4. **Update** relevant files
5. **Create Ops issue** if it affects process

## Severity Levels
- **process** — workflow/convention issues
- **technical** — code/tooling issues
- **critical (P1)** — no workarounds, permanent root-cause fix only

## Lessons
Permanent lessons are stored in `os/wolverine/lessons.jsonl`.
These survive log rotation and form institutional memory.

## Pattern Recognition
- What works → codify in SOPs
- What fails → fix and prevent recurrence
- What's ambiguous → clarify and document
- Edge cases → document handling
```

### 4.5 `os/evolve/EVOLVE.md`

```markdown
# Evolve — The Self-Evolution Engine

Evolve tracks, documents, and manages changes to CruzBot's core DNA.
Every modification to identity or operating system files is recorded.

## What Gets Tracked
DNA files: AGENTS.md, SOUL.md, IDENTITY.md, TOOLS.md, USER.md

## Change Types
- Core DNA changes (AGENTS.md, SOUL.md, IDENTITY.md)
- Operating system refinements (Canon, Neo, Wolverine configs)
- New skill acquisition or skill updates
- User feedback integration

## Change Triggers
- `user-request` — Tony asked for a change
- `self-improvement` — Agent identified improvement
- `wolverine-lesson` — Lesson led to DNA update
- `feedback` — External feedback integration

## Audit Trail
- `os/evolve/changelog.jsonl` — permanent, complete change history
- `os/evolve/log/YYYY-MM.jsonl` — monthly operational logs
- `evolve-diff.js` — detects uncommitted DNA changes via git

## Process
1. Detect change (git diff or manual trigger)
2. Log with justification and trigger
3. Append to changelog.jsonl (permanent)
4. Append to monthly log
```

---

## 5. Script Integration Plan

### 5.1 Current State → Target State

| Script | Current Output Path | Target Output Path | Changes Needed |
|--------|-------------------|-------------------|----------------|
| `canon-classify.js` | stdout only (no file write) | `os/canon/log/YYYY-MM.jsonl` + stdout | Add file write via log-writer |
| `neo-log-decision.js` | `logs/decisions.jsonl` | `os/neo/log/YYYY-MM.jsonl` | Update path resolution |
| `wolverine-log-lesson.js` | `logs/lessons.jsonl` | `os/wolverine/lessons.jsonl` + `os/wolverine/log/YYYY-MM.jsonl` | Dual write, update paths |
| `evolve-log.js` | `logs/evolution.jsonl` | `os/evolve/changelog.jsonl` + `os/evolve/log/YYYY-MM.jsonl` | Dual write, update paths |
| `evolve-diff.js` | stdout only | stdout only (unchanged) | No changes needed |
| `os-query.js` | N/A (new) | stdout | New script |
| `os-summary.js` | N/A (new) | stdout | New script |

### 5.2 `lib/log-writer.js` Changes

Current `defaultLogsDir()` returns `~/.openclaw/workspace/logs`. This changes to be config-driven.

```javascript
// NEW: resolve log path based on system + type
function resolveLogPath(system, options = {}) {
  const wsRoot = path.join(
    process.env.USERPROFILE || process.env.HOME || '',
    '.openclaw', 'workspace'
  );
  const osRoot = path.join(wsRoot, 'os');
  const now = new Date();
  const month = `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, '0')}`;

  // Monthly log
  const monthlyPath = path.join(osRoot, system, 'log', `${month}.jsonl`);

  // Permanent files (if applicable)
  let permanentPath = null;
  if (system === 'wolverine' && options.permanent) {
    permanentPath = path.join(osRoot, 'wolverine', 'lessons.jsonl');
  }
  if (system === 'evolve' && options.permanent) {
    permanentPath = path.join(osRoot, 'evolve', 'changelog.jsonl');
  }

  return { monthlyPath, permanentPath };
}

// NEW: dual append (monthly + optional permanent)
function appendToSystem(system, entry, options = {}) {
  const { monthlyPath, permanentPath } = resolveLogPath(system, options);
  append(monthlyPath, entry);
  if (permanentPath) {
    append(permanentPath, entry);
  }
  return { monthlyPath, permanentPath };
}
```

**Backward compatibility:** Keep `defaultLogsDir()` and old `append()` working during migration. Remove in Epic 3 cleanup.

### 5.3 `lib/config-loader.js` (New)

```javascript
const fs = require('fs');
const path = require('path');
const yaml = require('yaml'); // OR use simple YAML parser (see §10 dependencies)

function loadConfig(pillar) {
  const wsRoot = path.join(
    process.env.USERPROFILE || process.env.HOME || '',
    '.openclaw', 'workspace'
  );
  const configPath = pillar
    ? path.join(wsRoot, 'os', pillar, 'config.yaml')
    : path.join(wsRoot, 'os', 'config.yaml');

  if (!fs.existsSync(configPath)) {
    return null; // Graceful: return null, caller uses defaults
  }
  return yaml.parse(fs.readFileSync(configPath, 'utf8'));
}
```

**Dependency note:** The `yaml` npm package is the standard choice. If zero-dependency is required, use a minimal inline YAML parser or read YAML as text and parse with regex for the simple config structures we have. **Recommendation:** Allow `yaml` as the single external dependency — it's 45KB, well-maintained, and avoids fragile custom parsing.

### 5.4 `lib/schemas.js` Changes

Add JSONL record validation:

```javascript
// Add to existing schemas.js
const WOLVERINE_SEVERITIES = ['process', 'technical', 'critical'];
const IMPROVEMENT_CATEGORIES = ['process-bug', 'efficiency', 'documentation', 'tool'];

function validateRecord(record) {
  if (!record.ts || !record.system || !record.type) {
    return { valid: false, error: 'Missing required envelope fields: ts, system, type' };
  }
  if (!SYSTEMS.includes(record.system)) {
    return { valid: false, error: `Invalid system: ${record.system}` };
  }
  return { valid: true };
}
```

### 5.5 Script Changes Summary

**`canon-classify.js`:**
- Add: Import `appendToSystem` from log-writer
- Add: After classification, call `appendToSystem('canon', entry)` to write monthly log
- Keep: stdout JSON output unchanged (backward compatible)

**`neo-log-decision.js`:**
- Change: Replace `defaultLogsDir() + '/decisions.jsonl'` with `appendToSystem('neo', entry)`
- Keep: stdin/CLI arg parsing unchanged

**`wolverine-log-lesson.js`:**
- Change: Replace `defaultLogsDir() + '/lessons.jsonl'` with `appendToSystem('wolverine', entry, { permanent: true })`
- Effect: Writes to both `os/wolverine/lessons.jsonl` AND `os/wolverine/log/YYYY-MM.jsonl`

**`evolve-log.js`:**
- Change: Replace `defaultLogsDir() + '/evolution.jsonl'` with `appendToSystem('evolve', entry, { permanent: true })`
- Effect: Writes to both `os/evolve/changelog.jsonl` AND `os/evolve/log/YYYY-MM.jsonl`

**`evolve-diff.js`:**
- No changes needed (reads git state, writes to stdout)

### 5.6 New Scripts

**`os-query.js`** — Query across all pillar logs

```
Usage: os-query.js [options]
  --system <canon|neo|wolverine|evolve>   Filter by pillar
  --type <type>                           Filter by record type
  --since <Nd|Nw|Nm|ISO-date>            Time filter (e.g., 7d, 2w, 2026-01-01)
  --until <ISO-date>                      End date filter
  --tag <tag>                             Filter wolverine lessons by tag
  --severity <level>                      Filter by severity
  --limit <N>                             Max results (default: 20)
  --format <json|table|summary>           Output format (default: json)
```

**`os-summary.js`** — Generate context-efficient summary for session injection

```
Usage: os-summary.js [options]
  --days <N>         Look back N days (default: 7)
  --max-tokens <N>   Target token budget (default: 1000)
  --format <md|json> Output format (default: md)
```

Output: A brief Markdown summary of recent activity across all pillars, suitable for context injection.

---

## 6. Context Injection Strategy

### 6.1 Token Budget

| Content | Location | Injection | Est. Size | Est. Tokens |
|---------|----------|-----------|-----------|-------------|
| Workspace rules | AGENTS.md (slimmed) | **Always** | ~5KB | ~1,300 |
| Framework summary | os/README.md | **Always** | ~1.5KB | ~400 |
| Personality | SOUL.md | **Always** | ~3KB | ~800 |
| Identity | IDENTITY.md (slimmed) | **Always** | ~2KB | ~500 |
| User context | USER.md | **Always** | ~1KB | ~250 |
| **Always total** | | | **~12.5KB** | **~3,250** |
| | | | | |
| Canon definition | os/canon/CANON.md | On-demand | ~2KB | ~500 |
| Neo definition | os/neo/NEO.md | On-demand | ~2KB | ~500 |
| Wolverine definition | os/wolverine/WOLVERINE.md | On-demand | ~1.5KB | ~400 |
| Evolve definition | os/evolve/EVOLVE.md | On-demand | ~1.5KB | ~400 |
| Recent activity summary | os-summary.js output | On-demand | ~2KB | ~500 |
| **On-demand total** | | | **~9KB** | **~2,300** |

**Current always-injected total:** ~50K+ chars (~13K+ tokens)
**Target always-injected total:** ~12.5KB (~3,250 tokens)
**Savings:** ~37.5KB chars, ~10K tokens per session

### 6.2 Injection Rules

**Always (every session):**
1. `AGENTS.md` (slimmed ~150 lines) — workspace rules, pointers
2. `os/README.md` — framework overview + quick reference commands
3. `SOUL.md` — personality (unchanged)
4. `IDENTITY.md` (slimmed) — identity minus framework details
5. `USER.md` — human context (unchanged)

**On-demand (agent reads when needed):**
- Pillar `.md` files — when performing that pillar's work
- YAML configs — when scripts need configuration
- JSONL logs — via `os-query.js` or direct read
- `os-summary.js` output — during heartbeats or session start if available

**OpenClaw integration (future):** Configure `openclaw.json` workspace files to inject `os/README.md`. Currently controlled by whatever files are listed in OpenClaw's config.

### 6.3 How On-Demand Loading Works

No special tooling needed. The agent already has `Read` tool access. The key enabler is:

1. `os/README.md` tells the agent what exists and where
2. Agent reads pillar files when contextually relevant
3. Agent runs query scripts when it needs historical data

This is the simplest approach that works. Letta-style self-managed memory or RAG indexing is deferred.

---

## 7. Migration Plan

### Phase 1: Foundation (Epic 1) — Non-Breaking, Additive Only

**Goal:** Create the `os/` structure. Nothing is removed or changed in existing files.

| Step | Action | Parallelizable | Rollback |
|------|--------|---------------|----------|
| 1.1 | Create `os/` directory tree (all subdirs, `log/` dirs) | — | Delete `os/` |
| 1.2 | Write `os/README.md` | Yes (with 1.3-1.6) | Delete file |
| 1.3 | Write `os/config.yaml` | Yes | Delete file |
| 1.4 | Extract Canon definition → `os/canon/CANON.md` + `config.yaml` | Yes | Delete files |
| 1.5 | Extract Neo definition → `os/neo/NEO.md` + `config.yaml` | Yes | Delete files |
| 1.6 | Extract Wolverine definition → `os/wolverine/WOLVERINE.md` + `config.yaml` | Yes | Delete files |
| 1.7 | Extract Evolve definition → `os/evolve/EVOLVE.md` + `config.yaml` | Yes | Delete files |

**Validation:** All files exist, are well-formed, content matches current AGENTS.md prose.
**Risk:** Zero — purely additive.

### Phase 2: Runtime Logging (Epic 2) — Update Scripts

| Step | Action | Parallelizable | Rollback |
|------|--------|---------------|----------|
| 2.1 | Add `lib/config-loader.js` | — | Delete file |
| 2.2 | Update `lib/log-writer.js` (add `resolveLogPath`, `appendToSystem`) | After 2.1 | Revert file |
| 2.3 | Update `lib/schemas.js` (add validation, new constants) | Yes (with 2.2) | Revert file |
| 2.4 | Update `canon-classify.js` to write to `os/canon/log/` | After 2.2 | Revert file |
| 2.5 | Update `neo-log-decision.js` to write to `os/neo/log/` | After 2.2 | Revert file |
| 2.6 | Update `wolverine-log-lesson.js` (dual write) | After 2.2 | Revert file |
| 2.7 | Update `evolve-log.js` (dual write) | After 2.2 | Revert file |
| 2.8 | Update all tests | After 2.4-2.7 | Revert tests |
| 2.9 | Backfill: migrate existing `logs/*.jsonl` → `os/` structure | After 2.2 | Delete migrated files |

**Steps 2.4–2.7 are parallelizable** (independent scripts, shared dependency on 2.2).
**Validation:** Run all tests. Run each script manually, verify JSONL appears in correct paths.
**Risk:** Low — old `logs/` directory can coexist. Scripts can dual-write during transition.

### Phase 3: Core Doc Slimming (Epic 3) — The Cut

| Step | Action | Parallelizable | Rollback |
|------|--------|---------------|----------|
| 3.1 | Slim AGENTS.md (replace framework prose with pointers) | — | Git revert |
| 3.2 | Slim IDENTITY.md (remove framework details) | Yes (with 3.1) | Git revert |
| 3.3 | Slim MEMORY.md (remove framework state) | Yes (with 3.1) | Git revert |
| 3.4 | Update OpenClaw config to inject `os/README.md` | After 3.1 | Revert config |
| 3.5 | Validate context injection total is within budget | After 3.4 | — |

**Validation:** Start a fresh session, verify agent has framework awareness, can find and read pillar files.
**Risk:** Medium — this is where content moves. Git provides rollback. Phase 1+2 being validated first de-risks this.

### Phase 4: Query Tools (Epic 4) — Pure Additive

| Step | Action | Parallelizable | Rollback |
|------|--------|---------------|----------|
| 4.1 | Build `os-query.js` | Yes (with 4.2) | Delete file |
| 4.2 | Build `os-summary.js` | Yes (with 4.1) | Delete file |
| 4.3 | Write tests for query/summary tools | After 4.1-4.2 | Delete tests |
| 4.4 | Clean up old `logs/` directory | After all validation | Restore from git |

**Risk:** Zero — additive tools.

### Rollback Strategy

Every phase has file-level rollback:
- **Phase 1:** `rm -rf os/` — back to original
- **Phase 2:** `git checkout scripts/os/` — scripts revert to old paths
- **Phase 3:** `git checkout AGENTS.md IDENTITY.md MEMORY.md` — docs restored
- **Phase 4:** Delete new scripts

**Nuclear rollback:** `git checkout HEAD~N -- .` restores everything.

---

## 8. AGENTS.md Slim-Down Plan

### Current: 502 lines, ~28K chars

### Target: ~150 lines, ~5K chars

### What Stays in AGENTS.md

| Section | Lines | Rationale |
|---------|-------|-----------|
| Header + First Run | 8 | Still needed |
| Every Session (read order) | 12 | Boot sequence |
| Core Operating Systems (brief) | 15 | Four 2-line summaries + pointer to `os/README.md` |
| Subagent Task Scoping | 12 | Operational rule |
| Memory | 15 | Memory protocol |
| Concurrent Subagent Isolation | 10 | Operational rule (CRU-168) |
| Exec / Command Execution | 12 | Operational rule |
| Safety | 5 | Non-negotiable |
| Gateway Restart Protocol | 8 | Non-negotiable |
| External vs Internal | 8 | Safety boundary |
| Group Chats (brief) | 10 | Behavioral rule |
| Canon Workflow (brief) | 15 | Quick reference table + pointer |
| PR Review Gate | 20 | Process-critical |
| **Total** | **~150** | |

### What Moves to `os/`

| Current AGENTS.md Section | Moves To | Lines Freed |
|--------------------------|----------|-------------|
| Neo: The Autonomy Engine (full) | `os/neo/NEO.md` | ~50 |
| Wolverine: The Self-Healing Protocol (full) | `os/wolverine/WOLVERINE.md` | ~40 |
| Proactive Execution (CRU-128) detail | `os/neo/NEO.md` | ~20 |
| Execution Discipline detail | `os/neo/NEO.md` | ~15 |
| Continuous Improvement Protocol detail | `os/wolverine/WOLVERINE.md` | ~25 |
| Status Updates detail | `os/neo/NEO.md` | ~15 |
| Canon workflow tiers (detailed) | `os/canon/CANON.md` | ~30 |
| Canon checkpoints | `os/canon/CANON.md` | ~10 |
| The Stack (BMAD + Agent OS) | `os/canon/CANON.md` | ~10 |
| Non-Negotiable Rules (detailed) | `os/canon/CANON.md` | ~8 |
| **Total freed** | | **~223** |

### Slim AGENTS.md Core Operating Systems Section (replacement)

```markdown
## Core Operating Systems

Four pillars define how you operate. Full details: `os/README.md`

| Pillar | Purpose | Definition |
|--------|---------|------------|
| **Canon** | The canonical workflow (Full / Lightweight / Off-Canon) | `os/canon/CANON.md` |
| **Neo** | Autonomous decision-making + execution | `os/neo/NEO.md` |
| **Wolverine** | Self-healing, continuous improvement | `os/wolverine/WOLVERINE.md` |
| **Evolve** | Track DNA changes + self-evolution | `os/evolve/EVOLVE.md` |

**Default to Canon.** Log decisions (Neo). Learn from failures (Wolverine). Track changes (Evolve).
```

This replaces ~150 lines of inline prose with ~10 lines + a pointer.

---

## 9. File Naming Conventions

### JSONL Log Files

**Pattern:** `YYYY-MM.jsonl`
**Examples:** `2026-02.jsonl`, `2026-03.jsonl`
**Location:** `os/{pillar}/log/`
**Rotation:** New file created on first write of each calendar month

### Permanent Files

| File | Pattern | Location |
|------|---------|----------|
| Wolverine lessons | `lessons.jsonl` | `os/wolverine/` |
| Evolve changelog | `changelog.jsonl` | `os/evolve/` |

### Definition Files

**Pattern:** `{PILLAR}.md` (uppercase pillar name)
**Examples:** `CANON.md`, `NEO.md`, `WOLVERINE.md`, `EVOLVE.md`

### Config Files

**Pattern:** `config.yaml`
**Location:** `os/config.yaml` (global) or `os/{pillar}/config.yaml` (per-pillar)

### Month String Generation

```javascript
function currentMonth() {
  const now = new Date();
  return `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, '0')}`;
}
// Output: "2026-02"
```

---

## 10. Error Handling

### Missing Files

| Scenario | Behavior |
|----------|----------|
| `os/` directory doesn't exist | Scripts create it on first write (recursive mkdir) |
| `os/{pillar}/log/` doesn't exist | `log-writer.js` creates it on first append |
| Config YAML doesn't exist | `config-loader.js` returns `null`; scripts use hardcoded defaults |
| Permanent JSONL doesn't exist | Created on first append |
| Definition .md doesn't exist | Agent gets null from Read; framework still works via AGENTS.md fallback |
| `os/README.md` doesn't exist | AGENTS.md still contains brief framework summary (graceful degradation) |

### Corrupt JSONL

| Scenario | Behavior |
|----------|----------|
| Malformed line in JSONL | `log-writer.read()` already skips unparseable lines (`JSON.parse` try/catch returns null, filtered out) |
| Truncated last line (crash during write) | Same — skipped by parser. Data loss limited to that one record. |
| Empty file | `read()` returns `[]` — valid empty result |
| Binary garbage | All lines fail parse → empty result. File preserved for manual inspection. |

### YAML Parse Errors

| Scenario | Behavior |
|----------|----------|
| Malformed YAML | `config-loader.js` catches parse error, logs warning to stderr, returns `null` |
| Missing required field in config | Scripts use hardcoded defaults for any missing field |
| Invalid enum value in config | Schema validation at script level; warn and use default |

### Write Failures

| Scenario | Behavior |
|----------|----------|
| Disk full | `appendFileSync` throws → caught, error logged to stderr, script exits with error JSON |
| Permission denied | Same — caught, error JSON output |
| Path too long (Windows) | Mitigated by short path structure (`os/neo/log/2026-02.jsonl` = 27 chars relative) |

### Concurrent Writes

At CruzBot's scale (single agent, sequential operations), concurrent write contention is not expected. If two subagents write to the same JSONL simultaneously:
- `appendFileSync` is atomic for small writes (< PIPE_BUF, typically 4KB) on most OS
- JSONL records are independent lines — interleaving is safe as long as individual lines aren't split
- **Mitigation if needed (future):** File locking via `proper-lockfile` npm package

### Recovery

- **Corrupt JSONL:** Delete the corrupt monthly file. Permanent files (`lessons.jsonl`, `changelog.jsonl`) should be hand-repaired if corrupt (they're too valuable to lose).
- **Missing config:** System works with defaults. Recreate from templates in this architecture doc.
- **Missing definitions:** Recreate from templates in Section 4 of this doc.

---

## 11. Dependencies

### External (npm)

| Package | Purpose | Size | Required By |
|---------|---------|------|-------------|
| `yaml` | YAML config parsing | ~45KB | `lib/config-loader.js` |

**Alternative:** If zero external deps is mandated, implement a minimal YAML parser for the simple flat/nested structures in our configs. Not recommended — fragile and time-wasting.

### Internal (existing)

All existing dependencies (`fs`, `path`, `crypto`, `child_process`) remain. No new Node.js built-in dependencies.

---

## 12. Open Questions (Resolved)

| Question | Resolution |
|----------|------------|
| `os/` location | Workspace root, peer to `scripts/`, `memory/` |
| Config format | YAML (human-editable, supports comments) |
| Log rotation | Monthly JSONL (`YYYY-MM.jsonl`) |
| Config editing | Hand-edited initially; tool-based editing deferred |
| JSONL schema validation | At write time in scripts; `validateRecord()` in schemas.js |

---

## 13. Story Mapping to Epics

| Epic | Stories | Dependencies |
|------|---------|-------------|
| **1: Foundation** | Create dir tree; Write README.md; Extract 4 pillar .md files; Write 5 config.yaml files | None |
| **2: Runtime Logging** | config-loader.js; Update log-writer.js; Update 4 scripts; New tests; Backfill existing logs | Epic 1 |
| **3: Core Doc Slimming** | Slim AGENTS.md; Slim IDENTITY.md; Slim MEMORY.md; Update OpenClaw injection config | Epic 1 + 2 |
| **4: Query Tools** | os-query.js; os-summary.js; Tests; Clean up old logs/ | Epic 2 |

**Critical path:** Epic 1 → Epic 2 → Epic 3. Epic 4 can start after Epic 2.
**Parallelization:** Within each epic, stories are largely parallelizable (see §7 migration plan).
