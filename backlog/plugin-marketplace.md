---
title: "Plugin Marketplace"
type: exploratory
category: backlog
strategic_role: Defer
priority: low
journey_phases:
  - 5
---

# Plugin Marketplace

## Summary

The plugin marketplace enables third-party developers to build and distribute extensions for Nexus, creating an ecosystem that extends the platform's capabilities beyond what the core team can build. This is an exploratory initiative — strategically important long-term but deferred until the core platform stabilizes.

## Problem Statement

No single team can build every feature developers want. VS Code's success is largely attributable to its extension marketplace (50,000+ extensions). For Nexus to achieve broad adoption, it needs a similar ecosystem play.

However, building a marketplace too early is a common startup mistake. The platform API must be stable, the developer community must be large enough to attract extension builders, and the security model for third-party code must be robust.

## Conceptual Design

### Extension Types

| Type | Description | Example |
|------|-------------|---------|
| **Language** | Syntax highlighting, IntelliSense, linting | Rust analyzer, Python type checker |
| **Theme** | Visual themes and icon packs | Dracula, Material Theme |
| **Tool** | Development tool integrations | Docker, Kubernetes, database browsers |
| **Workflow** | Workflow automation and productivity | Pomodoro timer, Git workflow helpers |
| **AI** | AI model integrations and custom agents | Custom code review models, doc generators |

### Architecture Vision

Extensions would run in **sandboxed Web Workers** with a capability-based permission model:

- `file:read` — Read files in the workspace
- `file:write` — Modify files (with user approval per-extension)
- `network` — Make external network requests (allowlisted domains)
- `ui:panel` — Render custom UI in sidebar panels
- `ui:decoration` — Add editor decorations (inline hints, highlights)

### Revenue Model (Exploratory)

- Free extensions for community/open-source
- Paid extensions with 70/30 revenue split (developer/Nexus)
- Verified publisher program for enterprise-trusted extensions
- Enterprise customers can create private marketplace for internal extensions

## Why Defer?

1. **Core platform first** — Editor, collaboration, and AI features must be excellent before extending
2. **API stability** — Extension API breaks are ecosystem-killing; API must be stable first
3. **Market size** — Need 10,000+ active developers before extension builders will invest effort
4. **Security complexity** — Third-party code execution requires significant security infrastructure
5. **Resource focus** — Small team should focus on differentiating features (R2, R3)

## Revisit Criteria

Reconsider plugin marketplace when:

- Active developer count exceeds 10,000
- Core platform API has been stable for 6+ months
- At least 5 enterprise customers request it
- Team size allows dedicated marketplace team (2-3 engineers)

## Related Documents

- [Product Vision](../narratives/product-vision.md)
- [User Journey — Stage 5](../narratives/user-journey.md)
