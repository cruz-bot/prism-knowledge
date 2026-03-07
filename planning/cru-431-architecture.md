# Architecture: CRU-431 — Live State Aggregator

**Issue:** CRU-431 | **Project:** Ops | **Canon Tier:** Lightweight  
**Architect:** CruzBot | **Date:** 2026-03-05

---

## 1. Overview

### Problem Restated

CruzBot agents rely on markdown files (`MEMORY.md`, daily notes, context-injected snippets) for operational state — issue status, session activity, PR status. These files are stale the moment they're written, causing incorrect reports, redundant spawns, and eroded trust.

### Solution Summary

Create `scripts/ops/get-state.js` — a CLI tool that queries three live, authoritative sources (Linear, Langfuse, GitHub) and outputs a unified operational state summary. Additionally: wire Langfuse session correlation so traces are grouped by session, add Langfuse to `auth-status.js`, and update process docs to mandate live queries over file reads.

### Guiding Principles

1. **Live over stale** — Always prefer API queries over file reads for operational state
2. **Graceful degradation** — Each source fails independently; partial results beat no results
3. **Zero new infrastructure** — Query existing systems, don't create new state stores
4. **CLI-first** — Script outputs to stdout; usable by agents and humans alike
5. **Exit 0 always** — This is a reporting tool, not a gate

---

## 2. Component Design

### `scripts/ops/get-state.js` — Internal Structure

```
get-state.js
├── main()                    # Orchestrator — runs all queries in parallel, formats output
├── queryLinear()             # Linear GraphQL → active issues
├── queryLangfuse()           # Langfuse REST → recent traces/sessions
├── queryGitHub()             # gh CLI → open PRs, recent branch activity
└── formatOutput(results)     # Renders markdown table (default) or JSON (--json flag)
```

#### Data Flow

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Linear  │  │ Langfuse │  │  GitHub  │
│  GraphQL │  │ REST API │  │  gh CLI  │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │              │
     └──────┬──────┘──────┬───────┘
            │             │
     Promise.allSettled() │
            │             │
     ┌──────▼─────────────▼──────┐
     │     formatOutput()        │
     │  (markdown table / JSON)  │
     └───────────┬───────────────┘
                 │
              stdout
```

#### Error Handling Per Source

Each query function follows the same pattern:

```javascript
async function queryLinear() {
  try {
    // ... API call
    return { source: 'linear', ok: true, data: [...] };
  } catch (err) {
    console.error('⚠️  Linear query failed:', err.message);
    return { source: 'linear', ok: false, error: err.message };
  }
}
```

- Each source returns a `{ source, ok, data/error }` envelope
- `main()` uses `Promise.allSettled()` so one failure never blocks others
- Failed sources show a warning line in output, not a crash
- Script always exits 0

#### CLI Interface

```
node scripts/ops/get-state.js           # markdown table output
node scripts/ops/get-state.js --json    # structured JSON output
node scripts/ops/get-state.js --source linear   # single source only
```

---

## 3. API Integration Details

### 3.1 Linear API

**Endpoint:** `https://api.linear.app/graphql` (POST)  
**Auth:** `Authorization: lin_api_REDACTED` (no "Bearer" prefix)  
**Source:** `process.env.LINEAR_API_KEY` (set in `openclaw.json` → `env.LINEAR_API_KEY`)

**Query — Active Issues:**

```graphql
query ActiveIssues($teamId: String!) {
  team(id: $teamId) {
    issues(
      filter: {
        state: { type: { in: ["started", "unstarted"] } }
      }
      orderBy: updatedAt
      first: 25
    ) {
      nodes {
        identifier
        title
        state { name type }
        assignee { name }
        priority
        updatedAt
      }
    }
  }
}
```

**Variables:** `{ "teamId": "f37d329a-00f6-4c74-8dc2-4611306ac76a" }`

**Returned fields:** identifier, title, state name, assignee, priority (1-4), updatedAt

