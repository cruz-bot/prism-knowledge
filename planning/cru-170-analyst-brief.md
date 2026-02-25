# CRU-170 Analyst Brief: Agent Framework State Storage — Best Practices

**Date:** 2026-02-23  
**Issue:** CRU-170 — Four Pillars Framework Structured Runtime System  
**Author:** CruzBot (Analyst subagent)

---

## Executive Summary

**Recommendation: Hybrid Markdown + JSONL with a clear separation of concerns.**

- **Framework definitions** (Canon, Neo, Wolverine, Evolve) → **Markdown files** (human-editable, LLM-native, zero parsing overhead)
- **Runtime state/logs** (decisions, learning events, improvement tracking) → **JSONL files** (append-only, machine-queryable, cheap writes)
- **Configuration** → **YAML** (structured, human-readable, widely understood)
- **Context injection** → Selective loader that reads Markdown definitions + queries JSONL logs for relevant recent entries

This mirrors how production frameworks actually work: definitions are static/human-owned, runtime data is structured/append-only, and a context assembly layer decides what goes into each session.

SQLite is unnecessary at CruzBot's scale (single agent, <10K events/month). File-based storage is sufficient and simpler to debug, version-control, and hand-edit.

---

## 1. What Production Agent Frameworks Use

### Storage Format Survey

| Framework | Definitions | Runtime Memory | Long-term Storage | Context Strategy |
|---|---|---|---|---|
| **OpenClaw** | Markdown files (AGENTS.md, SOUL.md) | Markdown daily notes | SQLite + BM25/vector search for RAG | File injection at session start; hybrid search for recall |
| **LangChain/LangGraph** | Code/config | JSON documents in a store | SQLite (SqliteSaver), Postgres, Redis | Selective memory retrieval via store API |
| **AutoGPT** | YAML config | JSON file (LocalCache) | Redis or vector DB (Pinecone, Weaviate) | Embedding-based retrieval |
| **CrewAI** | Python code/config | ChromaDB (short-term) | **SQLite3** (long-term task results) | Automatic injection of relevant memories |
| **Letta (MemGPT)** | Memory blocks (structured text) | Core memory (in-context, editable text blocks) | Archival memory (vector DB) + Recall memory (conversation history) | Agent self-manages: moves data between in-context "RAM" and external "disk" |
| **Mem0** | N/A (middleware layer) | Vector store + graph store | Postgres/Redis/Qdrant | Extracts, consolidates, retrieves salient info; compresses chat history |

### Key Patterns Observed

1. **No framework stores definitions as JSON.** Definitions are always in human-readable formats (Markdown, YAML, code, or structured text blocks).
2. **Runtime/operational data is always structured** — JSON, SQLite, or vector stores.
3. **The split is universal:** static definitions ≠ runtime state ≠ configuration.
4. **Letta's tiered model is the gold standard:** core memory (always in context) → archival memory (searchable, out of context) → recall memory (conversation logs).

---

## 2. Agent Memory Architecture Patterns

### The Three-Tier Pattern (Industry Standard)

Every production framework separates memory into tiers:

| Tier | Purpose | Access Pattern | CruzBot Equivalent |
|---|---|---|---|
| **Core / Working Memory** | Always in context; agent's current state | Read every session | AGENTS.md sections, SOUL.md |
| **Episodic / Recall Memory** | Conversation & event logs | Append-only, query on demand | memory/YYYY-MM-DD.md daily notes |
| **Semantic / Archival Memory** | Long-term knowledge & learnings | RAG retrieval | MEMORY.md, framework-specific logs |

### Append-Only Logs vs. Queryable State

- **Append-only JSONL** excels for: decision logs, learning events, Wolverine improvement tracking, audit trails
- **Mutable state files** (Markdown/YAML) are better for: current framework definitions, active configuration, curated summaries
- **Best practice:** Log everything append-only, periodically summarize into mutable state. This is exactly what Mem0 does (extract → consolidate → retrieve).

### Context Window Budget Management

