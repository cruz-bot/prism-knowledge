---
title: "Release 1: Foundation — Core Platform"
type: canonical
category: roadmap
release: R1
status: complete
related_items:
  - workspace-editor
  - auth-and-permissions
---

# Release 1: Foundation — Core Platform

## Overview

Release 1 establishes the foundational capabilities of the Nexus developer collaboration platform. This release focuses on the two pillars every user interaction depends on: a powerful workspace editor and a robust authentication and permissions system.

R1 shipped on schedule in Q4 2024, delivering the minimum viable experience that enables individual developers to set up accounts, create workspaces, and begin editing code in the browser-based editor.

## Goals

1. **Secure onboarding** — Users can sign up via SSO (Google, GitHub, SAML) and be assigned to teams with appropriate role-based access.
2. **Workspace creation** — Every user can create and manage personal and team workspaces.
3. **Core editing experience** — A Monaco-based code editor with syntax highlighting for 40+ languages, IntelliSense, and Git integration.
4. **Reliability baseline** — 99.9% uptime SLA, < 200ms P95 latency for editor operations.

## Key Capabilities

### Workspace Editor

The workspace editor is the heart of Nexus. Built on Monaco (the engine behind VS Code), it provides a familiar editing experience in the browser. Key features in R1:

- Multi-tab editing with split panes
- Syntax highlighting for JavaScript, TypeScript, Python, Go, Rust, and 35 additional languages
- Integrated terminal (WebSocket-backed PTY)
- File tree explorer with drag-and-drop
- Basic Git operations: commit, push, pull, branch switching

### Authentication & Permissions

Nexus uses a layered auth model designed for enterprise teams:

| Layer | Description |
|-------|-------------|
| **Identity** | OAuth 2.0 / SAML SSO via Auth0 integration |
| **Roles** | Owner, Admin, Member, Viewer |
| **Scopes** | Workspace-level, Project-level, File-level |
| **Audit** | Full audit trail for all permission changes |

RBAC policies are defined as JSON documents and version-controlled alongside workspace settings.

## Milestones

- **2024-09-01** — Editor MVP internal alpha
- **2024-10-15** — Auth integration complete, SSO tested with 3 identity providers
- **2024-11-20** — Beta launch to 50 design partners
- **2024-12-10** — GA release

## Outcome

R1 successfully onboarded 50 design partners with a 78% weekly active rate. The editor received strong positive feedback, with the integrated terminal cited as the standout feature. Auth required two post-launch patches to address SAML edge cases with Okta configurations.

## Lessons Learned

- Monaco integration was smoother than expected; investing in the WebSocket terminal layer early paid dividends.
- SAML testing should include at least 5 identity providers, not just 3. Okta-specific quirks were caught late.
- File-level permissions were deprioritized to R2 but should be reconsidered for R1.1 based on enterprise feedback.

## Related Documents

- [Workspace Editor backlog item](../backlog/workspace-editor.md)
- [Auth & Permissions backlog item](../backlog/auth-and-permissions.md)
- [Architecture Overview](../narratives/architecture-overview.md)
