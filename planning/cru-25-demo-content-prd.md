# CRU-25: Populate prism-demo-kb with Demo Content

## Overview

Populate the `VTOR-Tech/prism-demo-kb` repository with realistic product knowledge base content that showcases all of Prism's features: Ask (AI search), document exploration, knowledge graph with items/stages/releases, citations, diagrams, and workspace views.

The demo KB simulates a fictional SaaS product called **"Nexus"** — a developer collaboration platform. This gives us realistic, relatable content without exposing any real product data.

## Why a Fictional Product?

- Avoids IP concerns for a public repo
- Can be designed to exercise every Prism feature (frontmatter fields, relationships, multiple releases, stages, categories)
- Relatable domain (dev tools) for Prism's target audience

## Content Architecture

### Directory Structure

```
prism-demo-kb/
├── README.md                          # Repo overview, how to use with Prism
├── roadmap/
│   ├── r1-foundation.md               # Release 1: Core platform
│   ├── r2-collaboration.md            # Release 2: Real-time collab
│   └── r3-intelligence.md             # Release 3: AI features
├── narratives/
│   ├── product-vision.md              # North star narrative
│   ├── architecture-overview.md       # System architecture narrative
│   ├── user-journey.md                # End-to-end user journey
│   └── competitive-landscape.md       # Market positioning
├── backlog/
│   ├── workspace-editor.md            # Core capability: editor
│   ├── real-time-sync.md              # Core capability: sync
│   ├── ai-code-review.md              # Differentiating capability
│   ├── smart-notifications.md         # Bet capability
│   ├── plugin-marketplace.md          # Defer capability
│   ├── auth-and-permissions.md        # Core capability: auth
│   └── analytics-dashboard.md         # Differentiating capability
├── discovery/
│   ├── user-research-interviews.md    # Discovery research
│   ├── technical-spike-websockets.md  # Technical spike
│   └── api-design-exploration.md      # API design exploration
├── executive-briefings/
│   ├── q1-2025-status.md              # Quarterly status briefing
│   └── board-update-feb-2025.md       # Board-level update
└── assets/
    └── architecture-diagram.md        # Diagram document (excalidraw-style)
```

### Frontmatter Conventions

All documents use Prism's `BaseFrontmatterSchema`. Every file MUST have at minimum:

```yaml
---
title: "Document Title"
type: canonical | exploratory
category: roadmap | narrative | discovery | backlog | asset | exec_briefing
---
```

**Roadmap documents** additionally include:
- `release`: Release identifier (e.g., "R1", "R2", "R3")
- `related_items`: Array of capability IDs referenced
- `status`: draft | in-progress | complete

**Backlog documents** additionally include:
- `strategic_role`: Core | Differentiating | Bet | Defer
- `priority`: high | medium | low
- `release`: Which release this ships in
- `related_items`: Dependencies/related capabilities
- `journey_phases`: Numeric stage identifiers [1, 2, 3]

**Discovery documents** additionally include:
- `tags`: Relevant tags for searchability
- `related_docs`: Links to related documents
- `status`: draft | in-progress | complete

**Executive briefings** additionally include:
- `persona`: executive
- `briefing_date`: ISO date string
- `created_by`: ai | human | ai+human

### Content Quality Requirements

Each document should be **300–800 words** of realistic, well-structured content with:

1. **Multiple headings** (H1, H2, H3) — exercises TOC extraction
2. **Cross-references** to other documents via `related_docs` and `related_items` — exercises relationship graph
3. **Varied frontmatter** — exercises all entity extractors (capabilities, releases, stages)
4. **Code blocks** where appropriate (architecture docs, API design) — exercises code rendering
5. **Tables and lists** — exercises markdown rendering
6. **Bold/italic emphasis** — realistic formatting

### Entity Coverage

The demo content must produce a rich knowledge graph:

| Entity Type | Target Count | Examples |
|---|---|---|
| Documents | 18–20 | All files listed above |
| Items (Capabilities) | 7+ | workspace-editor, real-time-sync, ai-code-review, etc. |
| Releases | 3 | R1 (Foundation), R2 (Collaboration), R3 (Intelligence) |
| Stages | 5+ | Onboarding, Setup, Daily Use, Collaboration, Analytics |
| Relationships | 30+ | doc↔item, item↔release, doc↔release mappings |

### Feature Showcase Mapping

| Prism Feature | Demo Content That Exercises It |
|---|---|
| **Ask (AI Search)** | Rich, varied content across categories for semantic search |
| **Document Explorer** | 6 directories with 18+ documents, proper frontmatter |
| **Knowledge Graph** | Cross-referenced items, releases, stages via frontmatter |
| **Citations** | Detailed narratives that AI can cite specific sections from |
| **Workspace Views** | Multiple categories visible in workspace sidebar |
| **Diagrams** | Architecture diagram document with diagram_type frontmatter |
| **Executive Briefings** | exec_briefing category documents with persona field |
| **TOC Navigation** | Multi-heading documents for table of contents |
| **Strategic Roles** | Backlog items with Core/Differentiating/Bet/Defer roles |

## Acceptance Criteria

1. **Repository structure**: All directories and files from the structure above exist in `VTOR-Tech/prism-demo-kb`
2. **Valid frontmatter**: Every `.md` file has valid frontmatter that passes `BaseFrontmatterSchema` validation (type + category + title required)
3. **Graph richness**: When ingested by Prism, the graph produces ≥7 items, 3 releases, ≥5 stages, and ≥30 relationships
4. **Content quality**: Each document has 300–800 words of realistic, well-structured markdown
5. **Cross-references**: At least 10 documents reference other documents or items via `related_docs` / `related_items`
6. **Category coverage**: All 6 categories represented (roadmap, narrative, discovery, backlog, asset, exec_briefing)
7. **README**: Root README explains what the repo is and how to connect it to Prism
8. **No real data**: All content is fictional (Nexus product)

## Out of Scope

- Custom domain kit (uses default `product-management-v1`)
- Excalidraw binary files (diagram doc uses markdown description + frontmatter)
- CI/CD or automation
- Non-markdown files (no code files needed — this is a knowledge base, not a codebase)
