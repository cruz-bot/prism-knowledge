# Product Requirements Document: CruzBot 2.0 Evolution

**Author:** BMAD PM Agent
**Date:** February 20, 2026
**Version:** 1.0

## 1. Introduction & Background

CruzBot 2.0 Evolution is a strategic project to transform the existing OpenClaw agent from a command-driven tool into a proactive, autonomous development partner for its user, Tony. This initiative does not involve forking OpenClaw but instead focuses on building a sophisticated **hybrid orchestration layer** that sits on top of the core gateway. This layer will manage the end-to-end BMAD (Build, Measure, Analyze, Decide) development lifecycle, enhance the agent's contextual understanding through advanced memory, and provide a workflow-optimized user interface. The ultimate goal is to dramatically increase development velocity, quality, and agent autonomy across Tony's five products.

## 2. Problem Statement

The current CruzBot 1.0, while effective for discrete tasks, acts as a bottleneck in a multi-product development environment. Its reliance on manual processes, coupled with architectural limitations, measures feature delivery in days, not hours. The key problems to be solved are:

-   **Brittle, Manual Integrations:** The current workflow depends on fragile, manually-triggered scripts (e.g., `bmad-to-linear.js`) to synchronize planning artifacts with project management tools. These scripts lack robust error handling and require constant human oversight.
-   **Lack of Event-Driven Automation:** The system is not reactive. A status change in a Linear ticket does not trigger a development agent, and a pull request merge on GitHub does not update the corresponding task. This forces manual coordination and introduces delays.
-   **Shallow, Inefficient Memory:** The agent's memory, based on flat Markdown files, lacks semantic relationships and historical context. This prevents the agent from understanding the "why" behind past decisions and forces costly re-reading of large files every session.
-   **Generic, Inadequate UI:** The standard OpenClaw interface is not tailored to the BMAD workflow. It provides no consolidated view of sprint progress, agent activity, or cross-tool status, forcing the user to context-switch between Linear, GitHub, and terminal windows.
-   **Static Agent Intelligence:** Agent and model routing is manual. There is no automated mechanism to assign complex architectural tasks to a powerful model (e.g., Opus) and simple bug fixes to a faster, more cost-effective model (e.g., Flash).

## 3. Target Audience & User Personas

### Primary Persona: Tony

-   **Role:** Sole developer, product owner, and architect for five distinct software products.
-   **Goals:**
    -   Maximize feature output across his entire product portfolio.
    -   Eliminate manual, repetitive development "toil."
    -   Focus his time on high-level strategy, creative problem-solving, and product direction.
    -   Maintain a high bar for code quality, architectural consistency, and testing standards.
-   **Frustrations:**
    -   Being the bottleneck for all development and decision-making.
    -   Wasting significant time manually syncing information between his notes, Linear, and GitHub.
    -   Having to micromanage the agent through every step of a complex workflow.
    -   Forgetting the context or rationale for architectural decisions made months prior.

## 4. Goals and Success Metrics

### Product Goals

-   **Goal 1: Automate the Full BMAD Lifecycle:** Eliminate all manual scripts and human touchpoints in moving a feature from planning to deployment.
-   **Goal 2: Achieve a 5-10x Increase in Development Velocity:** Reduce the cycle time for a standard feature from days to hours.
-   **Goal 3: Foster Deep Contextual Understanding:** Equip the agent with a long-term, relational memory that allows it to learn from and apply past decisions.
-   **Goal 4: Create a "Single Pane of Glass" for Workflow Management:** Provide a centralized, workflow-optimized UI to monitor and direct agent activity.
-   **Goal 5: Increase Agent Autonomy to 80%:** Shift the balance of work from human-directed to agent-driven.

### Key Success Metrics

| Metric | Baseline (Today) | Target (End of Project) |
| :--- | :--- | :--- |
| **Time to ship a standard feature** | 2-5 days | **2-4 hours** |
| **Manual steps per feature** | 15-20 | **0-2** |
| **Human interventions required per week**| 10-15 | **1-2** |
| **Agent-driven tasks** | 20% | **80%** |
| **Test coverage** | 70-80% | **100%** |
| **Ability to answer "why" questions**| None | **Successful in >90% of tests** |
| **Linear <-> GitHub sync time** | 10-15 min (manual)| **< 10s (event-driven)** |

## 5. Functional Requirements

### FR1: Orchestration Layer (CruzBot Core)