**Filter logic:** Only issues in `started` or `unstarted` state types (covers In Progress, In Dev, In QA, Analyst, PM, Architect, Sprint Ready, Todo). Backlog excluded by default.

### 3.2 Langfuse API

**Base URL:** `https://us.cloud.langfuse.com/api/public`  
**Auth:** Basic auth from `openclaw.json` → `diagnostics.otel.headers.Authorization`  
Value: `Basic cGstbGYtNmZlMDQ3NDYtMmI0ZC00ZjZiLWFiY2ItOWMyMjY1Mzk0NDg4OnNrLWxmLWNhOTRiMjE3LWU1ODMtNGEwOC04ZDNiLWJkZjQ3MGI1YWIzNA==`

**Endpoint — Recent Traces:**

```
GET /api/public/traces?limit=20&orderBy=timestamp.desc
```

**Response shape (relevant fields):**

```json
{
  "data": [
    {
      "id": "trace-uuid",
      "name": "agent:main:main",
      "sessionId": null,  // currently null — Phase 2 fixes this
      "timestamp": "2026-03-05T...",
      "metadata": { ... },
      "totalCost": 0.05
    }
  ]
}
```

**Endpoint — Sessions (post Phase 2):**

```
GET /api/public/sessions?limit=10&orderBy=createdAt.desc
```

**Endpoint — Health check (for auth-status):**

```
GET /api/public/traces?limit=1
```

Returns 200 with data if credentials valid. Used for the auth-status health check.

### 3.3 GitHub (`gh` CLI)

**Path:** `C:\Program Files\GitHub CLI\gh.exe`  
**Auth:** Pre-configured for `cruz-bot` account

**Open PRs:**

```bash
gh pr list --repo VTOR-Tech/knowledgebase-console --state open --json number,title,headRefName,author,updatedAt,reviewDecision
```

**Recent commits on active branches:**

```bash
gh api repos/VTOR-Tech/knowledgebase-console/branches --jq '.[].name' | head -10
```

**Implementation note:** Use `execSync` with a 10s timeout. Parse JSON output directly. If `gh` is unavailable or errors, return graceful failure.

### 3.4 Gateway API

**Base:** `http://127.0.0.1:18789`  
**Auth:** Token `432ef61c5d987ab0` (from `openclaw.json` → `gateway.auth.token`)

**Note from PM spec:** The analyst confirmed there is **no REST API** for live session state on the gateway. The `sessions_list` tool is available to agents at runtime but not as an HTTP endpoint.

**Decision:** `get-state.js` will **not** query the gateway directly. Session information comes from Langfuse traces (post Phase 2) or is omitted with a note. If a gateway API is added in the future, the script can be extended.

---

## 4. Langfuse Session Correlation

### Current State

All traces sent to Langfuse via the `diagnostics-otel` plugin have `sessionId: null`. This means Langfuse's native session grouping is non-functional, and session-level cost/activity queries return nothing useful.

### Investigation Approach

The OTEL traces are emitted by the OpenClaw gateway's built-in `diagnostics-otel` plugin. The question is: **where can we inject the `sessionKey` as the Langfuse `sessionId`?**

Three options, in order of preference:

#### Option A: OTEL Resource/Span Attribute Mapping (Preferred)

Langfuse's OTEL integration maps specific span attributes to its data model. Per Langfuse docs:
- `langfuse.session.id` span attribute → `sessionId`
- `langfuse.user.id` span attribute → `userId`

**Investigation step:** Check if the `diagnostics-otel` plugin or OpenClaw's OTEL SDK already sets span attributes that include the session key. If the gateway exposes a hook/config for custom span attributes, this is a config-only fix.

**Check in openclaw.json:** The `diagnostics.otel` config has `serviceName`, `traces`, `metrics` but no `resourceAttributes` or `spanAttributes` field. This suggests custom attributes aren't currently configurable via JSON.

#### Option B: Custom Extension Plugin