- **Letta approach:** Agent explicitly manages its own context via tools (core_memory_replace, archival_memory_insert, etc.)
- **LangGraph approach:** Store memories as JSON documents, retrieve relevant ones per session
- **OpenClaw approach:** Inject workspace files at session start, use RAG for recall
- **Mem0 approach:** Compress chat history into "optimized memory representations" — claims significant token reduction

**For CruzBot:** The current approach (inject AGENTS.md + SOUL.md + daily notes) is sound. The improvement is making framework state *structured* so you can inject selectively rather than dumping entire files.

---

## 3. File-Based vs. Database-Backed

### When File-Based Is Sufficient

✅ Single-agent system (CruzBot)  
✅ <10K state events per month  
✅ Human needs direct read/edit access  
✅ Git version control desired  
✅ No concurrent write contention (one agent writes at a time)  

### JSONL Scaling Characteristics

- **Write performance:** ~0.75ms per append (per Gemini CLI benchmarks) — essentially free
- **Read performance:** Linear scan, but for files <100MB this is sub-second
- **Practical limit:** JSONL files work well up to ~100MB-500MB. CruzBot would need years to hit this.
- **Mitigation:** Rotate files by date (e.g., `wolverine-log-2026-02.jsonl`) — natural partitioning

### When SQLite Becomes Necessary

- Multi-agent concurrent writes to the same store
- Need for complex queries (JOINs, aggregations, indexes)
- Data exceeds ~500MB
- Need ACID transactions

**Verdict for CruzBot:** File-based is correct. SQLite adds complexity without proportional benefit at this scale. Revisit if/when CruzBot becomes multi-tenant or log volume explodes.

---

## 4. Context Injection Strategies

### How Frameworks Decide What to Inject

| Strategy | Used By | Pros | Cons |
|---|---|---|---|
| **Full file injection** | OpenClaw (workspace files) | Simple, predictable | Wastes tokens on irrelevant content |
| **RAG retrieval** | Mem0, LangChain, AutoGPT | Token-efficient, scales | Retrieval quality varies; misses context |
| **Agent self-management** | Letta/MemGPT | Most flexible, agent decides | Complex to implement, agent must learn |
| **Selective injection** | Custom systems | Balance of simplicity + efficiency | Requires well-structured source data |

### Recommended Approach for CruzBot

**Structured selective injection:**
1. **Always inject:** Framework definitions (Markdown, ~2-3KB per framework = ~10KB total)
2. **Always inject:** Active configuration/state (YAML, <1KB)
3. **Conditionally inject:** Recent JSONL log entries (last N entries or entries matching current context)
4. **On-demand:** Historical logs via search/grep when agent needs them

This requires framework state to be *structured enough* to select from — which is exactly what CRU-170 enables.

### Token Budget Guideline

- **Framework definitions:** ~10K tokens (always loaded, this is the "core memory")
- **Recent state/logs:** ~2-5K tokens (last 24h of relevant events)
- **Available for conversation:** Remaining context window
- **Total system context:** Keep under 20-30K tokens for efficiency

---

## 5. Human Editability Factor

### The Editability Spectrum

| Format | Human Read | Human Edit | Machine Parse | Machine Query | LLM Native | Git Diff |
|---|---|---|---|---|---|---|
| **Markdown** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **YAML** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **JSONL** | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **JSON** | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| **SQLite** | ⭐ | ⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐ | ⭐ |

### Hybrid Approach (Recommended)

**Tony edits:** Markdown framework definitions + YAML config  
**Agent writes:** JSONL runtime logs (Tony can read but rarely needs to edit)  
**Agent curates:** Periodic summaries from JSONL → Markdown (human-readable digests)

This gives Tony full control over "what the frameworks ARE" while the agent manages "what happened at runtime" in a machine-efficient format.

---

## 6. Comparison Matrix: Storage Approaches

| Criterion | Pure Markdown | Pure JSONL | Markdown + JSONL Hybrid | SQLite | YAML + JSONL |
|---|---|---|---|---|---|
| **Human editability** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ (definitions editable) | ⭐ | ⭐⭐⭐⭐ |
| **Machine queryability** | ⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ (logs queryable) | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **LLM token efficiency** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐ (needs serialization) | ⭐⭐⭐⭐ |
| **Append-only logging** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Git-friendly** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐ (binary) | ⭐⭐⭐⭐ |
| **Implementation complexity** | ⭐⭐⭐⭐⭐ (trivial) | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| **Scale ceiling** | Low | High | High | Very high | High |
| **Matches industry patterns** | Partial | Partial | ✅ Yes | Overkill for single-agent | ✅ Yes |