-   **FR1.1: BMAD Workflow Engine:** The system shall provide a state machine to execute the BMAD lifecycle phases (Analyst, PM, Architect, SM, Dev, QA) for a given feature.
-   **FR1.2: Task Queue:** A Redis-based task queue shall be implemented to manage tasks for different agents. This must support prioritization and routing.
-   **FR1.3: Intelligent Agent/Model Routing:** The orchestrator shall route tasks to the appropriate agent (e.g., "Developer Agent," "QA Agent") and select the optimal model (e.g., Opus for architecture, Flash for simple file edits) based on pre-defined rules.
-   **FR1.4: Parallel Execution:** The workflow engine must support parallel execution of non-dependent phases (e.g., running Analyst and PM concurrently) to optimize for speed.

### FR2: Integration Hub

-   **FR2.1: Bidirectional Linear Sync:**
    -   The system shall automatically create Linear issues from approved BMAD artifacts.
    -   The system shall listen for webhooks from Linear. A change in issue status (e.g., "In Progress") shall trigger the corresponding agent workflow (e.g., spawn a Dev Agent).
    -   Agent progress and comments shall be automatically posted back to the relevant Linear issue.
-   **FR2.2: Bidirectional GitHub Sync:**
    -   Dev Agents shall automatically create branches, commits, and pull requests in the correct GitHub repository.
    -   PRs must be linked to the corresponding Linear issue.
    -   The system shall listen for webhooks from GitHub. A PR merge shall automatically update the status of the linked Linear issue (e.g., to "Done").

### FR3: Enhanced Memory Layer

-   **FR3.1: Vector Store:** The system shall use a Qdrant vector database for semantic search across documents, conversations, and code. This enables "what did we say about X?" queries.
-   **FR3.2: Knowledge Graph:** The system shall use a Neo4j graph database to model entities (decisions, people, products, epics, tickets) and their relationships. This enables "why did we choose technology Y for product Z?" queries.
-   **FR3.3: Auto-Indexing Pipeline:** An automated pipeline shall be created to ingest, process, and index data from Telegram, Linear, GitHub, and BMAD artifacts into both the vector store and knowledge graph in near real-time.

### FR4: Custom Interface Layer

-   **FR4.1: Web Dashboard:**
    -   A remotely accessible web application shall provide a "single pane of glass" view.
    -   It must display a real-time sprint board, showing the status of epics and issues across Linear and GitHub.
    -   It must include a monitor for all active agents, showing their current task, logs, and progress.
-   **FR4.2: VS Code Extension:**
    -   An extension shall provide an "Ask CruzBot" interface directly within the IDE.
    -   It shall offer context-aware actions (e.g., "explain this code," "generate unit tests").
    -   The extension must have access to the full enhanced memory layer.

## 6. Non-Functional Requirements (NFRs)

| Category | Requirement | Metric/Target |
| :--- | :--- | :--- |
| **Performance** | API latency for event triggers (e.g., Linear webhook to agent start). | **P95 < 500ms** |
| | Knowledge graph query response time. | **P95 < 2 seconds** |
| **Reliability** | Orchestration service uptime. | **> 99.9%** |
| | All stateful services (Redis, Qdrant, Neo4j, Postgres) must have automated daily backups and a tested restore procedure. | **100% backup coverage** |
| **Security** | All inter-service communication must be authenticated. | **mTLS or API keys** |
| | All external API keys and secrets must be stored in a secure vault (e.g., Doppler, HashiCorp Vault), not in code or config files. | **No secrets in git** |
| **Scalability** | The system must handle the workload for all 5 products without performance degradation. | **Maintain performance targets under full load** |
| **Observability** | The orchestration layer must generate structured logs and metrics (e.g., Prometheus) for workflow duration, success rates, and errors. | **Dashboards in Grafana/Datadog** |

## 7. Technical Constraints

-   The solution **must not** fork the OpenClaw repository. It must be built as a separate layer that consumes OpenClaw as a dependency.
-   All new services (Orchestrator, Qdrant, Neo4j) must be containerized (Docker) for consistent deployment.
-   Initial deployment will be on existing self-hosted hardware. Cloud deployment is out of scope for the initial 6-month project.

## 8. Out of Scope

-   Building a new messaging layer (will continue to use OpenClaw's channel integrations).
-   Replacing core OpenClaw tools (e.g., `exec`, `browser`). The project will orchestrate these tools, not replace them.
-   Cloud deployment and management (e.g., Kubernetes, Terraform).
