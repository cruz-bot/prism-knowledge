---
title: "User Research: Developer Interviews"
type: exploratory
category: discovery
status: complete
tags:
  - research
  - users
related_docs:
  - narratives/user-journey.md
---

# User Research: Developer Interviews

## Overview

Between October and December 2024, we conducted 12 in-depth interviews with developers across 8 companies to understand their collaboration pain points, tool preferences, and willingness to adopt a browser-based development environment.

## Methodology

- **Format:** 60-minute semi-structured interviews via Zoom
- **Participants:** 12 developers (4 senior, 5 mid-level, 3 junior) from 8 companies (3 enterprise, 3 mid-market, 2 startup)
- **Industries:** FinTech (3), SaaS (4), E-commerce (2), Healthcare (1), Gaming (2)
- **Roles:** 6 full-stack, 3 backend, 2 frontend, 1 DevOps

## Key Findings

### Finding 1: Tool Fragmentation Is Universal

**100% of participants** reported using 6+ tools daily. The average was 8.3 tools.

> "I have VS Code, GitHub, Slack, Jira, Notion, Datadog, my terminal, and Chrome DevTools all open at once. My monitor is a mosaic of context." — P4, Senior Backend Engineer

The most common tool stack: VS Code + GitHub + Slack + Jira + CI/CD dashboard + monitoring.

### Finding 2: Context Switching Is the #1 Pain Point

When asked "What is your biggest daily frustration?", **9 of 12 participants** cited context switching or tool fragmentation.

| Rank | Pain Point | Mentions |
|------|-----------|----------|
| 1 | Context switching between tools | 9 |
| 2 | Waiting for code reviews | 7 |
| 3 | Environment setup for new projects | 6 |
| 4 | Unclear documentation | 5 |
| 5 | Meeting overload | 4 |

### Finding 3: Browser-Based Editors Face Trust Deficit

**7 of 12 participants** expressed skepticism about browser-based editors, primarily around performance and offline support.

> "I've tried Codespaces. It's fine for quick edits, but I'd never do my main work there. The latency is noticeable." — P7, Senior Full-Stack

However, **10 of 12** said they would try a browser-based editor if it:
1. Matched VS Code performance
2. Supported offline mode
3. Imported their VS Code settings

### Finding 4: Real-Time Collaboration Is Desired but Unfamiliar

Only **3 of 12** had used real-time collaborative editing for code (via VS Code Live Share or Replit). But **11 of 12** used Google Docs for collaborative writing and understood the value.

> "If someone made Google Docs for code, I'd use it immediately. Live Share is too clunky." — P2, Mid-Level Full-Stack

### Finding 5: AI Code Review Has High Interest, Low Trust

**10 of 12** expressed interest in AI code review, but **8 of 12** didn't trust it to be accurate enough today.

Key concerns:
- False positives creating noise
- Not understanding team-specific conventions
- Security/privacy of code sent to AI providers

**Trust accelerators identified:**
- Confidence scores on suggestions
- Explanations for every finding
- Ability to dismiss and teach the system
- On-premise / self-hosted options

## Personas Identified

### "Alex" — The Pragmatic Senior

- 8+ years experience
- Deep VS Code investment (50+ extensions)
- Values keyboard shortcuts and efficiency
- Will switch tools only if provably better
- Primary concern: performance and control

### "Jordan" — The Collaborative Mid-Level

- 3-5 years experience
- Enjoys pair programming but finds tooling lacking
- Open to new tools and workflows
- Values real-time feedback and team visibility
- Primary concern: ease of use

### "Sam" — The Learning Junior

- 0-2 years experience
- No strong tool preferences yet
- Wants guidance and examples from senior peers
- Values onboarding speed and documentation
- Primary concern: not looking foolish

## Recommendations

1. **VS Code parity is table stakes** — Performance and keybinding compatibility must be indistinguishable
2. **Lead with collaboration** — The browser-based angle alone isn't compelling enough
3. **Build trust incrementally** — AI features should start conservative (high confidence only) and expand
4. **Offline is non-negotiable** — Enterprise developers travel and work in restricted environments
5. **Import is the killer feature** — Frictionless VS Code settings import eliminates the biggest adoption barrier

## Related Documents

- [User Journey](../narratives/user-journey.md)
- [Competitive Landscape](../narratives/competitive-landscape.md)
- [Product Vision](../narratives/product-vision.md)
