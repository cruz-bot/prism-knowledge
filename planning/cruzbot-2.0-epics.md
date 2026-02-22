# Epics: CruzBot 2.0 Evolution

**Author:** BMAD PM Agent
**Date:** February 20, 2026
**Version:** 1.0

---

## Overview

This document breaks down the CruzBot 2.0 Evolution project into six shippable epics. The epics are sequenced based on dependencies to ensure incremental value delivery, starting with the foundational orchestration layer and progressively adding intelligence and user-facing features. The breakdown is derived from the [CruzBot 2.0 Product Requirements Document](./cruzbot-2.0-prd.md).

---

## Epic 1: Foundational Orchestration Layer

**Priority:** P0
**Effort:** L
**Goal:** Build the core engine that can manage and execute the BMAD development lifecycle. This is the backbone for all future automation.
**Dependencies:** None
**Risks:** High complexity in designing a robust and extensible state machine. Getting the core abstractions wrong could require significant rework later.

### Key Stories & Acceptance Criteria

-   **Story 1.1: Develop BMAD Workflow Engine:**
    -   **As a** system, I need a state machine that can model and transition through the BMAD phases (Analyst, PM, etc.).
    -   **AC:** The engine can successfully execute a hardcoded "happy path" workflow from start to finish.
-   **Story 1.2: Implement Redis-based Task Queue:**
    -   **As an** orchestrator, I need to be able to enqueue, dequeue, and route tasks to specific agent types.
    -   **AC:** Tasks can be added to a Redis queue and consumed by a worker process. The system can handle at least two different task types (e.g., `dev_task`, `qa_task`).
-   **Story 1.3: Implement Agent/Model Router:**
    -   **As an** orchestrator, I need a rules-based system to select the correct agent and LLM for a given task.
    -   **AC:** A task defined as "high_complexity" is routed to an `opus` model, while a "low_complexity" task is routed to a `flash` model.

---

## Epic 2: Event-Driven Integrations (Linear & GitHub)

**Priority:** P0
**Effort:** L
**Goal:** Automate the handoff between planning, project management, and source control, eliminating the most painful manual processes.
**Dependencies:** Epic 1
**Risks:** API instability or rate limiting from Linear/GitHub. Webhook handling can be complex to test and debug.

### Key Stories & Acceptance Criteria

-   **Story 2.1: Implement BMAD -> Linear Creation:**
    -   **As a** PM agent, I want my approved PRD and Epics to automatically generate corresponding issues in Linear.
    -   **AC:** A successful BMAD planning run triggers an API call that creates an Epic and several Issues in the correct Linear team.
-   **Story 2.2: Implement Linear -> Agent Trigger:**
    -   **As a** user, when I move a Linear issue to "In Progress," I want the CruzBot Dev Agent to automatically pick it up and start work.
    -   **AC:** A Linear webhook for an issue status change successfully places a `dev_task` in the Redis queue.
-   **Story 2.3: Implement Agent -> GitHub & Linear Sync:**
    -   **As a** Dev Agent, when I work on a task, I need to create a feature branch, commit my code, and open a PR linked to the Linear issue.
    -   **AC:** The agent successfully creates a PR in GitHub, and the PR link is posted as a comment in the originating Linear issue.
-   **Story 2.4: Implement GitHub -> Linear Trigger:**
    -   **As a** user, when a PR is merged, I want the linked Linear issue to be automatically marked as "Done".
    -   **AC:** A GitHub webhook for a `merged` PR event triggers a status update on the correct Linear issue.

---

## Epic 3: Vector Memory & Auto-Indexing

**Priority:** P1
**Effort:** M
**Goal:** Provide the agent with a basic long-term semantic memory, enabling it to recall relevant information from past conversations and documents.
**Dependencies:** Epic 1
**Risks:** Choosing the right embedding model and chunking strategy is critical for good performance and can require experimentation.

### Key Stories & Acceptance Criteria

-   **Story 3.1: Deploy Qdrant Service:**
    -   **As an** operator, I need a Qdrant vector database service running and accessible to the orchestration layer.
    -   **AC:** The Qdrant service is running in Docker and the orchestrator can successfully connect to it.
-   **Story 3.2: Create Auto-Indexing Pipeline for Documents:**
    -   **As a** system, I want all new and updated Markdown files in the workspace (especially BMAD artifacts) to be automatically indexed.
    -   **AC:** When a `.md` file is created or modified, its content is chunked, embedded, and stored in Qdrant within 1 minute.