Write a small plugin in `extensions/` that hooks into the OTEL span creation lifecycle and injects `langfuse.session.id` from the active session context.

```javascript
// extensions/langfuse-session-tagger/index.js
module.exports = {
  name: 'langfuse-session-tagger',
  hooks: {
    'otel:span:create': (span, context) => {
      if (context.sessionKey) {
        span.setAttribute('langfuse.session.id', context.sessionKey);
      }
      if (context.agentId) {
        span.setAttribute('langfuse.user.id', context.agentId);
      }
    }
  }
};
```

**Risk:** The `otel:span:create` hook may not exist in OpenClaw's plugin API. This needs investigation of the gateway's plugin hook inventory. If no such hook exists, this option is blocked.

#### Option C: Fork Modification (Last Resort)

Modify the `cruzbot-gateway` fork's OTEL instrumentation code directly to include session key in span attributes.

**Risk:** Maintenance burden on fork, potential merge conflicts with upstream.

### Recommended Approach

1. **Investigate Option A first** — check OpenClaw docs/source for OTEL attribute configuration
2. **If A is blocked, try Option B** — check plugin hook inventory for span lifecycle hooks
3. **If B is blocked, use Option C** — modify the fork's OTEL code (smallest possible change)

### Verification

After implementation, verify by:
1. Trigger a few agent interactions
2. Query `GET /api/public/traces?limit=5` — confirm `sessionId` is non-null
3. Query `GET /api/public/sessions` — confirm sessions appear with grouped traces
4. Check Langfuse UI → Sessions view shows entries

---

## 5. `auth-status.js` Extension

### Existing Structure

The current `scripts/ops/auth-status.js` (CRU-424) has a clean pattern:
- Array of `{ name, fn }` check objects
- Each `fn` returns `{ ok: boolean, detail: string }`
- `Promise.allSettled()` runs all checks in parallel
- Output: emoji + name + detail per check

### Integration Plan

Add a new check function `checkLangfuse()` to the existing checks array:

```javascript
async function checkLangfuse() {
  // Extract Basic auth from openclaw.json → diagnostics.otel.headers.Authorization
  const authHeader = openclawConfig?.diagnostics?.otel?.headers?.Authorization;
  if (!authHeader) {
    return { ok: false, detail: 'no OTEL auth header in openclaw.json' };
  }

  try {
    const res = await httpsRequest({
      hostname: 'us.cloud.langfuse.com',
      path: '/api/public/traces?limit=1',
      method: 'GET',
      headers: {
        'Authorization': authHeader,
      },
    });

    if (res.statusCode === 200) {
      const json = JSON.parse(res.body);
      const count = json?.data?.length ?? 0;
      // Optionally: query today's trace count for richer detail
      return { ok: true, detail: `connected (${count >= 1 ? 'traces flowing' : 'no recent traces'})` };
    }
    return { ok: false, detail: `HTTP ${res.statusCode}` };
  } catch (e) {
    return { ok: false, detail: e.message };
  }
}
```

### Changes to `auth-status.js`

1. Add `checkLangfuse` function (as above)
2. Add to checks array: `{ name: 'Langfuse            ', fn: checkLangfuse }`
3. Total checks increases from 6 → 7

**Output example:**
```
✅ Langfuse             — connected (traces flowing)
```
or
```
❌ Langfuse             — HTTP 401
```

---

## 6. Process & Docs Changes

### 6.1 `AGENTS.md` Changes

**Location:** `C:\Users\cruzb\.openclaw\workspace\AGENTS.md`

**Add new section** after the "Memory" section:

```markdown
## Live State (non-negotiable)

**Never store issue state in markdown.** Issue state lives in Linear. Session state lives in Langfuse. PR state lives in GitHub.

To get current operational state:
```
node scripts/ops/get-state.js
```

This queries Linear, Langfuse, and GitHub live and returns the authoritative view. Use this instead of reading MEMORY.md or daily notes for status information.

**Anti-patterns to avoid:**
- Writing "CRU-XXX is In Progress" to markdown files
- Reading MEMORY.md to determine if an issue is still open
- Hardcoding PR status in notes
```

