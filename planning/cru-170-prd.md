# CRU-170 PRD: Four Pillars Framework Structured Runtime System

**Date:** 2026-02-23  
**Author:** CruzBot (PM subagent)  
**Issue:** CRU-170  
**Status:** Draft  
**Canon:** Full (Analyst → PM → Architect → SM → Dev → QA → PR → Merge)

---

## 1. Problem Statement

AGENTS.md is 502 lines / ~28K chars. OpenClaw injects it into every session — and already truncates it. The Four Pillars (Canon, Neo, Wolverine, Evolve) are embedded as prose, mixing **what they are** (definitions), **how they're configured** (rules), and **what happened** (runtime state) into a single monolithic file.

**Three concrete problems:**

1. **Context bloat.** ~50K+ chars of system context before conversation starts. Framework definitions that don't change session-to-session waste tokens every time.
2. **No queryable state.** "What did Wolverine learn last week?" requires scanning Markdown prose in MEMORY.md. There's no structured data to filter, search, or aggregate.
3. **Scattered framework content.** Framework logic lives in AGENTS.md, IDENTITY.md, MEMORY.md, and inline in scripts. No single source of truth per pillar.

**Impact:** CruzBot operates with a bloated, unqueryable, tangled context that degrades quality and wastes ~5K tokens per session on static prose that could be loaded on-demand.

---

## 2. Goals & Success Metrics

| Goal | Metric | Target |
|------|--------|--------|
| Reduce always-injected context | AGENTS.md line count | ≤ 150 lines (~5K chars), down from 502 |
| Reduce total system context injection | Total chars injected at session start | ≤ 30K chars (from ~50K+) |
| Token savings per session | Estimated tokens freed | ~5K tokens |
| Structured runtime state | Framework events queryable via script | 100% of new Canon/Neo/Wolverine/Evolve events written as JSONL |
| Single source of truth | Each pillar has exactly one definition file | 4 Markdown files in `os/` |
| No regression | Existing workflows unbroken | All scripts pass, daily notes unaffected |
| Human editability preserved | Tony can edit framework defs in any text editor | Markdown definitions, YAML config |

---

## 3. User Stories

### CruzBot (Agent) Perspective

| ID | Story | Acceptance Criteria |
|----|-------|-------------------|
| A1 | As CruzBot, I want framework definitions loaded only when relevant, so my context window isn't wasted on static prose every session. | AGENTS.md contains pointers, not full definitions. Pillar .md files loaded on-demand. |
| A2 | As CruzBot, I want to append structured runtime events (decisions, lessons, classifications) without parsing/rewriting Markdown. | JSONL append via scripts; no read-modify-write. |
| A3 | As CruzBot, I want to query "what did I learn last week?" and get structured results. | `os-query.js` returns filtered JSONL entries by date, pillar, tag. |
| A4 | As CruzBot, I want a brief framework summary always in context so I know the pillars exist and where to find details. | `os/README.md` (~30 lines) injected every session. |
| A5 | As CruzBot, I want Wolverine lessons to persist permanently and never be lost to log rotation. | `os/wolverine/lessons.jsonl` is append-only, never rotated. |

### Tony (Human) Perspective

| ID | Story | Acceptance Criteria |
|----|-------|-------------------|
| H1 | As Tony, I want to edit framework definitions in my text editor without breaking the system. | Definitions are plain Markdown. No special syntax required. |
| H2 | As Tony, I want to see what CruzBot has learned/decided via readable summaries. | `os-summary.js` generates human-readable digests from JSONL. |
| H3 | As Tony, I want AGENTS.md to be concise and scannable, not a wall of text. | ≤ 150 lines, clear sections, pointers to detail files. |
| H4 | As Tony, I want framework config (what needs permission, classification signals) in a structured format I can review. | YAML config files in `os/` or per-pillar. |

---

## 4. Requirements

### Functional

| ID | Requirement | Priority |
|----|-------------|----------|
| F1 | Create `os/` directory structure with per-pillar subdirectories | P0 |
| F2 | Extract pillar definitions from AGENTS.md into `os/{pillar}/{PILLAR}.md` | P0 |
| F3 | Create YAML config files for framework parameters/rules | P1 |
| F4 | Define JSONL schemas for each pillar's runtime events | P0 |
| F5 | Update/create scripts to write runtime events to JSONL | P0 |
| F6 | Slim AGENTS.md to ≤150 lines with pointers to `os/` | P0 |
| F7 | Create `os/README.md` as brief always-injected summary | P0 |
| F8 | Build `os-query.js` — search/filter across pillar logs | P1 |
| F9 | Build `os-summary.js` — generate session-relevant summaries | P1 |
| F10 | Backfill existing runtime state from MEMORY.md into JSONL | P1 |
| F11 | Slim IDENTITY.md and MEMORY.md (remove framework content) | P1 |

### Non-Functional

| ID | Requirement | Target |
|----|-------------|-------|
| NF1 | JSONL append latency | < 5ms per write |
| NF2 | Query across 1 month of logs | < 1 second |
| NF3 | No breaking changes during migration | Phase 1 is additive only |
| NF4 | All files git-friendly | Meaningful diffs, no binary blobs |
| NF5 | Zero external dependencies | Node.js stdlib + existing workspace tools only |

---

## 5. Key Decisions

### Format: Hybrid Markdown + JSONL + YAML (Confirmed)

The analyst brief evaluated pure Markdown, pure JSONL, SQLite, and hybrid approaches against 8 criteria. **Hybrid MD + JSONL + YAML wins decisively** for CruzBot's scale and needs:

