# Prism Demo Knowledge Base — Nexus

This repository is a **demo knowledge base** for [Prism](https://github.com/VTOR-Tech/knowledgebase-console), showcasing how product teams can organize and explore their product knowledge using Prism's AI-powered features.

## What's Inside

The content simulates a fictional SaaS product called **Nexus** — a developer collaboration platform. All content is fictional and designed to exercise every Prism feature.

### Directory Structure

```
├── roadmap/                    # Release plans (R1, R2, R3)
├── narratives/                 # Vision, architecture, user journey, competitive analysis
├── backlog/                    # Capability specs with strategic roles and priorities
├── discovery/                  # Research, technical spikes, explorations
├── executive-briefings/        # Quarterly reports and board updates
└── assets/                     # Architecture diagrams and visual assets
```

### Content Overview

| Directory | Files | Description |
|-----------|-------|-------------|
| `roadmap/` | 3 | Three releases: Foundation (complete), Collaboration (in-progress), Intelligence (draft) |
| `narratives/` | 4 | Product vision, architecture overview, user journey (5 stages), competitive landscape |
| `backlog/` | 7 | Capabilities with strategic roles: Core, Differentiating, Bet, Defer |
| `discovery/` | 3 | User research interviews, WebSocket technical spike, API design exploration |
| `executive-briefings/` | 2 | Q1 2025 status report, February board update |
| `assets/` | 1 | System architecture diagram description |

### Prism Features Exercised

- **Knowledge Graph** — 7 items, 3 releases, 5 journey stages, 30+ relationships via frontmatter
- **Ask (AI Search)** — Rich, varied content across 6 categories for semantic search
- **Document Explorer** — 20 documents organized in 6 directories
- **Strategic Views** — Backlog items tagged with Core/Differentiating/Bet/Defer roles
- **Journey Stages** — User journey mapped across 5 stages with capability-to-stage mapping
- **Citations** — Detailed narratives with specific sections for AI to cite
- **Executive Briefings** — Persona-tagged briefing documents

## Connecting to Prism

1. Open Prism (`knowledgebase-console`)
2. Go to **Settings → Sources**
3. Add this repository as a Git source:
   ```
   https://github.com/cruz-bot/prism-knowledge.git
   ```
4. Prism will ingest all markdown files, extract frontmatter, and build the knowledge graph
5. Explore via Ask, Document Explorer, or Knowledge Graph views

## Frontmatter Schema

All documents use Prism's `BaseFrontmatterSchema`. Required fields:

```yaml
---
title: "Document Title"
type: canonical | exploratory
category: roadmap | narrative | discovery | backlog | asset | exec_briefing
---
```

See individual documents for category-specific fields like `release`, `strategic_role`, `priority`, `journey_phases`, `persona`, and `briefing_date`.

## License

This demo content is part of the Prism project. See the main repository for license details.
