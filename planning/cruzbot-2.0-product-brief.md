# Product Brief: CruzBot 2.0 Evolution

**Author:** BMAD Analyst
**Date:** February 20, 2026
**Version:** 1.0
**Status:** Planning Complete

---

## 1. Executive Summary

CruzBot 2.0 Evolution is a strategic initiative to transform the existing OpenClaw-based agent into a highly autonomous, efficient, and intelligent development partner for Tony. This project will not fork OpenClaw but will instead build a sophisticated **hybrid orchestration layer** on top of the proven gateway.

The core of this evolution involves three key enhancements:
1.  **Automated Workflow Orchestration:** Replacing brittle, manual scripts with a robust engine to manage the BMAD (Build, Measure, Analyze, Decide) development lifecycle, seamlessly integrating with Linear and GitHub.
2.  **Advanced Hybrid Memory:** Upgrading the agent's memory from simple files to a powerful combination of a vector store (for semantic search) and a knowledge graph (for relational reasoning), allowing the agent to understand context and history.
3.  **Custom User Interface:** Introducing a dedicated web dashboard and VS Code extension to provide a workflow-optimized view of agent activities, progress, and insights, including a remote dashboard as requested by Tony.

The project is planned for a **phased 6-month delivery**, requiring an estimated **~440 hours** of effort and a modest **$50-100/month** in API costs. The expected impact is a **5-10x increase in development velocity**, a move towards **100% quality** through automation, and achieving **80% agent autonomy** in routine development tasks.

---

## 2. Problem Statement

The current CruzBot, while functional, operates under significant constraints that create a bottleneck in Tony's multi-product development workflow. Feature delivery is measured in days, not hours, due to a series of manual processes and architectural limitations.

**Key Problems:**
*   **Brittle, Manual Integrations:** The current workflow relies on manually executed scripts (`bmad-to-linear.js`) to sync planning artifacts with project management tools (Linear). These scripts are fragile, lack error handling, and require constant human intervention.
*   **Lack of Event-Driven Automation:** The system is not reactive. A status change in Linear does not trigger a development agent, and a GitHub merge does not update the corresponding task. This creates delays and requires manual coordination.
*   **Shallow, Inefficient Memory:** The agent's memory consists of flat Markdown files (`MEMORY.md`). This system lacks semantic relationships, preventing the agent from answering critical "why" questions about past decisions. It forces the agent to re-read large context files every session, which is inefficient and costly.
*   **Generic, Inadequate UI:** The default OpenClaw control panel is not designed for the BMAD workflow. It offers no specialized views for sprint progress, agent activity, or cross-tool (Linear/GitHub) status, forcing context-switching between multiple browser tabs and terminals.
*   **No Specialized Intelligence:** Agent and model routing is static and manual. All tasks are handled by a generalist agent, with no mechanism to route complex architectural questions to a more powerful model (like Opus) and simple fixes to a faster, cheaper one (like Flash).

Addressing these issues is critical to scaling development capacity and enabling the agent to function as a true, autonomous partner.

---

## 3. Vision & Goals

### Vision

To evolve CruzBot into a proactive, autonomous peer partner that manages the end-to-end development lifecycle, enabling Tony to transition from a hands-on-keyboard "doer" to a strategic "director." CruzBot 2.0 will not just execute tasks; it will anticipate needs, manage workflows, and learn from outcomes, fundamentally accelerating the pace of innovation across all five products.

### Goals

*   **Goal 1: Automate the Full BMAD Lifecycle.**
    *   **Description:** Eliminate all manual scripts and processes involved in moving a feature from conception (Analyst) to completion (QA).
    *   **Key Result:** Achieve a "zero-toil" integration where BMAD artifacts automatically generate Linear issues and progress is bidirectionally synced.

*   **Goal 2: Achieve a 5-10x Increase in Development Velocity.**
    *   **Description:** Drastically reduce the time it takes to ship a standard feature.
    *   **Key Result:** Reduce feature delivery time from 2-5 days down to 2-4 hours by the end of the project.

