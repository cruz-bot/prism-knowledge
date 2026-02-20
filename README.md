# Prism Knowledge Base

**Repository:** `cruz-bot/prism-knowledge`  
**Purpose:** Canonical location for Prism planning documentation and BMAD workflow artifacts  
**Dogfooding:** We practice what we preach — using GitHub as a knowledge base, just like our users will  

---

## What's Here

### `/planning/` - BMAD Planning Artifacts
All artifacts from the BMAD (Build → Measure → Architect → Deploy) workflow:
- **Product Brief** - Vision, problem statement, personas, user journeys, North Star metrics
- **PRD** - Requirements, NFRs (non-functional requirements), success criteria
- **Epics** - High-level feature groupings with acceptance criteria
- **Architecture** - System design, technical decisions, ADRs

### `/standards/` - AgentOS Standards
Discovered standards from the AgentOS standards engine:
- **Prism Standards** - Code patterns, conventions, quality gates specific to Prism

### `/docs/` - Workflow Documentation
Process and methodology docs:
- **BMAD→Linear→GitHub Flow** - The canonical workflow for how planning → development → dogfooding works

---

## Philosophy

**GitHub as Operating System:** We're building Prism to make GitHub accessible to non-technical users. We dogfood our own product by storing all our planning docs here.

**Linear as System of Record:** Active development work tracked in Linear (issues, sprints, progress). GitHub = code repo + knowledge base.

**BMAD Workflow:** Analyst → PM → Architect → Implementation Readiness → SM → Dev → QA. All planning artifacts flow here, then into Linear, then into code.

---

## Auto-Sync

This repository is automatically synced from the CruzBot workspace:
- **Script:** `D:\backups\sync-prism-knowledge.ps1`
- **Schedule:** Daily at 3:00 AM CST (as part of backup automation)
- **Source:** `C:\Users\cruzb\.openclaw\workspace\_bmad-output\planning-artifacts\`

Manual sync: Run `D:\backups\sync-prism-knowledge.ps1` anytime.

---

## Contributing

This repo is auto-maintained by CruzBot 💠 as part of the development workflow. Changes should originate from BMAD workflow runs in the main workspace.

**Questions?** See `docs/bmad-linear-flow.md` for the full development process.

---

Built with discipline. Shipped with velocity. 🚀
