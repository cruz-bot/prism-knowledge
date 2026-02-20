# Prism Knowledge Base

**Welcome to the Prism product knowledge base** — where we practice what we preach.

This repository contains all product planning, architecture, decisions, and team knowledge for building Prism. It's structured using our own **Product Workspace** template and queried via Prism itself.

## What is Prism?

**Mission:** GitHub as an OS for any business — no coding required.

Prism is the UX layer that makes GitHub accessible to everyone, transforming it into an AI-powered workspace for business knowledge. You shouldn't have to be a developer to live on GitHub.

## Repository Structure

```
prism-knowledge/
├── vision/              # Mission, north star, strategic direction
├── product/             # Product planning artifacts
│   ├── briefs/          # Product briefs (from BMAD Analyst)
│   ├── prds/            # Product requirements (from BMAD PM)
│   ├── epics/           # Epic breakdowns (from BMAD PM)
│   └── architecture/    # Architecture docs (from BMAD Architect)
├── decisions/           # Decision log (ADRs, key choices)
├── team/                # Team roster, roles, contact info
└── processes/           # How we work (BMAD + Agent OS workflow)
```

## How We Use This

1. **Planning**: BMAD agents (Analyst → PM → Architect) produce artifacts that live here
2. **Querying**: We connect this repo to Prism and use it to query our own product knowledge
3. **Building**: Story files reference these docs; Dev agents read them via Prism
4. **Dog-fooding**: This repo validates that Prism works for the exact use case we're selling

## Quick Links

- [Mission & Vision](./vision/mission.md)
- [Current PRD: Frictionless Onboarding](./product/prds/frictionless-onboarding.md)
- [Decision Log](./decisions/decision-log.md)
- [How We Work](./processes/how-we-work.md)
- [Team](./team/team.md)

---

**Built with Prism. For Prism.**