*   **Goal 3: Enhance Agent Intelligence for Deeper Contextual Understanding.**
    *   **Description:** Empower the agent with a long-term, relational memory.
    *   **Key Result:** The agent can successfully answer complex "why was X decision made?" questions by referencing a knowledge graph of past decisions, projects, and conversations.

*   **Goal 4: Create a "Single Pane of Glass" for Workflow Management.**
    *   **Description:** Provide a centralized, workflow-optimized UI for monitoring and interacting with the agent and its tasks.
    *   **Key Result:** Launch a web dashboard and VS Code extension that consolidates views from Linear, GitHub, and agent logs, reducing context-switching.

*   **Goal 5: Increase Agent Autonomy to 80%.**
    *   **Description:** Shift the balance of work from human to agent.
    *   **Key Result:** Reduce the number of required human interventions per week from 10-15 down to 1-2.

---

## 4. User Personas

### Primary Persona: Tony

*   **Role:** Sole developer, product owner, and architect for five distinct software products.
*   **Goals:**
    *   Maximize feature output across all products.
    *   Reduce manual, repetitive development work (toil).
    *   Spend more time on high-level strategy and creative problem-solving.
    *   Ensure high-quality, consistent code and architecture across his portfolio.
*   **Frustrations:**
    *   "I'm a bottleneck. Everything requires my input."
    *   "I waste too much time syncing information between my notes, Linear, and GitHub."
    *   "My agent is helpful, but I have to guide it through every single step."
    *   "I forget why we made certain architectural decisions months ago."
*   **Needs for CruzBot 2.0:**
    *   An agent that can run the development process on its own.
    *   A system that "just works" in the background, connecting the tools he already uses.
    *   A way to quickly see "what are all the agents working on right now?"
    *   An agent that remembers past decisions and applies them consistently.

---

## 5. Core Features

### 1. Orchestration Layer (CruzBot Core)

A custom-built workflow engine that sits on top of OpenClaw.
*   **BMAD Workflow Executor:** Automates the execution of the Analyst, PM, Architect, SM, Dev, and QA phases. It will support parallel execution of phases (e.g., Analyst and PM) to save time.
*   **Integration Hub:** Provides native, event-driven, bidirectional integrations with Linear and GitHub. Replaces all manual synchronization scripts.
*   **Task Queue & Agent Router:** Manages a distributed task queue (using Redis) and intelligently routes tasks to the most appropriate agent and model based on complexity and type.

### 2. Enhanced Memory Layer

A hybrid memory system to provide the agent with deep, queryable context.
*   **Vector Store (Qdrant):** For semantic search and recalling similar past conversations, documents, or code. Answers "what did we say about X?"
*   **Knowledge Graph (Neo4j):** To store and query entities (decisions, people, products, technologies) and their relationships. Answers "why did we choose X for product Y?"
*   **Auto-Indexing Pipeline:** Automatically ingests and processes data from BMAD artifacts, Telegram conversations, Linear issues, and GitHub activity into the memory system.

### 3. Custom Interface Layer

A user-facing layer designed specifically for the BMAD workflow.
*   **Web Dashboard:** A Linear-inspired web application showing a real-time sprint board, BMAD epic progress, and a monitor for all active agents. This dashboard will be remotely accessible.
*   **VS Code Extension:** An inline agent that brings CruzBot directly into the editor, providing context-aware suggestions, code actions ("Ask CruzBot"), and automated test generation.

---

## 6. Success Metrics

Success will be measured against clear velocity, quality, and autonomy targets.

### Velocity Metrics
| Metric | Baseline (Today) | Target (Phase 3) | Target (Phase 6) |
|---|---|---|---|
| **Time to ship feature** | 2-5 days | 4-8 hours | 2-4 hours |
| **BMAD workflow time** | 60-90 min | 30-45 min | 20-30 min |
| **Manual steps per feature**| 15-20 | 5-8 | 0-2 |
| **Linear sync time** | 10-15 min (manual)| < 30s (auto) | < 10s (auto) |

