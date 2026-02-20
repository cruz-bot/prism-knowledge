---
title: How We Work
date: 2026-02-20
---

# How We Work

Prism development follows the **BMAD + Agent OS** integrated workflow.

## The Stack

- **BMAD v6** — Lifecycle engine: Analyst → PM → Architect → SM → Dev → QA
- **Agent OS v3** — Standards engine: discover → index → inject → spec
- **Linear** — Issue tracking (issues created FROM story files, not before)
- **GitHub** — Code + knowledge base (this repo)

## The Two Integration Seams

### Seam 1: Standards into Planning (Architect Phase)

Before the Architect runs:
1. Check if `agent-os/standards/` exists
2. If empty, run `/discover-standards` on the codebase
3. Architect references relevant standards in the architecture doc

**Why:** Ensures architectural decisions follow existing patterns instead of inventing new ones.

### Seam 2: Standards into Every Dev Session

When BMAD Dev implements any story:
1. **FIRST command is ALWAYS `/inject-standards`** — loads coding patterns into context
2. Read the story file (authoritative source)
3. Implement following story tasks AND injected standards
4. Write tests, mark done only when tests pass 100%

**Why:** Prevents vibe coding, ensures consistency, maintains quality.

## Non-Negotiable Rules

1. **Never write code without a story file OR at minimum `/inject-standards`**
2. **Standards FIRST, code SECOND** — always
3. **Stories are authoritative** — read full story, execute tasks in order
4. **Tests before done** — 100% pass rate required
5. **Fresh sessions** — planning and dev agents never share sessions

## Task Size → Workflow

| Signal | Workflow |
|---|---|
| Bug fix / < 3 files | `/inject-standards` → code directly |
| Small feature | `/shape-spec` + BMAD Quick Flow |
| Medium/large feature | Full BMAD track |

## Model Routing

- **Planning (Analyst, PM, Architect):** `gemini` (large context + free)
- **SM (story creation):** `gemini` (document synthesis + free)
- **Dev, Code Review, QA:** `antigravity` (Claude quality + free)
- **Main session (Tony ↔ CruzBot):** `sonnet` (quality for direct interaction)

**Rule:** Sub-agents ALWAYS get an explicit `model:` parameter. Never spawn without specifying.

## Quality Gates

### Before Marking Any Story Done:
- [ ] All tasks in story marked `[x]`
- [ ] Tests written for every task
- [ ] Full test suite passes 100%
- [ ] Code review run
- [ ] Story file updated with Dev Agent Record

### Before Shipping Any Epic:
- [ ] All stories in epic are done
- [ ] QA workflow run on epic
- [ ] Retrospective complete

### Before Any Public Launch:
- [ ] All P0/SEV-1 security issues resolved
- [ ] Implementation Readiness gate passed
- [ ] Production deployment checklist complete

## Linear Integration

Issues are created **ONLY after SM produces story files**:
1. Read story file
2. Create one Linear issue per story
3. Issue title = story title
4. Issue description = story acceptance criteria + technical notes
5. Link to story file in description

**Never create issues from assumptions.** Story files are truth.

## Reference Docs

- **Process checklist:** `workspace/skills/dev-discipline/SKILL.md`
- **Workflow DNA:** `workspace/docs/dev-workflow-dna.md`
- **Integration guide:** `workspace/docs/agentOS-bmad-integration.md`

---

**This is how we ship quality code without chaos.**
