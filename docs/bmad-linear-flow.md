# Canon: The Canonical Flow

**aka:** BMAD → Linear → GitHub Flow

**System of Record:** Linear (for active development work)
**Code Repository:** GitHub (practicing what we preach for Prism)
**Planning Artifacts:** `_bmad-output/` (temporary — pushed to Linear, then deleted locally)

**What is Canon?** The right way. The full track. BMAD → AgentOS → Linear → Dev → PR → Merge.

> **Single-Agent Execution (current architecture):** All BMAD phases (Analyst → PM → Architect → SM → Dev → QA) run inline within a single pe-agent session. Phases are file-artifact checkpoints, not agent boundaries. Only the PR Reviewer spawns as a separate agent. See `os/canon/CANON.md` for the full tier/spawn reference.

---

## Canon: The Full Track (Non-Negotiable)

### **Phase 1: Planning (BMAD — inline in pe-agent session)**

All planning phases run within a single pe-agent session. No separate phase agents.

1. **Analyst phase** → Write `stories/analyst-{id}.md` (problem, context, constraints)
2. **PM phase** → Write `stories/story-{id}.md` (PRD, ACs, scope, done criteria)
3. **Architect phase** → Write `stories/arch-{id}.md` (design decisions, tradeoffs, risks)
4. **Implementation Readiness Gate** (verify ACs are clear and scope is bounded)

**Artifacts written to disk at each phase boundary — the file IS the checkpoint.**

---

### **Phase 2: Story Generation (BMAD SM)**

5. **SM** → Story Files (`_bmad-output/stories/`)
   - One file per epic
   - Each story has:
     - Title
     - User story format
     - Acceptance criteria
     - Technical notes
     - Standards applied

**Story files are NOT issues yet. They're specifications.**

---

### **Phase 3: Linear Issue Creation (THIS IS WHERE I WAS BROKEN)**

6. **Push FULL artifact content to Linear** → Create Linear issues with complete story content
   - Script: `scripts/bmad-to-linear.js <epics-file> <ProjectName> --delete-after-push`
   - One Linear issue per story — **full artifact content** in the description (not summaries)
   - Epic-level issues created first (parent issues)
   - Story-level issues linked to epic parents
   - **All metadata preserved:**
     - Epic relationship
     - Full user story + acceptance criteria in description
     - Technical notes and standards applied
   - **After successful push: local artifact files are deleted** — Linear is now the only copy

7. **Linear IS the source of truth** for active work
   - Developers fetch stories from Linear at spawn time (NOT from disk)
   - Fetch command: `node scripts/ops/fetch-linear-story.js CRU-XXX`
   - Issue status tracked in Linear
   - PRs reference Linear issue IDs
   - **NEVER read `_bmad-output/` story files** — they may not exist after push

---

### **Phase 4: Development (GitHub)**

8. **Dev work** (always start with fetching story from Linear)
   ```bash
   node scripts/ops/fetch-linear-story.js CRU-XX   # get your story — Linear is source of truth
   /inject-standards                                 # load coding standards
   ```
   - Create branch from Linear issue: `feature/CRU-XX-description`
   - Commit referencing Linear: `git commit -m "CRU-XX: Implement feature"`
   - PR title: `[CRU-XX] Feature description`
   - PR links to Linear issue

9. **QA** validates against acceptance criteria in Linear issue

10. **Mark Linear issue as Done** when PR merges

---

### **Phase 5: Dogfooding (GitHub as Knowledge Base)**

11. **BMAD artifacts pushed to Prism repo**
    - Product Brief → `docs/planning/product-brief.md`
    - PRD → `docs/planning/prd.md`
    - Epics → `docs/planning/epics.md`
    - Architecture → `docs/architecture/onboarding-architecture.md`
    - Story files → `docs/stories/epic-N/`

12. **Prism uses its own repo** for documentation
    - Practice what we preach
    - Use Prism AI to query our own planning docs
    - Validate Prism's value prop on ourselves

---

## Linear Project Structure

### **Projects:**
- **Prism** (create if doesn't exist)
- **Rook App**
- **Rook Website**
- **Lessnz App**
- **Lessnz Website**
- **Ops** (already exists)

### **Issue Hierarchy:**
```
Epic (Parent Issue)
├── Story 1 (Child Issue)
├── Story 2 (Child Issue)
└── Story 3 (Child Issue)
```

### **Labels:**
- `bmad-workflow` - Generated from BMAD process
- `epic:security-hardening` - Epic-specific tag
- `epic:welcome-setup` - etc.
- `phase:architect` - Which BMAD phase
- `ready-for-dev` - SM complete, standards injected

---

## Scripts Required

### `scripts/bmad-to-linear.js`
**Purpose:** Parse BMAD epics/stories and push FULL artifact content to Linear as ticket descriptions

**Usage:**
```bash
node scripts/bmad-to-linear.js <epics-file> <project-name> [--delete-after-push]
```

**What it does:**
1. Parse epics markdown file
2. For each epic:
   - Create parent Linear issue (Epic-level)
   - Push **full epic artifact content** as the ticket description
3. For each story in epic:
   - Create child Linear issue linked to parent epic
   - Push **full story artifact content** (user story + ACs + technical notes) as ticket description
4. Output summary of created issues
5. With `--delete-after-push`: deletes local artifact files — Linear is now the only copy

### `scripts/ops/fetch-linear-story.js` ← **DEV AGENTS USE THIS**
**Purpose:** Fetch a Linear ticket description at dev spawn time — the Linear-first story read path

**Usage:**
```bash
node scripts/ops/fetch-linear-story.js CRU-123
```

**What it does:**
1. Accepts a Linear issue identifier (e.g. `CRU-123`)
2. Fetches the full ticket description via Linear GraphQL API
3. Prints story content to stdout
4. Dev agents pipe this output into their context at the start of every session

**This replaces reading `_bmad-output/` story files.** Linear is the source of truth.

### `scripts/sync-bmad-to-github.js`
**Purpose:** Push BMAD artifacts to Prism GitHub repo

**Usage:**
```bash
node scripts/sync-bmad-to-github.js <artifact-type>
```

**What it does:**
1. Copy BMAD artifacts to Prism repo structure
2. Git commit with proper message
3. Push to GitHub
4. Create PR if needed

---

## Validation Checklist

**Before creating Linear issues:**
- [ ] SM phase complete (story files exist)
- [ ] Story files have acceptance criteria
- [ ] Standards applied section exists
- [ ] Implementation Readiness gate passed

**Before starting dev work:**
- [ ] Linear issue exists and is assigned
- [ ] Issue has acceptance criteria
- [ ] `/inject-standards` run at session start
- [ ] Branch created from issue

**Before merging PR:**
- [ ] All acceptance criteria met
- [ ] Tests passing (100%)
- [ ] PR references Linear issue
- [ ] QA validation complete

---

## Current State (Prism Onboarding)

**Completed:**
- ✅ Analyst (Product Brief)
- ✅ PM (PRD + Epics)
- ⏳ Architect (next)

**Missing:**
- ❌ Linear Prism project doesn't exist
- ❌ Epics not in Linear
- ❌ Stories not generated yet (need SM phase)
- ❌ No Linear issues created

**Next Actions:**
1. Create Linear Prism project
2. Run Architect phase
3. Run Implementation Readiness gate
4. Run SM phase (generate story files)
5. Run `bmad-to-linear.js` to create issues
6. Start dev work from Linear

---

**This is the canonical flow. No shortcuts. No skipping steps.**
