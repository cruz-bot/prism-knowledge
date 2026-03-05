# Canon: The Canonical Flow

**aka:** BMAD → Linear → GitHub Flow

**System of Record:** Linear (for active development work)  
**Code Repository:** GitHub (practicing what we preach for Prism)  
**Planning Artifacts:** `_bmad-output/` (source documents for Linear issues)

**What is Canon?** The right way. The full track. BMAD → AgentOS → Linear → Dev → PR → Merge.

---

## Canon: The Full Track (Non-Negotiable)

### **Phase 1: Planning (BMAD)**

1. **Analyst** → Product Brief (`_bmad-output/planning-artifacts/`)
2. **PM** → PRD + Epics (`_bmad-output/planning-artifacts/`)
3. **Architect** → Architecture Doc (`_bmad-output/planning-artifacts/`)
4. **Implementation Readiness Gate** (verify architecture is complete)

**Artifacts stay in `_bmad-output/` as source documents.**

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

6. **Parse story files** → Create Linear issues
   - Script: `scripts/bmad-to-linear.js`
   - One Linear issue per story
   - Epic-level issues created first (parent issues)
   - Story-level issues linked to epic parents
   - **All metadata preserved:**
     - Epic relationship
     - Acceptance criteria in description
     - Labels (epic name, BMAD workflow)
     - Links back to story files

7. **Linear becomes source of truth** for active work
   - Developers work from Linear issues
   - Issue status tracked in Linear
   - PRs reference Linear issue IDs

---

### **Phase 4: Development (GitHub)**

8. **Dev work** (always start with `/inject-standards`)
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
**Purpose:** Parse BMAD epics/stories and create Linear issues

**Usage:**
```bash
node scripts/bmad-to-linear.js <epics-file> <project-name>
```

**What it does:**
1. Parse epics markdown file
2. For each epic:
   - Create parent Linear issue (Epic-level)
   - Set project (e.g., "Prism")
   - Add labels
3. For each story in epic:
   - Create child Linear issue
   - Link to parent epic
   - Set acceptance criteria in description
   - Add story file reference
4. Output summary of created issues

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