### Quality Metrics
| Metric | Baseline | Target (Phase 3) | Target (Phase 6) |
|---|---|---|---|
| **Test coverage** | 70-80% | 95%+ | 100% |
| **Regressions per sprint** | 2-3 | 0-1 | 0 |
| **Code review rounds** | 2-3 | 1-2 | 1 |
| **Standards compliance** | 80% | 95% | 100% |

### Autonomy Metrics
| Metric | Baseline | Target (Phase 3) | Target (Phase 6) |
|---|---|---|---|
| **Agent-driven tasks** | 20% | 60% | 80% |
| **Human interventions** | 10-15/week | 3-5/week | 1-2/week |
| **Proactive suggestions**| 0-1/week | 3-5/week | 10+/week |

---

## 7. Risks & Mitigation

| Risk | Level | Mitigation Strategy |
|---|---|---|
| **Integration Complexity** | Medium | The orchestration layer adds new potential failure points. We will mitigate this by starting with a simple, sequential workflow in Phase 1 and implementing parallel execution later. Comprehensive error handling and latency monitoring will be built in from day one. |
| **Scope Creep** | Medium | There's a risk of adding too many features beyond the core MVP. This will be managed through strict phase gates, a dedicated "Future Phase" backlog, and weekly progress reviews to ensure focus remains on the primary goals. |
| **Knowledge Graph Learning Curve** | Low-Medium | Neo4j and graph database concepts are new. The risk of delay will be mitigated by leveraging existing open-source implementations (Mem0, Cognee) as references, starting with a simple schema, and timeboxing learning activities. The graph component can be deferred if necessary. |
| **OpenClaw Upstream Changes**| Low | Breaking changes in the core OpenClaw dependency could disrupt our custom layer. We will pin the OpenClaw version in `package.json` and test all upgrades in a development environment before deploying to production. |
| **Data Loss** | Low | The PostgreSQL and Neo4j databases are single points of failure. This will be mitigated with daily automated backups, tested restore procedures, and version control for all schema changes. |

---

## 8. Out of Scope

*   **Forking OpenClaw:** This project will explicitly **not** create a fork of OpenClaw. The goal is to build on top of the existing, stable gateway to leverage its ecosystem and avoid maintenance overhead.
*   **Building a New Messaging Layer:** We will use OpenClaw's existing channel integrations (Telegram, Discord, etc.) and will not build any new messaging infrastructure.
*   **Replacing Core OpenClaw Tools:** The project will utilize the existing battle-tested tools like browser control, file operations, and `exec`, enhancing their coordination rather than replacing them.
*   **Cloud Deployment (Initial Phases):** All services will be self-hosted on existing hardware (Docker on Windows/Linux). A migration to the cloud is a future consideration, not part of this initial 6-month project.

---

## 9. Go/No-Go Criteria

This project will follow a phased approach with clear go/no-go decision points at the end of each major phase.

*   **End of Phase 1 (Foundation):**
    *   **Go:** Proceed to Phase 2 if the orchestration layer successfully automates the BMAD-to-Linear workflow without manual intervention and is stable.
    *   **No-Go/Pivot:** If the custom layer proves too complex or unreliable, simplify the scope to focus only on vector memory and basic UI before re-evaluating the orchestration strategy.

*   **End of Phase 2 (Intelligence):**
    *   **Go:** Proceed to Phase 3 if the hybrid memory system demonstrates clear value in agent recall and contextual understanding.
    *   **No-Go/Pivot:** If the memory system is not stable or valuable, defer the UI and focus on hardening the backend before proceeding.

*   **End of Phase 3 (Interface):**
    *   **Go:** Ship the complete system to "production" (daily use) if the success metrics for velocity and quality show at least a 50% improvement over the baseline.
    *   **No-Go/Pivot:** If the metrics are not met, the project will enter an "extended beta" phase to focus on reliability and performance improvements before a full production rollout.