-   **Story 3.3: Integrate Semantic Search into Agents:**
    -   **As an** agent, before starting a task, I need to perform a semantic search for relevant context from my memory.
    -   **AC:** Agents are observed querying the Qdrant database with the task description and receiving relevant context snippets.

---

## Epic 4: Knowledge Graph for Relational Reasoning

**Priority:** P1
**Effort:** L
**Goal:** Evolve the agent's memory from simple recall to relational understanding, allowing it to answer "why" questions.
**Dependencies:** Epic 3
**Risks:** The learning curve for Neo4j and Cypher is non-trivial. Designing a useful and scalable graph schema can be challenging.

### Key Stories & Acceptance Criteria

-   **Story 4.1: Deploy Neo4j Service:**
    -   **As an** operator, I need a Neo4j graph database service running and accessible.
    -   **AC:** The Neo4j service is running in Docker and its browser interface is accessible.
-   **Story 4.2: Define and Implement Graph Schema:**
    -   **As an** architect, I need a graph schema that models key entities (Product, Epic, Issue, Decision, Person, PR) and their relationships (e.g., `IMPLEMENTS`, `DECIDED_BY`, `RELATES_TO`).
    -   **AC:** The schema is documented, and nodes/relationships can be created in the database.
-   **Story 4.3: Extend Indexing Pipeline to KG:**
    -   **As a** system, when a Linear issue is created or a PR is merged, I need to create corresponding nodes and relationships in the knowledge graph.
    -   **AC:** A new Linear issue creates an `Issue` node linked to an `Epic` node. A merged PR creates a `PR` node linked to the `Issue` node.
-   **Story 4.4: Implement "Why" Query Capability:**
    -   **As a** user, I want to ask the agent "Why was feature X built?" and get an answer summarizing the associated Epic and decisions.
    -   **AC:** The agent can translate the user's question into a Cypher query, traverse the graph, and synthesize a coherent answer from the results.

---

## Epic 5: Web Dashboard (Single Pane of Glass)

**Priority:** P2
**Effort:** M
**Goal:** Provide a centralized UI for Tony to monitor workflows and agent activity without switching between multiple tools.
**Dependencies:** Epic 2
**Risks:** Frontend development can be time-consuming. Achieving a "Linear-like" feel requires significant attention to UX/UI details.

### Key Stories & Acceptance Criteria

-   **Story 5.1: Build Real-time Sprint Board:**
    -   **As** Tony, I want to see a unified board view of all in-progress work, pulling data from Linear and GitHub.
    -   **AC:** The dashboard displays columns (e.g., "Todo," "In Progress," "In Review") with cards representing issues. Each card shows the issue title, assignee, and PR status.
-   **Story 5.2: Build Active Agent Monitor:**
    -   **As** Tony, I want a view that shows all currently running agents, their active tasks, and a stream of their latest logs.
    -   **AC:** A dashboard widget lists active agents. Clicking an agent reveals its current goal and a real-time, tailing log output.
-   **Story 5.3: Ensure Remote Accessibility:**
    -   **As** Tony, I need to be able to access this dashboard securely from outside my local network.
    -   **AC:** The web dashboard is deployed with a public URL and is protected by an authentication layer (e.g., Cloudflare Access, OAuth2-proxy).

---

## Epic 6: VS Code Extension

**Priority:** P2
**Effort:** M
**Goal:** Bring CruzBot's intelligence directly into the development environment, providing context-aware assistance.
**Dependencies:** Epic 4
**Risks:** VS Code extension development has its own complexities and lifecycle. Ensuring the extension has performant access to the memory layer is critical.

### Key Stories & Acceptance Criteria

-   **Story 6.1: Implement "Ask CruzBot" Chat Panel:**
    -   **As a** developer in VS Code, I want a chat panel where I can ask CruzBot questions about my current codebase.
    -   **AC:** A new panel in the VS Code sidebar allows users to have a conversation with an agent that has access to the Enhanced Memory Layer.
-   **Story 6.2: Create Context-Aware Code Actions:**
    -   **As a** developer, when I highlight a function, I want to see code actions like "Explain this code" or "Generate unit tests."
    -   **AC:** Right-clicking on a block of code and selecting "Generate unit tests" results in the agent creating a new `_test.go` or `.test.ts` file with relevant tests.