### 6.2 `HEARTBEAT.md` Changes

**Location:** `C:\Users\cruzb\.openclaw\workspace\HEARTBEAT.md`

**Change in "🚀 Continue Active Sprint" section:**

Add as the first bullet:
```markdown
- **Run `node scripts/ops/get-state.js`** to get live state of all active work (replaces reading MEMORY.md / daily notes for status)
```

**Change in "✅ Monitoring Checklist" section:**

Add new bullet:
```markdown
- **Live state check:** `node scripts/ops/get-state.js` — verify active issues, sessions, and PRs match expectations
```

---

## 7. Implementation Sequence

### Phase 1: Core Script & Health Check

**Dependencies:** None  
**Files:**

| File | Action | Description |
|------|--------|-------------|
| `scripts/ops/get-state.js` | **CREATE** | Main script: `queryLinear()`, `queryLangfuse()`, `queryGitHub()`, `formatOutput()`, `main()` |
| `scripts/ops/auth-status.js` | **EDIT** | Add `checkLangfuse()` function and entry in checks array |

**Acceptance criteria mapping:**
- AC #1: `get-state.js` queries all three sources, outputs clean summary, exits 0 always ✅
- AC #3: `auth-status.js` includes Langfuse health check ✅

**Estimated effort:** Small (single script + minor edit to existing script)

### Phase 2: Langfuse Session Correlation

**Dependencies:** Phase 1 complete (so we can verify session data in `get-state.js` output)  
**Files:**

| File | Action | Description |
|------|--------|-------------|
| (investigation) | **RESEARCH** | Check OpenClaw OTEL config, plugin hooks, and Langfuse attribute mapping |
| `extensions/langfuse-session-tagger/index.js` | **CREATE** (if Option B) | Plugin to inject `langfuse.session.id` span attribute |
| `openclaw.json` | **EDIT** (if Option A) | Add OTEL resource/span attributes config |
| `cruzbot-gateway` fork | **EDIT** (if Option C) | Modify OTEL instrumentation to include session key |

**Acceptance criteria mapping:**
- AC #2: Langfuse traces show non-null `sessionId`, Langfuse Sessions view populated ✅

**Estimated effort:** Medium (investigation + implementation varies by option)  
**Risk:** Medium — the implementation path depends on what OpenClaw's plugin/OTEL API supports

### Phase 3: Process Integration & Documentation

**Dependencies:** Phase 1 complete (script must exist before docs reference it)  
**Files:**

| File | Action | Description |
|------|--------|-------------|
| `AGENTS.md` | **EDIT** | Add "Live State" section after Memory section |
| `HEARTBEAT.md` | **EDIT** | Add `get-state.js` references to Sprint and Monitoring sections |

**Acceptance criteria mapping:**
- AC #4: AGENTS.md and HEARTBEAT.md updated with live state guidance ✅

**Estimated effort:** Trivial (doc edits only)

**Note:** Phase 3 can run in parallel with Phase 2 since both only depend on Phase 1.

---

## Appendix: Config Reference

### Credentials Location Map

| Source | Credential | Location |
|--------|-----------|----------|
| Linear | API Key | `openclaw.json` → `env.LINEAR_API_KEY` |
| Langfuse | Basic Auth | `openclaw.json` → `diagnostics.otel.headers.Authorization` |
| GitHub | OAuth | `gh` CLI pre-configured (Windows Credential Manager) |
| Gateway | Token | `openclaw.json` → `gateway.auth.token` (not used — no API) |

### `openclaw.json` Paths (for `get-state.js` config loading)

```javascript
const OPENCLAW_JSON = path.join(process.env.USERPROFILE || '', '.openclaw', 'openclaw.json');
// or hardcoded: 'C:\\Users\\cruzb\\.openclaw\\openclaw.json'
```

---

*Architecture complete. Ready for Story Manager review.*
