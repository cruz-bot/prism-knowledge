# CRU-185: Cognee Knowledge Graph Plugin Evaluation

**Date:** 2026-02-24
**Evaluator:** CruzBot
**Verdict:** DEFER (revisit when LanceDB limitations surface)

---

## What Is Cognee?

Cognee is a knowledge graph layer for AI agents. The OpenClaw plugin (`@cognee/cognee-openclaw`) syncs Markdown memory files to a Cognee server, builds a knowledge graph, and enables graph-traversal search (not just vector similarity).

## Eval Criteria (from issue AC)

### 1. Can it track relationships between Linear issues, files, decisions?

**Yes, in theory.** Cognee's `GRAPH_COMPLETION` search type traverses relationships between concepts. It would infer connections like "CRU-183 created the canon-enforcer plugin" or "MEMORY.md references LanceDB config." This is genuinely better than vector search for relational queries.

**But:** It requires a running Cognee server (Docker). It doesn't understand Linear natively — it would only know what's in our Markdown files.

### 2. Does it integrate via before_agent_start?

**Yes.** The plugin hooks `before_agent_start` for auto-recall (injects relevant graph context) and `after_agent_run` for auto-index (syncs changed files). Same hook pattern as our canon-enforcer.

### 3. Actively maintained?

**Yes.** Official Cognee integration, documented on docs.cognee.ai, GitHub repo at `topoteretes/cognee-integrations`. Blog post from ~1 week ago. Active development.

### 4. Adds meaningful context beyond LanceDB vector search?

**Potentially yes.** Vector search finds similar text. Graph search finds related concepts even when the text is different. Example: "What decisions affect the gateway?" would traverse relationships, not just keyword-match.

**But:** Our current setup (LanceDB + structured MEMORY.md + os/ JSONL logs) already provides decent context through a combination of vector recall and file-based reading. The incremental value is unclear without testing.

## Infrastructure Cost

- Requires Docker running Cognee server locally
- Additional API calls on every agent run (index + recall)
- State tracking at `~/.openclaw/memory/cognee/`
- Another service to maintain and monitor

## Recommendation: DEFER

**Rationale:**
1. LanceDB + Gemini embeddings just went live today. We haven't hit its limits yet.
2. Cognee adds real value (graph traversal) but also real overhead (Docker, server, maintenance).
3. Our structured file system (os/ JSONL, MEMORY.md, daily logs) already provides relationship context through explicit organization.
4. Better to stress-test LanceDB for 2-4 weeks, identify specific recall failures, then evaluate if Cognee's graph search would have caught them.

**Revisit trigger:** If we find repeated cases where vector search misses relational context that a graph would catch (e.g., "what changed since CRU-170?" returning irrelevant results).

**If adopted later:** Install via `openclaw plugins install @cognee/cognee-openclaw`, run Cognee in Docker, configure alongside (not replacing) LanceDB.