---

## 7. Final Recommendation

### Proposed Storage Architecture

```
workspace/
├── frameworks/                     # DEFINITIONS (Markdown — Tony edits)
│   ├── canon.md                    # Canon workflow definition
│   ├── neo.md                      # Neo autonomy rules
│   ├── wolverine.md                # Wolverine self-healing protocol
│   └── evolve.md                   # Evolve self-evolution engine
├── config/                         # CONFIGURATION (YAML — structured, editable)
│   ├── frameworks.yaml             # Framework activation state, parameters
│   └── injection-rules.yaml        # What gets injected into which session type
├── runtime/                        # RUNTIME STATE (JSONL — agent writes)
│   ├── decisions.jsonl             # Neo decision log
│   ├── improvements.jsonl          # Wolverine improvement tracking
│   ├── evolution.jsonl             # Evolve change tracking
│   └── workflow-events.jsonl       # Canon workflow state transitions
└── memory/                         # EXISTING (keep as-is)
    ├── YYYY-MM-DD.md               # Daily notes
    └── ...
```

### Format by Purpose

| Data Type | Format | Why |
|---|---|---|
| Framework definitions | Markdown | LLM-native, human-editable, Tony's domain |
| Configuration/parameters | YAML | Structured, human-readable, easily parsed |
| Runtime event logs | JSONL | Append-only, queryable, date-rotatable |
| Daily notes | Markdown | Already works well, keep it |
| Curated memory | Markdown | Already works well (MEMORY.md) |

### Key Tradeoffs Accepted

1. **Two formats to maintain** (Markdown + JSONL) — mitigated by clear separation of who writes what
2. **JSONL is less LLM-native** than Markdown — mitigated by selective injection (only recent/relevant entries)
3. **No complex querying** without loading full JSONL — acceptable at CruzBot's scale; add SQLite layer later if needed
4. **No vector search** — not needed for structured framework state; daily notes already handled by OpenClaw's memory system

### What This Enables

- Tony can edit framework definitions in his editor without breaking anything
- Agent can append structured events without parsing/rewriting Markdown
- Context injection can be surgical: load framework Markdown + last N JSONL entries
- Wolverine improvements become queryable: "show me all improvements from last week"
- Evolve changes get a proper audit trail
- Neo decisions become reviewable and searchable
- Git diffs remain meaningful for definitions; JSONL rotates cleanly

---

## Sources

- LangChain/LangGraph memory: JSON documents in stores, SQLite for persistence ([docs.langchain.com](https://docs.langchain.com/oss/python/langchain/long-term-memory))
- AutoGPT: JSON file (LocalCache) default, Redis for Docker ([autogptdocs.com](https://autogptdocs.com/configuration/memory))
- CrewAI: SQLite3 for long-term memory ([docs.crewai.com](https://docs.crewai.com/core-concepts/Memory/))
- Letta/MemGPT: Tiered core/archival/recall memory model ([letta.com/blog/agent-memory](https://www.letta.com/blog/agent-memory))
- Mem0: Memory orchestration layer with extraction/consolidation ([arxiv.org/abs/2504.19413](https://arxiv.org/abs/2504.19413))
- OpenClaw: Markdown files + SQLite RAG index ([docs.openclaw.ai](https://docs.openclaw.ai/concepts/memory), [pingcap.com](https://www.pingcap.com/blog/local-first-rag-using-sqlite-ai-agent-memory-openclaw/))
- Gemini CLI JSONL benchmarks: ~0.75ms append, >9000x faster than read-modify-rewrite ([github.com/google-gemini/gemini-cli#15292](https://github.com/google-gemini/gemini-cli/issues/15292))
- Context engineering best practices ([redis.io/blog](https://redis.io/blog/context-engineering-best-practices-for-an-emerging-discipline/))