- **Markdown** for definitions — LLM-native, human-editable, Tony's domain
- **JSONL** for runtime logs — append-only, queryable, date-rotatable, ~0.75ms writes
- **YAML** for configuration — structured, human-readable, widely understood

SQLite is unnecessary at single-agent / <10K events/month scale. This decision is **closed**.

### Directory: `os/` at workspace root

Peer to `scripts/`, `docs/`, `memory/`. The pillars are an operating system — `os/` is the natural name.

### Log rotation: Monthly JSONL files

Format: `os/{pillar}/log/YYYY-MM.jsonl`. Keeps files small, easy to grep by date range. Wolverine `lessons.jsonl` is the exception — permanent, never rotated.

### Config format: YAML over JSON

YAML supports comments, is more readable for config, and Tony already knows it. JSON for schemas/runtime; YAML for human-edited config.

---

## 6. Epics Breakdown

### Epic 1: Foundation — Create Structure & Definitions
- Create `os/` directory tree
- Extract Canon definition → `os/canon/CANON.md`
- Extract Neo definition → `os/neo/NEO.md`
- Extract Wolverine definition → `os/wolverine/WOLVERINE.md`
- Extract Evolve definition → `os/evolve/EVOLVE.md`
- Create `os/README.md` (brief summary + pointers)
- Create YAML config files per pillar
- Define JSONL schemas (documented in each pillar's README or schema file)

### Epic 2: Runtime Logging — Scripts & JSONL
- Create/update `scripts/os/canon-classify.js` → writes to `os/canon/log/`
- Create/update `scripts/os/neo-log-decision.js` → writes to `os/neo/log/`
- Create/update `scripts/os/wolverine-log-lesson.js` → writes to `os/wolverine/log/` + `lessons.jsonl`
- Create/update `scripts/os/evolve-log.js` → writes to `os/evolve/log/` + `changelog.jsonl`
- Validate JSONL schema on write

### Epic 3: Core Doc Slimming
- Slim AGENTS.md to ≤150 lines (replace framework prose with pointers)
- Slim IDENTITY.md (remove framework implementation details)
- Slim MEMORY.md (migrate framework state to JSONL)
- Backfill existing runtime data from MEMORY.md → JSONL

### Epic 4: Query & Summary Tools
- Build `os-query.js` (filter by pillar, date range, tags, severity)
- Build `os-summary.js` (generate context-efficient summary for session injection)
- Integration point: heartbeat can call `os-summary.js` for relevant recent state

---

## 7. Scope

### V1 (This Issue)

✅ `os/` directory structure with 4 pillar definitions  
✅ JSONL schemas and logging scripts  
✅ AGENTS.md slimmed to ≤150 lines  
✅ `os/README.md` as always-injected summary  
✅ YAML config files  
✅ `os-query.js` basic filtering  
✅ `os-summary.js` basic summary generation  
✅ Backfill existing state from MEMORY.md  
✅ IDENTITY.md / MEMORY.md slimmed  

### Deferred (Future)

🔜 OpenClaw injection rules (automatic selective loading based on session type)  
🔜 Vector search over JSONL logs  
🔜 Agent self-managed context (Letta-style core_memory_replace)  
🔜 Config editing via tool/command  
🔜 Dashboard/UI for framework state visualization  
🔜 SQLite migration (only if scale demands it)  
🔜 Heartbeat auto-surfacing of relevant lessons/decisions  

---

## 8. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Migration breaks existing workflows | Medium | High | Phase 1 is purely additive — nothing removed until structure is validated |
| AGENTS.md too slim — agent loses critical context | Medium | Medium | `os/README.md` provides always-on summary; pillar files are one `read` away |
| JSONL files grow unbounded | Low | Low | Monthly rotation; at CruzBot's scale, years before this matters |
| Tony doesn't adopt new structure (edits AGENTS.md out of habit) | Medium | Low | AGENTS.md pointers make it obvious where content lives; brief transition period |
| OpenClaw truncation behavior changes | Low | Medium | Keep always-injected content well under truncation threshold |
| Backfill from MEMORY.md is lossy (unstructured → structured) | High | Low | Best-effort backfill; going forward all events are structured. Historical gaps acceptable. |

---

## 9. Migration Strategy

**Non-breaking, phased rollout:**

1. **Phase 1 (Epic 1):** Create `os/` structure and extract definitions. AGENTS.md unchanged. Zero risk.
2. **Phase 2 (Epic 2):** Scripts write to new JSONL locations. Old scripts still work. Parallel operation.
3. **Phase 3 (Epic 3):** Slim core docs. This is the "cut" — but structure is already validated.
4. **Phase 4 (Epic 4):** Query/summary tools. Pure additive value.

At no point during migration does the system break. Each phase is independently valuable and testable.

---

## Appendix: JSONL Schemas

### Canon Classification
```json
{"ts":"ISO8601","issue":"string","title":"string","workflow":"full|lightweight|off-canon","reason":"string","override":false}
```

### Neo Decision
```json
{"ts":"ISO8601","decision":"string","reasoning":"string","alternatives":["string"],"tradeoffs":"string","category":"execution|technical|process|resource"}
```

### Wolverine Lesson
```json
{"ts":"ISO8601","trigger":"string","lesson":"string","severity":"process|technical|critical","permanent":true,"tags":["string"]}
```

### Evolve Change
```json
{"ts":"ISO8601","file":"string","section":"string","action":"added|modified|removed","summary":"string","diff_ref":"string?"}
```
