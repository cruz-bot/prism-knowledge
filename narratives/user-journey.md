---
title: "Nexus User Journey"
type: canonical
category: narrative
journey_phases:
  - 1
  - 2
  - 3
  - 4
  - 5
---

# Nexus User Journey

## Overview

This document maps the end-to-end user journey through Nexus, from first signup to becoming an engaged power user. Understanding this journey is critical for prioritizing features — every backlog item maps to one or more journey stages.

## Stage 1: Onboarding & Access

**User goal:** "I want to get started quickly without IT tickets."

The journey begins when a developer receives an invitation link or discovers Nexus through their organization's SSO portal. The [authentication and permissions system](../backlog/auth-and-permissions.md) is the first touchpoint.

**Key moments:**

1. Click invite link → SSO authentication (< 5 seconds)
2. Welcome screen with team context pre-loaded
3. First workspace auto-created with a sample project
4. Guided tour highlighting core features (skippable)

**Success metric:** Time from invite click to first code edit < 3 minutes.

**Pain points discovered in [user research](../discovery/user-research-interviews.md):**
- Developers hate multi-step setup wizards
- SSO must work with their existing identity provider — no new passwords
- Sample project should be in a language relevant to their team

## Stage 2: Setup & Personalization

**User goal:** "I want this to feel like my editor."

Once authenticated, developers customize their workspace. The [workspace editor](../backlog/workspace-editor.md) must support enough personalization to feel familiar.

**Key moments:**

1. Import VS Code settings (keybindings, theme, extensions)
2. Connect existing Git repositories
3. Configure terminal preferences (shell, font, colors)
4. Set notification preferences

**Success metric:** 80% of users complete setup in a single session.

**Critical insight:** Developers won't switch editors if the new one doesn't respect their muscle memory. VS Code keybinding import is non-negotiable.

## Stage 3: Daily Use & Productivity

**User goal:** "I want to write great code efficiently."

This is where developers spend 80% of their time. The [workspace editor](../backlog/workspace-editor.md) and [real-time sync](../backlog/real-time-sync.md) are the primary capabilities exercised in this stage.

**Key moments:**

1. Open workspace → resume exactly where they left off
2. Edit code with IntelliSense, autocompletion, and inline docs
3. Commit and push changes via integrated Git
4. Quick file search and navigation (Cmd+P equivalent)
5. Pair programming sessions via shared editing

**Success metric:** Developers spend > 4 hours/day in Nexus (replacing their local IDE).

**Feature requirements:**
- Editor performance must match local VS Code (< 50ms keystroke-to-render)
- Offline mode for plane/train coding sessions
- Split-pane editing for side-by-side file comparison

## Stage 4: Collaboration & Review

**User goal:** "I want feedback on my code without the overhead."

Collaboration is where Nexus differentiates. [Real-time sync](../backlog/real-time-sync.md), [smart notifications](../backlog/smart-notifications.md), and eventually [AI code review](../backlog/ai-code-review.md) converge in this stage.

**Key moments:**

1. Share a workspace link for instant collaboration
2. See teammates' cursors and selections in real time
3. Receive smart notification when a teammate modifies a related file
4. Open pull request → AI review appears within 60 seconds
5. Inline discussion threads on specific code lines

**Success metric:** PR review cycle time < 4 hours (industry average: 24 hours).

## Stage 5: Insights & Growth

**User goal:** "I want to understand how my team is performing."

The final stage is about reflection and improvement. The [analytics dashboard](../backlog/analytics-dashboard.md) and [plugin marketplace](../backlog/plugin-marketplace.md) serve this stage.

**Key moments:**

1. View personal coding activity trends (opt-in)
2. Team lead reviews sprint velocity and code quality metrics
3. Discover and install community plugins for additional functionality
4. Share custom dashboard views with stakeholders

**Success metric:** 60% of team leads check the analytics dashboard weekly.

## Journey-to-Capability Mapping

| Stage | Primary Capabilities | Release |
|-------|---------------------|---------|
| 1. Onboarding | auth-and-permissions | R1 |
| 2. Setup | workspace-editor | R1 |
| 3. Daily Use | workspace-editor, real-time-sync | R1, R2 |
| 4. Collaboration | real-time-sync, smart-notifications, ai-code-review | R2, R3 |
| 5. Insights | analytics-dashboard, plugin-marketplace | R3, Future |

This mapping ensures that every release delivers value across the journey, and no stage is left unaddressed for too long.
