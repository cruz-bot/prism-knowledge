# Technical Architecture: CruzBot 2.0 Evolution

**Project:** CruzBot 2.0 Evolution - Autonomous Development Partner  
**Author:** BMAD Architect Agent  
**Date:** February 20, 2026  
**Version:** 1.0

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [System Context & Goals](#system-context--goals)
3. [Architecture Principles](#architecture-principles)
4. [High-Level Architecture](#high-level-architecture)
5. [Technology Stack & Dependencies](#technology-stack--dependencies)
6. [Component Architecture](#component-architecture)
7. [Data Models & Schemas](#data-models--schemas)
8. [API Design](#api-design)
9. [Security & Authentication](#security--authentication)
10. [Performance & Scalability Considerations](#performance--scalability-considerations)
11. [Migration Strategy](#migration-strategy)
12. [Testing Strategy](#testing-strategy)
13. [Deployment & Operations](#deployment--operations)
14. [Epic-Level Technical Breakdown](#epic-level-technical-breakdown)
15. [Risk Mitigation & Architectural Decision Records](#risk-mitigation--architectural-decision-records)

---

## Executive Summary

CruzBot 2.0 Evolution transforms the current OpenClaw agent from a reactive command-driven tool into a **proactive, autonomous development orchestration platform**. This architecture document defines a hybrid orchestration layer built **on top** of the OpenClaw gateway runtime—not as a fork—that manages the complete BMAD (Build, Measure, Analyze, Decide) development lifecycle across Tony's five-product portfolio.

**Core Architectural Pillars:**

1. **Orchestration Layer (CruzBot Core)**: BMAD workflow state machine with Redis-based task queue and intelligent agent/model routing
2. **Integration Hub**: Event-driven bidirectional sync with Linear and GitHub via webhooks
3. **Enhanced Memory Layer**: Dual-mode memory with Qdrant (semantic search) + Neo4j (knowledge graph)
4. **Custom Interface Layer**: Web dashboard and VS Code extension for workflow visibility

**Key Architectural Decisions:**

- **Non-forking design**: CruzBot 2.0 consumes OpenClaw as a library dependency, preserving upstream compatibility
- **Event-driven architecture**: Webhooks + Redis task queue eliminate manual coordination
- **Hybrid memory system**: Vector embeddings for "what was said" + knowledge graph for "why we decided"
- **Subagent-based orchestration**: Each BMAD phase runs as an isolated OpenClaw subagent
- **Model routing optimization**: Opus for planning/architecture, Sonnet for development, Flash for simple tasks

**Success Metrics:**

- Feature delivery time: **2-5 days → 2-4 hours** (10x improvement)
- Manual steps per feature: **15-20 → 0-2** (95% automation)
- Agent autonomy: **20% → 80%** (4x increase in agent-driven work)

---

## System Context & Goals

### Problem Statement

The current CruzBot 1.0 architecture exhibits several bottlenecks:

1. **Manual Integration Layer**: `bmad-to-linear.js` script requires human trigger and lacks error recovery
2. **Static Workflow**: No event-driven automation; status changes in Linear don't trigger agents
3. **Shallow Memory**: Flat Markdown files lack semantic search and relational context
4. **Generic UI**: No workflow-specific dashboard for sprint/agent monitoring
5. **Manual Model Selection**: No automated routing based on task complexity

### Target Audience

**Primary User:** Tony (sole developer, product owner, architect for 5 products)

**Usage Scenarios:**
- **High-velocity feature development**: Ship features in hours, not days
- **Cross-product architecture**: Maintain consistency across 5 codebases
- **Context preservation**: "Why did we choose X for product Y three months ago?"
- **Autonomous execution**: Agent handles routine dev tasks end-to-end

### Architectural Goals

1. **Full BMAD Automation**: Eliminate all manual scripts and human touchpoints
2. **Event-Driven Coordination**: Webhooks trigger agent workflows automatically
3. **Deep Contextual Understanding**: Relational memory for "why" questions
4. **Workflow Visibility**: Single-pane-of-glass dashboard for monitoring
5. **Intelligent Routing**: Automatic agent/model selection based on task complexity

---

## Architecture Principles

### 1. **OpenClaw-Native Design**
- Build **on top** of OpenClaw, not as a fork
- Leverage OpenClaw's gateway, session management, and tool execution
- Use OpenClaw subagents as the primitive for BMAD phase execution
- Follow OpenClaw's hub-and-spoke architecture pattern

### 2. **Event-Driven First**
- Webhooks trigger workflows, not cron jobs or polling
- Redis Streams for reliable, ordered task delivery
- Idempotent event handlers for at-least-once delivery semantics
- Push-based completion announcements (no polling)

### 3. **Hybrid Memory Architecture**
- **Vector store (Qdrant)**: Semantic search for "what was discussed"
- **Knowledge graph (Neo4j)**: Relational reasoning for "why we decided"
- **Auto-indexing pipeline**: Near real-time ingestion from all sources
- **Temporal awareness**: Track when decisions were made and who made them

### 4. **Security by Design**
- All inter-service communication authenticated (API keys or mTLS)
- Secrets stored in Doppler/HashiCorp Vault (never in code)
- Webhook payload verification (HMAC signatures)
- Rate limiting on all external-facing endpoints

### 5. **Observability from Day One**
- Structured logging (JSON) with correlation IDs
- Prometheus metrics for workflow duration, success rate, errors
- Distributed tracing via OpenTelemetry
- Pre-built Grafana dashboards for operators

### 6. **Graceful Degradation**
- Linear/GitHub outages don't block agent workflows
- Memory layer failures fall back to session-only context
- Dashboard unavailable doesn't prevent CLI/agent operation

---

## High-Level Architecture

### System Overview Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              User Interfaces                                │
│  ┌────────────┐  ┌────────────────┐  ┌──────────────┐  ┌────────────────┐ │
│  │ Telegram   │  │ Web Dashboard  │  │ VS Code Ext  │  │ CLI            │ │
│  └──────┬─────┘  └───────┬────────┘  └──────┬───────┘  └────────┬───────┘ │
└─────────┼─────────────────┼────────────────────┼──────────────────┼──────────┘
          │                 │                    │                  │
          ▼                 ▼                    ▼                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         OpenClaw Gateway (Existing)                         │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │  Gateway Control Plane (ws://127.0.0.1:18789)                         │ │
│  │  - Session Management                                                 │ │
│  │  - Channel Adapters (Telegram, Discord, etc.)                         │ │
│  │  - Agent Runtime (PiEmbeddedRunner)                                   │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      CruzBot 2.0 Orchestration Layer                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  BMAD Workflow Engine (Node.js + TypeScript)                        │   │
│  │  - State Machine (Analyst → PM → Architect → SM → Dev → QA)        │   │
│  │  - Task Queue (Redis Streams)                                        │   │
│  │  - Agent/Model Router (Rule-based complexity analysis)              │   │
│  │  - Subagent Spawner (OpenClaw sessions_spawn integration)           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Integration Hub (Express.js + Webhooks)                            │   │
│  │  - Linear Webhook Handler → Task Queue                              │   │
│  │  - GitHub Webhook Handler → Task Queue                              │   │
│  │  - Linear API Client (Bidirectional Sync)                           │   │
│  │  - GitHub API Client (PR creation, status updates)                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└───────┬───────────────────────────────────────────────────┬─────────────────┘
        │                                                   │
        ▼                                                   ▼
┌────────────────────────────────┐      ┌──────────────────────────────────┐
│  Enhanced Memory Layer         │      │  External Integrations           │
│  ┌──────────────────────────┐  │      │  ┌────────────────────────────┐ │
│  │  Qdrant (Vector Store)   │  │      │  │  Linear API                │ │
│  │  - Semantic search       │  │      │  │  - Issues, Epics, Comments │ │
│  │  - Document embeddings   │  │      │  │  - Webhooks (outbound)     │ │
│  └──────────────────────────┘  │      │  └────────────────────────────┘ │
│  ┌──────────────────────────┐  │      │  ┌────────────────────────────┐ │
│  │  Neo4j (Knowledge Graph) │  │      │  │  GitHub API                │ │
│  │  - Entity relationships  │  │      │  │  - Repos, PRs, Branches    │ │
│  │  - Temporal tracking     │  │      │  │  - Webhooks (outbound)     │ │
│  └──────────────────────────┘  │      │  └────────────────────────────┘ │
│  ┌──────────────────────────┐  │      └──────────────────────────────────┘
│  │  Auto-Indexing Pipeline  │  │
│  │  - Telegram listener     │  │
│  │  - BMAD artifact watcher │  │
│  │  - Embedding service     │  │
│  └──────────────────────────┘  │
└────────────────────────────────┘

        ┌────────────────────────────────────┐
        │  Data Persistence                  │
        │  ┌──────────────────────────────┐  │
        │  │  PostgreSQL                  │  │
        │  │  - Workflow state            │  │
        │  │  - Task queue metadata       │  │
        │  │  - Audit logs                │  │
        │  └──────────────────────────────┘  │
        │  ┌──────────────────────────────┐  │
        │  │  Redis                       │  │
        │  │  - Task queue (Streams)      │  │
        │  │  - Workflow locks            │  │
        │  │  - Rate limiting             │  │
        │  └──────────────────────────────┘  │
        └────────────────────────────────────┘
```

### Data Flow: Feature Development Lifecycle

```
1. PM Agent (Opus) completes Epic → BMAD artifacts created
2. Orchestrator pushes Epic to Linear API → Linear Issue created
3. Tony moves Linear Issue to "In Progress" → Linear webhook fires
4. Integration Hub receives webhook → Validates signature → Enqueues task
5. Task Queue (Redis) dispatches to Dev Agent (Sonnet)
6. Dev Agent spawned as OpenClaw subagent → Reads story file
7. Dev Agent implements feature → Creates branch, commits, opens PR
8. PR creation → GitHub webhook fires → Integration Hub updates Linear
9. PR merged → GitHub webhook fires → Linear Issue → "Done"
10. All interactions auto-indexed to Qdrant + Neo4j
```

**Critical Path Optimizations:**
- Webhook handlers respond in <100ms (enqueue + return 202)
- Task queue processing in <500ms (spawn subagent)
- Parallel BMAD phases (Analyst + PM can run concurrently when no dependencies)

---

## Technology Stack & Dependencies

### Core Runtime

| Component | Technology | Version | Rationale |
|-----------|-----------|---------|-----------|
| **OpenClaw Gateway** | Node.js 22 + TypeScript | Latest | Core agent runtime; battle-tested session management |
| **Orchestration Engine** | Node.js 22 + TypeScript | N/A | Consistency with OpenClaw; strong async primitives |
| **Task Queue** | Redis Streams | 7.x | Persistent, ordered, distributed queue; native Node support |
| **State Machine** | XState v5 | 5.x | Type-safe, visualizable state machines; strong TypeScript support |
| **API Server** | Express.js | 4.x | Lightweight, proven; easy webhook handling |

### Memory Layer

| Component | Technology | Version | Rationale |
|-----------|-----------|---------|-----------|
| **Vector Database** | Qdrant | 1.7+ | Native Docker deployment; excellent semantic search; OpenClaw MCP server available |
| **Knowledge Graph** | Neo4j | 5.x | Industry-standard graph DB; Cypher query language; Graphiti integration patterns |
| **Embedding Model** | OpenAI text-embedding-3-small | Latest | Cost-effective; 1536 dimensions; excellent recall |
| **Auto-Indexer** | Custom Node.js Service | N/A | File watcher + webhook listener; batched processing |

### Integrations

| Component | Technology | Version | Rationale |
|-----------|-----------|---------|-----------|
| **Linear Client** | @linear/sdk | Latest | Official Linear SDK; TypeScript types |
| **GitHub Client** | Octokit (REST) | Latest | Official GitHub SDK; REST API stability |
| **Webhook Validation** | crypto (Node.js) | Built-in | HMAC-SHA256 signature verification |

### Data Persistence

| Component | Technology | Version | Rationale |
|-----------|-----------|---------|-----------|
| **Primary Database** | PostgreSQL | 16+ | ACID compliance; JSON support; robust backup/restore |
| **ORM** | Prisma | 5.x | Type-safe queries; migrations; strong TypeScript support |
| **Caching/Queue** | Redis | 7.x | Multi-purpose: task queue, rate limiting, workflow locks |

### Observability

| Component | Technology | Version | Rationale |
|-----------|-----------|---------|-----------|
| **Metrics** | Prometheus + prom-client | Latest | Industry standard; pull-based metrics |
| **Logging** | Pino (JSON) | Latest | Fast structured logging; correlation ID support |
| **Tracing** | OpenTelemetry | Latest | Distributed tracing across services |
| **Dashboards** | Grafana | Latest | Pre-built dashboards for ops visibility |

### Frontend

| Component | Technology | Version | Rationale |
|-----------|-----------|---------|-----------|
| **Web Dashboard** | Next.js 14 (App Router) | 14.x | Server components; streaming; excellent DX |
| **UI Framework** | shadcn/ui + Tailwind | Latest | Accessible components; customizable |
| **Real-time Updates** | Server-Sent Events (SSE) | Built-in | Simpler than WebSocket for server→client updates |
| **VS Code Extension** | VS Code Extension API | Latest | Official API; TypeScript support |

### Deployment

| Component | Technology | Version | Rationale |
|-----------|-----------|---------|-----------|
| **Container Runtime** | Docker + Docker Compose | Latest | Consistent local + production deployment |
| **Process Manager** | PM2 | Latest | Process monitoring; auto-restart; log management |
| **Reverse Proxy** | Nginx (optional) | Latest | SSL termination; rate limiting |

---

## Component Architecture

### 1. BMAD Workflow Engine

**Location:** `cruzbot-core/workflow-engine/`  
**Responsibility:** State machine orchestration for BMAD lifecycle phases

**Core Modules:**

#### 1.1 State Machine Definition (`state-machine.ts`)

```typescript
import { createMachine, assign } from 'xstate';

export const bmadWorkflowMachine = createMachine({
  id: 'bmad-workflow',
  initial: 'planning',
  context: {
    projectId: '',
    productName: '',
    artifacts: {},
    currentPhase: null,
    errors: [],
  },
  states: {
    planning: {
      initial: 'analyst',
      states: {
        analyst: {
          invoke: {
            src: 'runAnalystAgent',
            onDone: {
              target: 'pm',
              actions: assign({
                artifacts: (ctx, event) => ({
                  ...ctx.artifacts,
                  productBrief: event.data.output,
                }),
              }),
            },
            onError: { target: '#bmad-workflow.failed' },
          },
        },
        pm: {
          invoke: {
            src: 'runPMAgent',
            onDone: {
              target: 'architect',
              actions: assign({
                artifacts: (ctx, event) => ({
                  ...ctx.artifacts,
                  prd: event.data.output.prd,
                  epics: event.data.output.epics,
                }),
              }),
            },
            onError: { target: '#bmad-workflow.failed' },
          },
        },
        architect: {
          invoke: {
            src: 'runArchitectAgent',
            onDone: {
              target: '#bmad-workflow.readinessGate',
              actions: assign({
                artifacts: (ctx, event) => ({
                  ...ctx.artifacts,
                  architecture: event.data.output,
                }),
              }),
            },
            onError: { target: '#bmad-workflow.failed' },
          },
        },
      },
    },
    readinessGate: {
      invoke: {
        src: 'checkImplementationReadiness',
        onDone: [
          { target: 'storyGeneration', cond: 'artifactsComplete' },
          { target: 'failed', actions: 'logReadinessFailure' },
        ],
      },
    },
    storyGeneration: {
      invoke: {
        src: 'runSMAgent',
        onDone: {
          target: 'linearPopulation',
          actions: assign({
            artifacts: (ctx, event) => ({
              ...ctx.artifacts,
              stories: event.data.output,
            }),
          }),
        },
        onError: { target: 'failed' },
      },
    },
    linearPopulation: {
      invoke: {
        src: 'populateLinearIssues',
        onDone: { target: 'devReady' },
        onError: { target: 'failed' },
      },
    },
    devReady: {
      type: 'final',
      entry: 'notifyDevReadyEvent',
    },
    failed: {
      type: 'final',
      entry: 'notifyFailureEvent',
    },
  },
});
```

**Key Design Decisions:**
- **XState for visualization**: State machine can be visualized with @xstate/inspect
- **Immutable context**: All state transitions create new context objects
- **Service invocations**: Each BMAD phase invoked as async service
- **Error boundaries**: Phase failures don't crash entire workflow

#### 1.2 Agent Spawner (`agent-spawner.ts`)

```typescript
import { OpenClawClient } from '../openclaw-client';

export class AgentSpawner {
  constructor(private openclawClient: OpenClawClient) {}

  async spawnAgent(params: {
    agentType: 'analyst' | 'pm' | 'architect' | 'sm' | 'dev' | 'qa';
    task: string;
    model: string;
    thinking: 'high' | 'medium' | 'low';
    dependencies?: string[];
  }): Promise<{ runId: string; sessionKey: string }> {
    const modelMap = {
      analyst: 'google-antigravity/claude-opus-4-6',
      pm: 'google-antigravity/claude-opus-4-6',
      architect: 'google-antigravity/claude-opus-4-6',
      sm: 'google-gemini-cli/gemini-2.5-pro', // Large context for epic → stories
      dev: 'anthropic/claude-sonnet-4-5',
      qa: 'anthropic/claude-sonnet-4-5',
    };

    // Inject dependencies into task description
    const taskWithDeps = this.buildTaskWithDependencies(params.task, params.dependencies);

    // Spawn OpenClaw subagent via sessions_spawn tool
    const result = await this.openclawClient.spawnSubagent({
      task: taskWithDeps,
      label: `BMAD ${params.agentType.toUpperCase()} Agent`,
      model: params.model || modelMap[params.agentType],
      thinking: params.thinking || 'high',
      runTimeoutSeconds: 600, // 10 min timeout per phase
      cleanup: 'delete', // Auto-cleanup after completion
    });

    return {
      runId: result.runId,
      sessionKey: result.sessionKey,
    };
  }

  private buildTaskWithDependencies(task: string, dependencies?: string[]): string {
    if (!dependencies || dependencies.length === 0) return task;

    const depsList = dependencies
      .map((path) => `- Read \`${path}\``)
      .join('\n');

    return `${task}\n\n**Required inputs:**\n${depsList}\n\nRead all dependencies before starting your work.`;
  }
}
```

**Integration with OpenClaw:**
- Uses OpenClaw's `sessions_spawn` tool via RPC
- Each BMAD phase = one OpenClaw subagent session
- Subagent completion announcements trigger state machine transitions
- Automatic cleanup after phase completion

#### 1.3 Model Router (`model-router.ts`)

```typescript
export class ModelRouter {
  /**
   * Select optimal model based on task complexity
   */
  selectModel(task: {
    type: 'planning' | 'architecture' | 'development' | 'qa' | 'simple';
    contextSize: number; // Number of tokens in context
    requiresReasoning: boolean;
  }): string {
    // Planning & Architecture → Opus (deep reasoning)
    if (task.type === 'planning' || task.type === 'architecture') {
      return 'google-antigravity/claude-opus-4-6';
    }

    // Large context (>100k tokens) → Gemini (2M context window)
    if (task.contextSize > 100000) {
      return 'google-gemini-cli/gemini-2.5-pro';
    }

    // Development & QA → Sonnet (balance of quality/speed)
    if (task.type === 'development' || task.type === 'qa') {
      return 'anthropic/claude-sonnet-4-5';
    }

    // Simple tasks → Flash (speed/cost optimized)
    if (task.type === 'simple' && !task.requiresReasoning) {
      return 'google-gemini-cli/gemini-2.5-flash';
    }

    // Default fallback
    return 'anthropic/claude-sonnet-4-5';
  }

  /**
   * Estimate task complexity from description
   */
  analyzeComplexity(taskDescription: string): {
    type: string;
    contextSize: number;
    requiresReasoning: boolean;
  } {
    const keywords = {
      planning: ['prd', 'requirements', 'epic', 'strategy'],
      architecture: ['architecture', 'design', 'system', 'component'],
      development: ['implement', 'code', 'feature', 'fix'],
      qa: ['test', 'qa', 'quality', 'validate'],
    };

    let type = 'simple';
    for (const [key, terms] of Object.entries(keywords)) {
      if (terms.some((term) => taskDescription.toLowerCase().includes(term))) {
        type = key;
        break;
      }
    }

    const requiresReasoning = /why|analyze|decide|choose/i.test(taskDescription);
    const contextSize = taskDescription.length * 4; // Rough estimate

    return { type, contextSize, requiresReasoning };
  }
}
```

---

### 2. Task Queue (Redis Streams)

**Location:** `cruzbot-core/task-queue/`  
**Responsibility:** Persistent, ordered task distribution to agents

#### 2.1 Queue Producer (`producer.ts`)

```typescript
import { Redis } from 'ioredis';
import { v4 as uuidv4 } from 'uuid';

export class TaskQueue {
  private redis: Redis;
  private streamKey = 'cruzbot:tasks';

  constructor(redisUrl: string) {
    this.redis = new Redis(redisUrl);
  }

  async enqueue(task: {
    type: 'dev_task' | 'qa_task' | 'webhook_event';
    priority: 'high' | 'normal' | 'low';
    payload: Record<string, unknown>;
    correlationId?: string;
  }): Promise<string> {
    const taskId = uuidv4();
    const timestamp = Date.now();

    await this.redis.xadd(
      this.streamKey,
      '*', // Auto-generate entry ID
      'taskId',
      taskId,
      'type',
      task.type,
      'priority',
      task.priority,
      'payload',
      JSON.stringify(task.payload),
      'correlationId',
      task.correlationId || taskId,
      'enqueuedAt',
      timestamp.toString()
    );

    return taskId;
  }

  async enqueueWithDelay(
    task: Parameters<typeof this.enqueue>[0],
    delayMs: number
  ): Promise<string> {
    // Store in Redis with TTL, then move to stream after delay
    const taskId = uuidv4();
    const key = `cruzbot:delayed:${taskId}`;
    
    await this.redis.setex(
      key,
      Math.ceil(delayMs / 1000),
      JSON.stringify(task)
    );

    // Schedule background job to move to stream
    setTimeout(async () => {
      const stored = await this.redis.get(key);
      if (stored) {
        await this.enqueue(JSON.parse(stored));
        await this.redis.del(key);
      }
    }, delayMs);

    return taskId;
  }
}
```

#### 2.2 Queue Consumer (`consumer.ts`)

```typescript
export class TaskConsumer {
  private redis: Redis;
  private consumerGroup = 'cruzbot-workers';
  private consumerName = `worker-${process.pid}`;
  private streamKey = 'cruzbot:tasks';

  async start() {
    // Create consumer group if it doesn't exist
    try {
      await this.redis.xgroup('CREATE', this.streamKey, this.consumerGroup, '0', 'MKSTREAM');
    } catch (err) {
      // Group already exists
    }

    // Start consuming
    while (true) {
      const results = await this.redis.xreadgroup(
        'GROUP',
        this.consumerGroup,
        this.consumerName,
        'COUNT',
        1, // Process one at a time
        'BLOCK',
        5000, // 5-second block
        'STREAMS',
        this.streamKey,
        '>' // Only new messages
      );

      if (results && results.length > 0) {
        const [_streamName, messages] = results[0];
        for (const [messageId, fields] of messages) {
          await this.processTask(messageId, this.parseFields(fields));
        }
      }
    }
  }

  private async processTask(messageId: string, task: Task) {
    try {
      const handler = this.getHandler(task.type);
      await handler(task);

      // Acknowledge successful processing
      await this.redis.xack(this.streamKey, this.consumerGroup, messageId);
    } catch (error) {
      // Log error, task remains in pending state for retry
      console.error('Task processing failed:', error);
      // TODO: Implement retry logic with exponential backoff
    }
  }

  private getHandler(taskType: string): TaskHandler {
    const handlers = {
      dev_task: this.handleDevTask.bind(this),
      qa_task: this.handleQATask.bind(this),
      webhook_event: this.handleWebhookEvent.bind(this),
    };
    return handlers[taskType];
  }
}
```

**Key Features:**
- **Consumer groups**: Multiple workers can process tasks in parallel
- **At-least-once delivery**: Tasks remain in pending until acknowledged
- **Automatic retry**: Failed tasks can be reprocessed after timeout
- **Priority queues**: High-priority tasks processed first (separate streams)

---

### 3. Integration Hub

**Location:** `cruzbot-core/integrations/`  
**Responsibility:** Webhook handlers + API clients for Linear and GitHub

#### 3.1 Linear Webhook Handler (`linear-webhook.ts`)

```typescript
import express from 'express';
import crypto from 'crypto';

export class LinearWebhookHandler {
  constructor(
    private taskQueue: TaskQueue,
    private linearSecret: string
  ) {}

  handleWebhook = async (req: express.Request, res: express.Response) => {
    // Verify webhook signature
    const signature = req.headers['linear-signature'] as string;
    const isValid = this.verifySignature(req.body, signature);

    if (!isValid) {
      return res.status(401).json({ error: 'Invalid signature' });
    }

    const event = req.body;
    const eventType = event.type;

    // Respond immediately (within 100ms)
    res.status(202).json({ received: true });

    // Process asynchronously
    try {
      if (eventType === 'Issue' && event.action === 'update') {
        const issue = event.data;
        const statusChanged = event.updatedFrom?.state?.name !== issue.state.name;

        if (statusChanged && issue.state.name === 'In Progress') {
          // Trigger Dev Agent
          await this.taskQueue.enqueue({
            type: 'dev_task',
            priority: 'normal',
            payload: {
              issueId: issue.id,
              issueIdentifier: issue.identifier,
              title: issue.title,
              description: issue.description,
              storyPath: issue.description.match(/Story: `(.+)`/)?.[1],
            },
            correlationId: `linear-${issue.id}`,
          });
        }

        if (statusChanged && issue.state.name === 'In Review') {
          // Trigger QA Agent (optional)
          await this.taskQueue.enqueue({
            type: 'qa_task',
            priority: 'normal',
            payload: {
              issueId: issue.id,
              prUrl: issue.branchName, // Extract from linked PR
            },
          });
        }
      }
    } catch (error) {
      console.error('Linear webhook processing error:', error);
      // Error is logged but webhook already acknowledged
    }
  };

  private verifySignature(body: unknown, signature: string): boolean {
    const hmac = crypto.createHmac('sha256', this.linearSecret);
    const digest = hmac.update(JSON.stringify(body)).digest('hex');
    return crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(digest));
  }
}
```

#### 3.2 GitHub Webhook Handler (`github-webhook.ts`)

```typescript
import { Webhooks } from '@octokit/webhooks';

export class GitHubWebhookHandler {
  private webhooks: Webhooks;

  constructor(
    private linearClient: LinearClient,
    private taskQueue: TaskQueue,
    secret: string
  ) {
    this.webhooks = new Webhooks({ secret });
    this.registerHandlers();
  }

  private registerHandlers() {
    this.webhooks.on('pull_request.opened', async ({ payload }) => {
      const prNumber = payload.pull_request.number;
      const prUrl = payload.pull_request.html_url;
      const branchName = payload.pull_request.head.ref;

      // Extract Linear issue ID from branch name (e.g., "cruz-123-feature")
      const issueMatch = branchName.match(/cruz-(\d+)/i);
      if (issueMatch) {
        const issueIdentifier = `CRUZ-${issueMatch[1]}`;

        // Update Linear issue with PR link
        await this.linearClient.addComment(issueIdentifier, {
          body: `🔗 Pull Request opened: [#${prNumber}](${prUrl})`,
        });
      }
    });

    this.webhooks.on('pull_request.closed', async ({ payload }) => {
      if (!payload.pull_request.merged) return;

      const branchName = payload.pull_request.head.ref;
      const issueMatch = branchName.match(/cruz-(\d+)/i);

      if (issueMatch) {
        const issueIdentifier = `CRUZ-${issueMatch[1]}`;

        // Update Linear issue to "Done"
        await this.linearClient.updateIssueState(issueIdentifier, 'Done');

        // Add completion comment
        await this.linearClient.addComment(issueIdentifier, {
          body: `✅ Pull Request merged! Issue automatically marked as Done.`,
        });
      }
    });
  }

  middleware() {
    return this.webhooks.middleware;
  }
}
```

**Security Features:**
- **HMAC signature verification**: Both Linear and GitHub webhooks verified
- **Timing-safe comparison**: Prevents timing attacks on signature validation
- **Rate limiting**: Express middleware limits webhook endpoint to 100 req/min
- **Idempotency**: Duplicate webhooks (retries) don't create duplicate tasks

---

### 4. Enhanced Memory Layer

#### 4.1 Qdrant Vector Store Integration (`memory/vector-store.ts`)

```typescript
import { QdrantClient } from '@qdrant/js-client-rest';
import { OpenAIEmbeddings } from 'langchain/embeddings/openai';

export class VectorMemoryStore {
  private client: QdrantClient;
  private embeddings: OpenAIEmbeddings;
  private collectionName = 'cruzbot-memory';

  constructor(qdrantUrl: string) {
    this.client = new QdrantClient({ url: qdrantUrl });
    this.embeddings = new OpenAIEmbeddings({
      modelName: 'text-embedding-3-small',
      dimensions: 1536,
    });
  }

  async initialize() {
    // Create collection if it doesn't exist
    const collections = await this.client.getCollections();
    const exists = collections.collections.some((c) => c.name === this.collectionName);

    if (!exists) {
      await this.client.createCollection(this.collectionName, {
        vectors: {
          size: 1536,
          distance: 'Cosine',
        },
        optimizers_config: {
          indexing_threshold: 10000,
        },
      });
    }
  }

  async indexDocument(doc: {
    id: string;
    text: string;
    metadata: {
      source: 'telegram' | 'bmad-artifact' | 'linear' | 'github';
      timestamp: number;
      userId?: string;
      product?: string;
      type?: string;
    };
  }) {
    const embedding = await this.embeddings.embedQuery(doc.text);

    await this.client.upsert(this.collectionName, {
      points: [
        {
          id: doc.id,
          vector: embedding,
          payload: {
            text: doc.text,
            ...doc.metadata,
          },
        },
      ],
    });
  }

  async semanticSearch(params: {
    query: string;
    limit?: number;
    filter?: Record<string, unknown>;
  }): Promise<Array<{ text: string; score: number; metadata: Record<string, unknown> }>> {
    const queryEmbedding = await this.embeddings.embedQuery(params.query);

    const results = await this.client.search(this.collectionName, {
      vector: queryEmbedding,
      limit: params.limit || 10,
      filter: params.filter,
    });

    return results.map((result) => ({
      text: result.payload.text as string,
      score: result.score,
      metadata: result.payload,
    }));
  }

  async hybridSearch(params: {
    query: string;
    keywords?: string[];
    limit?: number;
  }) {
    // Combine vector search with BM25 keyword search
    const vectorResults = await this.semanticSearch(params);

    if (params.keywords && params.keywords.length > 0) {
      // TODO: Implement BM25 keyword search + result fusion
    }

    return vectorResults;
  }
}
```

#### 4.2 Neo4j Knowledge Graph (`memory/knowledge-graph.ts`)

```typescript
import neo4j, { Driver, Session } from 'neo4j-driver';

export class KnowledgeGraph {
  private driver: Driver;

  constructor(uri: string, user: string, password: string) {
    this.driver = neo4j.driver(uri, neo4j.auth.basic(user, password));
  }

  async initialize() {
    const session = this.driver.session();
    try {
      // Create constraints
      await session.run(`
        CREATE CONSTRAINT product_name IF NOT EXISTS
        FOR (p:Product) REQUIRE p.name IS UNIQUE
      `);
      await session.run(`
        CREATE CONSTRAINT epic_id IF NOT EXISTS
        FOR (e:Epic) REQUIRE e.id IS UNIQUE
      `);
      await session.run(`
        CREATE CONSTRAINT issue_id IF NOT EXISTS
        FOR (i:Issue) REQUIRE i.id IS UNIQUE
      `);
    } finally {
      await session.close();
    }
  }

  async recordDecision(params: {
    decisionId: string;
    title: string;
    description: string;
    madeBy: string;
    relatedTo: { type: string; id: string }[];
    timestamp: number;
  }) {
    const session = this.driver.session();
    try {
      await session.run(
        `
        CREATE (d:Decision {
          id: $decisionId,
          title: $title,
          description: $description,
          madeBy: $madeBy,
          timestamp: $timestamp
        })
        WITH d
        UNWIND $relatedTo AS relation
        MATCH (entity)
        WHERE entity.id = relation.id AND relation.type IN labels(entity)
        CREATE (d)-[:RELATES_TO]->(entity)
        `,
        params
      );
    } finally {
      await session.close();
    }
  }

  async recordEpic(params: {
    epicId: string;
    title: string;
    productName: string;
    prdPath: string;
    architecturePath: string;
  }) {
    const session = this.driver.session();
    try {
      await session.run(
        `
        MERGE (p:Product {name: $productName})
        CREATE (e:Epic {
          id: $epicId,
          title: $title,
          prdPath: $prdPath,
          architecturePath: $architecturePath,
          createdAt: timestamp()
        })
        CREATE (e)-[:BELONGS_TO]->(p)
        `,
        params
      );
    } finally {
      await session.close();
    }
  }

  async recordIssue(params: {
    issueId: string;
    issueIdentifier: string;
    title: string;
    epicId: string;
    storyPath?: string;
  }) {
    const session = this.driver.session();
    try {
      await session.run(
        `
        MATCH (e:Epic {id: $epicId})
        CREATE (i:Issue {
          id: $issueId,
          identifier: $issueIdentifier,
          title: $title,
          storyPath: $storyPath,
          createdAt: timestamp()
        })
        CREATE (i)-[:IMPLEMENTS]->(e)
        `,
        params
      );
    } finally {
      await session.close();
    }
  }

  async recordPullRequest(params: {
    prId: string;
    prNumber: number;
    title: string;
    url: string;
    issueId: string;
    mergedAt?: number;
  }) {
    const session = this.driver.session();
    try {
      await session.run(
        `
        MATCH (i:Issue {id: $issueId})
        CREATE (pr:PullRequest {
          id: $prId,
          number: $prNumber,
          title: $title,
          url: $url,
          mergedAt: $mergedAt,
          createdAt: timestamp()
        })
        CREATE (pr)-[:CLOSES]->(i)
        `,
        params
      );
    } finally {
      await session.close();
    }
  }

  async queryWhy(question: string): Promise<string> {
    // Example: "Why was feature X built for product Y?"
    // This would use LLM to parse question → Cypher query → synthesize answer
    
    // Simplified example:
    const session = this.driver.session();
    try {
      const result = await session.run(`
        MATCH path = (i:Issue)-[:IMPLEMENTS]->(e:Epic)-[:BELONGS_TO]->(p:Product)
        WHERE i.title CONTAINS $keyword OR e.title CONTAINS $keyword
        MATCH (d:Decision)-[:RELATES_TO]->(e)
        RETURN path, collect(d) as decisions
        LIMIT 5
      `, { keyword: question });

      // Synthesize answer from graph traversal results
      // TODO: Use LLM to generate natural language answer
      return JSON.stringify(result.records, null, 2);
    } finally {
      await session.close();
    }
  }

  async close() {
    await this.driver.close();
  }
}
```

**Graph Schema:**

```cypher
// Nodes
(:Product {name, description})
(:Epic {id, title, prdPath, architecturePath, createdAt})
(:Issue {id, identifier, title, storyPath, createdAt})
(:PullRequest {id, number, title, url, mergedAt, createdAt})
(:Decision {id, title, description, madeBy, timestamp})
(:Person {name, role})

// Relationships
(:Epic)-[:BELONGS_TO]->(:Product)
(:Issue)-[:IMPLEMENTS]->(:Epic)
(:PullRequest)-[:CLOSES]->(:Issue)
(:Decision)-[:RELATES_TO]->(:Epic)
(:Decision)-[:MADE_BY]->(:Person)
(:Epic)-[:DEPENDS_ON]->(:Epic)
```

#### 4.3 Auto-Indexing Pipeline (`memory/auto-indexer.ts`)

```typescript
import { watch } from 'chokidar';
import { readFile } from 'fs/promises';
import { EventEmitter } from 'events';

export class AutoIndexer extends EventEmitter {
  constructor(
    private vectorStore: VectorMemoryStore,
    private knowledgeGraph: KnowledgeGraph
  ) {
    super();
  }

  startFileWatcher(watchPath: string) {
    const watcher = watch(watchPath, {
      persistent: true,
      ignoreInitial: true,
      awaitWriteFinish: {
        stabilityThreshold: 2000,
        pollInterval: 100,
      },
    });

    watcher.on('add', async (path) => {
      if (path.endsWith('.md')) {
        await this.indexMarkdownFile(path);
      }
    });

    watcher.on('change', async (path) => {
      if (path.endsWith('.md')) {
        await this.indexMarkdownFile(path);
      }
    });
  }

  private async indexMarkdownFile(filePath: string) {
    const content = await readFile(filePath, 'utf-8');
    const metadata = this.extractMetadata(filePath);

    // Index to vector store
    await this.vectorStore.indexDocument({
      id: `file:${filePath}`,
      text: content,
      metadata: {
        source: 'bmad-artifact',
        timestamp: Date.now(),
        ...metadata,
      },
    });

    // Extract structured entities for knowledge graph
    if (metadata.type === 'prd') {
      // TODO: Extract Epic information and create nodes
    }
    if (metadata.type === 'architecture') {
      // TODO: Extract architectural decisions
    }

    this.emit('indexed', { filePath, metadata });
  }

  private extractMetadata(filePath: string): Record<string, unknown> {
    // Extract product name, type (prd/architecture/epic/story) from path
    const pathParts = filePath.split('/');
    const fileName = pathParts[pathParts.length - 1];

    return {
      product: this.extractProductName(filePath),
      type: this.extractDocType(fileName),
      filePath,
    };
  }

  private extractProductName(path: string): string {
    // e.g., "_bmad-output/cruzbot-2.0/..." → "cruzbot"
    const match = path.match(/_bmad-output\/([^\/]+)/);
    return match ? match[1].split('-')[0] : 'unknown';
  }

  private extractDocType(fileName: string): string {
    if (fileName.includes('prd')) return 'prd';
    if (fileName.includes('architecture')) return 'architecture';
    if (fileName.includes('epic')) return 'epic';
    if (fileName.match(/story-\d+/)) return 'story';
    return 'unknown';
  }

  async indexTelegramMessage(message: {
    text: string;
    from: string;
    timestamp: number;
  }) {
    await this.vectorStore.indexDocument({
      id: `telegram:${message.timestamp}`,
      text: message.text,
      metadata: {
        source: 'telegram',
        timestamp: message.timestamp,
        userId: message.from,
      },
    });
  }
}
```

---

### 5. Web Dashboard

**Location:** `cruzbot-dashboard/`  
**Tech Stack:** Next.js 14 (App Router) + shadcn/ui + Server-Sent Events

#### 5.1 Real-time Sprint Board (`app/sprint/page.tsx`)

```typescript
// Server Component
import { LinearClient } from '@linear/sdk';
import { SprintBoard } from '@/components/SprintBoard';

export default async function SprintPage() {
  const linearClient = new LinearClient({ apiKey: process.env.LINEAR_API_KEY });
  
  // Fetch current sprint issues
  const issues = await linearClient.issues({
    filter: {
      state: { type: { in: ['started', 'unstarted'] } },
    },
  });

  const columns = [
    { id: 'todo', title: 'To Do', issues: [] },
    { id: 'in-progress', title: 'In Progress', issues: [] },
    { id: 'in-review', title: 'In Review', issues: [] },
    { id: 'done', title: 'Done', issues: [] },
  ];

  // Organize issues by state
  for (const issue of issues.nodes) {
    const state = issue.state?.name.toLowerCase();
    const column = columns.find((c) => 
      state?.includes(c.id.replace('-', ' '))
    );
    if (column) column.issues.push(issue);
  }

  return <SprintBoard initialData={columns} />;
}
```

#### 5.2 Active Agent Monitor (`app/agents/page.tsx`)

```typescript
'use client';

import { useEffect, useState } from 'react';
import { AgentCard } from '@/components/AgentCard';

export default function AgentsPage() {
  const [agents, setAgents] = useState([]);

  useEffect(() => {
    // Subscribe to Server-Sent Events
    const eventSource = new EventSource('/api/agents/stream');

    eventSource.onmessage = (event) => {
      const data = JSON.parse(event.data);
      setAgents(data.agents);
    };

    return () => eventSource.close();
  }, []);

  return (
    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
      {agents.map((agent) => (
        <AgentCard key={agent.runId} agent={agent} />
      ))}
    </div>
  );
}
```

#### 5.3 SSE Endpoint (`app/api/agents/stream/route.ts`)

```typescript
export async function GET(req: Request) {
  const stream = new ReadableStream({
    async start(controller) {
      // Send agent status every 2 seconds
      const interval = setInterval(async () => {
        const agents = await getActiveAgents();
        const data = `data: ${JSON.stringify({ agents })}\n\n`;
        controller.enqueue(new TextEncoder().encode(data));
      }, 2000);

      // Cleanup on disconnect
      req.signal.addEventListener('abort', () => {
        clearInterval(interval);
        controller.close();
      });
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  });
}
```

---

### 6. VS Code Extension

**Location:** `vscode-extension/`  
**Tech Stack:** VS Code Extension API + TypeScript

#### 6.1 Extension Main (`src/extension.ts`)

```typescript
import * as vscode from 'vscode';
import { CruzBotClient } from './client';

export function activate(context: vscode.ExtensionContext) {
  const client = new CruzBotClient(
    vscode.workspace.getConfiguration('cruzbot').get('apiUrl')
  );

  // Ask CruzBot command
  const askCommand = vscode.commands.registerCommand('cruzbot.ask', async () => {
    const question = await vscode.window.showInputBox({
      prompt: 'Ask CruzBot a question',
      placeHolder: 'e.g., Why did we choose Redis for the task queue?',
    });

    if (question) {
      const panel = vscode.window.createWebviewPanel(
        'cruzbotAnswer',
        'CruzBot Answer',
        vscode.ViewColumn.Beside,
        {}
      );

      const answer = await client.ask(question);
      panel.webview.html = getWebviewContent(answer);
    }
  });

  // Generate tests command
  const genTestsCommand = vscode.commands.registerCommand(
    'cruzbot.generateTests',
    async () => {
      const editor = vscode.window.activeTextEditor;
      if (!editor) return;

      const selection = editor.selection;
      const code = editor.document.getText(selection);

      const tests = await client.generateTests(code, editor.document.languageId);

      // Create new test file
      const testFileName = editor.document.fileName.replace(/\.(ts|js)$/, '.test.$1');
      const testUri = vscode.Uri.file(testFileName);
      
      await vscode.workspace.fs.writeFile(
        testUri,
        new TextEncoder().encode(tests)
      );

      vscode.window.showInformationMessage(`Tests generated: ${testFileName}`);
    }
  );

  context.subscriptions.push(askCommand, genTestsCommand);
}
```

---

## Data Models & Schemas

### PostgreSQL Schema (Prisma)

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model WorkflowRun {
  id            String   @id @default(cuid())
  projectId     String
  productName   String
  status        String   // 'planning' | 'dev-ready' | 'in-progress' | 'completed' | 'failed'
  currentPhase  String?  // 'analyst' | 'pm' | 'architect' | 'sm' | 'dev' | 'qa'
  startedAt     DateTime @default(now())
  completedAt   DateTime?
  artifacts     Json     // { productBrief, prd, epics, architecture, stories }
  context       Json     // XState context snapshot
  
  tasks         Task[]
  
  @@index([status])
  @@index([productName])
}

model Task {
  id              String   @id @default(cuid())
  workflowRunId   String?
  type            String   // 'dev_task' | 'qa_task' | 'webhook_event'
  priority        String   // 'high' | 'normal' | 'low'
  status          String   // 'queued' | 'processing' | 'completed' | 'failed'
  payload         Json
  correlationId   String?
  
  enqueuedAt      DateTime @default(now())
  startedAt       DateTime?
  completedAt     DateTime?
  error           String?
  
  workflowRun     WorkflowRun? @relation(fields: [workflowRunId], references: [id])
  
  @@index([status])
  @@index([correlationId])
  @@index([type])
}

model LinearSync {
  id              String   @id @default(cuid())
  issueId         String   @unique
  issueIdentifier String
  epicId          String?
  workflowRunId   String?
  prUrl           String?
  lastSyncedAt    DateTime @default(now())
  
  @@index([issueIdentifier])
  @@index([epicId])
}

model GitHubSync {
  id              String   @id @default(cuid())
  repoFullName    String
  prNumber        Int
  prId            String   @unique
  issueId         String?
  mergedAt        DateTime?
  lastSyncedAt    DateTime @default(now())
  
  @@index([repoFullName, prNumber])
  @@index([issueId])
}

model AuditLog {
  id              String   @id @default(cuid())
  event           String   // 'workflow_started' | 'task_completed' | 'webhook_received' | etc.
  entityType      String   // 'workflow' | 'task' | 'issue' | 'pr'
  entityId        String
  userId          String?
  metadata        Json
  timestamp       DateTime @default(now())
  
  @@index([event])
  @@index([entityType, entityId])
  @@index([timestamp])
}
```

### Redis Data Structures

**Task Queue (Redis Streams):**

```
Stream: cruzbot:tasks
Entry: {
  taskId: "uuid",
  type: "dev_task",
  priority: "normal",
  payload: "{...}",
  correlationId: "linear-12345",
  enqueuedAt: "1645564800000"
}

Consumer Group: cruzbot-workers
Consumers: worker-<pid>
```

**Workflow Locks (Redis Keys):**

```
Key: cruzbot:lock:workflow:<projectId>
Value: "worker-12345"
TTL: 600 (10 minutes)
```

**Rate Limiting (Redis Sorted Sets):**

```
Key: cruzbot:ratelimit:webhook:<ip>
Score: timestamp
Members: request IDs
```

---

## API Design

### Internal APIs

#### 1. BMAD Workflow API

**Start Workflow:**
```http
POST /api/workflows/start
Content-Type: application/json
Authorization: Bearer <token>

{
  "projectId": "cruzbot-2.0-feature-x",
  "productName": "CruzBot",
  "productBrief": "...",
  "skipPhases": []
}

Response 201:
{
  "workflowId": "cuid",
  "status": "planning",
  "startedAt": "2026-02-20T12:00:00Z"
}
```

**Get Workflow Status:**
```http
GET /api/workflows/:id

Response 200:
{
  "id": "cuid",
  "status": "planning",
  "currentPhase": "architect",
  "artifacts": {
    "productBrief": "path/to/file",
    "prd": "path/to/file"
  },
  "progress": 60
}
```

#### 2. Memory Query API

**Semantic Search:**
```http
POST /api/memory/search
Content-Type: application/json

{
  "query": "Why did we choose Redis for task queue?",
  "filters": {
    "product": "CruzBot",
    "source": ["telegram", "bmad-artifact"]
  },
  "limit": 10
}

Response 200:
{
  "results": [
    {
      "text": "...",
      "score": 0.92,
      "metadata": { "source": "telegram", "timestamp": 1645564800000 }
    }
  ]
}
```

**Knowledge Graph Query:**
```http
POST /api/memory/graph/query
Content-Type: application/json

{
  "question": "Why was feature X built for product Y?",
  "maxDepth": 3
}

Response 200:
{
  "answer": "Feature X was built because...",
  "reasoning": "...",
  "relatedEntities": [...]
}
```

### External Webhook Endpoints

**Linear Webhook:**
```http
POST /webhooks/linear
Content-Type: application/json
Linear-Signature: sha256=...

{
  "type": "Issue",
  "action": "update",
  "data": { ... }
}

Response 202:
{ "received": true }
```

**GitHub Webhook:**
```http
POST /webhooks/github
Content-Type: application/json
X-Hub-Signature-256: sha256=...

{
  "action": "opened",
  "pull_request": { ... }
}

Response 202:
{ "received": true }
```

---

## Security & Authentication

### 1. API Authentication

**Internal Services:**
- **API Keys**: Each service (dashboard, VS Code extension) has unique API key
- **Storage**: Keys stored in Doppler/HashiCorp Vault
- **Rotation**: Keys rotated every 90 days via automated script
- **Validation**: JWT tokens with 24-hour expiration

**OpenClaw Gateway:**
- **Device Pairing**: All remote connections require device approval
- **Token-based Auth**: `OPENCLAW_GATEWAY_TOKEN` for non-loopback connections
- **Tailscale Integration**: Optional Tailscale Serve for secure remote access

### 2. Webhook Security

**Signature Verification:**
```typescript
function verifyLinearWebhook(body: string, signature: string): boolean {
  const hmac = crypto.createHmac('sha256', process.env.LINEAR_WEBHOOK_SECRET);
  const digest = hmac.update(body).digest('hex');
  return crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(digest));
}
```

**Rate Limiting:**
- **Per IP**: 100 requests/hour for webhook endpoints
- **Per API key**: 1000 requests/hour for internal APIs
- **Burst tolerance**: 10 requests/second spike allowed

### 3. Secrets Management

**Architecture:**

```
┌─────────────────────────────────────┐
│  Application (CruzBot Services)     │
│  - Fetches secrets at startup       │
│  - Refreshes every 5 minutes        │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│  Doppler CLI / API                  │
│  - Encrypted secrets storage        │
│  - Audit logging                    │
│  - Access control                   │
└─────────────────────────────────────┘
```

**Secrets Inventory:**
- `OPENCLAW_GATEWAY_TOKEN`
- `LINEAR_API_KEY`
- `LINEAR_WEBHOOK_SECRET`
- `GITHUB_TOKEN`
- `GITHUB_WEBHOOK_SECRET`
- `DATABASE_URL`
- `REDIS_URL`
- `QDRANT_API_KEY`
- `NEO4J_PASSWORD`
- `OPENAI_API_KEY`

### 4. Network Security

**Service Communication:**
- **Internal**: All services communicate over `127.0.0.1` (loopback)
- **External**: HTTPS only (TLS 1.3)
- **Firewall**: Only webhook endpoints (`:3000/webhooks/*`) exposed externally

**Docker Network Isolation:**
```yaml
# docker-compose.yml
networks:
  internal:
    driver: bridge
    internal: true  # No external access
  external:
    driver: bridge  # Internet access for webhooks

services:
  orchestrator:
    networks:
      - internal
      - external
  
  qdrant:
    networks:
      - internal  # Only accessible to orchestrator
```

---

## Performance & Scalability Considerations

### Performance Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Webhook Response Time** | P95 < 100ms | Enqueue + return 202 |
| **Task Processing Latency** | P95 < 500ms | Dequeue → spawn subagent |
| **Semantic Search** | P95 < 2s | Qdrant query + result formatting |
| **Knowledge Graph Query** | P95 < 3s | Cypher query + LLM synthesis |
| **E2E Feature Delivery** | **< 4 hours** | BMAD planning → PR merged |

### Scalability Architecture

**Horizontal Scaling:**

```
┌────────────────────────────────────────────────────────┐
│  Load Balancer (Nginx)                                 │
│  - Round-robin to webhook handlers                    │
│  - Health checks every 10s                            │
└───────────┬────────────────────────────────────────────┘
            │
            ▼
┌──────────────────────────┐  ┌──────────────────────────┐
│  Webhook Handler 1       │  │  Webhook Handler 2       │
│  - Express.js            │  │  - Express.js            │
│  - Stateless (no memory) │  │  - Stateless (no memory) │
└──────────┬───────────────┘  └──────────┬───────────────┘
           │                              │
           └──────────────┬───────────────┘
                          ▼
            ┌──────────────────────────┐
            │  Redis (Task Queue)      │
            │  - Shared state          │
            │  - Consumer groups       │
            └──────────┬───────────────┘
                       │
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
┌────────────┐ ┌────────────┐ ┌────────────┐
│ Worker 1   │ │ Worker 2   │ │ Worker N   │
│ (Consumer) │ │ (Consumer) │ │ (Consumer) │
└────────────┘ └────────────┘ └────────────┘
```

**Bottleneck Analysis:**

1. **OpenClaw Gateway** (single instance):
   - **Bottleneck**: Single process handles all agent sessions
   - **Mitigation**: Subagents run in parallel (concurrency = 8)
   - **Future**: Multi-gateway architecture (not in scope for v1)

2. **Redis Task Queue**:
   - **Bottleneck**: Single Redis instance
   - **Mitigation**: Redis Streams handle 10k+ msgs/sec easily
   - **Future**: Redis Cluster for multi-million msg/sec

3. **Qdrant Vector Search**:
   - **Bottleneck**: Query latency increases with index size
   - **Mitigation**: HNSW index (sub-second queries for 1M vectors)
   - **Monitoring**: P95 latency alerts at 2s threshold

4. **Neo4j Knowledge Graph**:
   - **Bottleneck**: Complex graph traversals (3+ hops)
   - **Mitigation**: Index on commonly-queried properties
   - **Caching**: Query result cache (5-minute TTL)

### Caching Strategy

**Multi-Layer Cache:**

```
Request → Application Cache (in-memory) → Redis Cache → Database
          ├─ TTL: 1 minute           → TTL: 5 minutes  → Source of truth
          └─ LRU eviction
```

**Cache Keys:**
- `workflow:status:<id>` → Workflow status (1 min TTL)
- `linear:issue:<id>` → Linear issue metadata (5 min TTL)
- `memory:search:<query_hash>` → Semantic search results (10 min TTL)

---

## Migration Strategy

### Phase 1: Parallel Operation (Weeks 1-2)

**Goal:** CruzBot 2.0 runs alongside CruzBot 1.0 without disruption

**Implementation:**
1. Deploy CruzBot 2.0 orchestration layer on separate port (`:3001`)
2. Configure separate Redis instance (`redis://localhost:6380`)
3. Use separate database (`cruzbot_v2`)
4. Test with single product (lowest-risk product)

**Validation:**
- [ ] BMAD workflow completes end-to-end for test product
- [ ] Linear sync works (create issue, update status)
- [ ] GitHub sync works (PR creation, merge detection)
- [ ] Memory indexing pipeline running (Qdrant + Neo4j populated)

### Phase 2: Incremental Cutover (Weeks 3-4)

**Goal:** Migrate products one-by-one to CruzBot 2.0

**Migration Checklist (per product):**
1. Export existing BMAD artifacts to `_bmad-output/<product>/`
2. Run initial memory indexing batch job
3. Create Linear team/project mapping in config
4. Test full workflow on staging branch
5. Monitor for 48 hours
6. Cut over production traffic

**Rollback Plan:**
- Keep CruzBot 1.0 scripts available for 2 weeks post-cutover
- Database snapshots before each product migration
- Feature flag to disable CruzBot 2.0 per product

### Phase 3: Decommission Legacy (Week 5)

**Goal:** Remove CruzBot 1.0 dependencies

**Tasks:**
- [ ] Delete `bmad-to-linear.js` manual script
- [ ] Remove cron jobs that triggered old workflows
- [ ] Archive CruzBot 1.0 database to cold storage
- [ ] Update documentation to reflect v2 workflows
- [ ] Team training on new dashboard/VS Code extension

### Data Migration

**BMAD Artifacts:**
```bash
# One-time migration script
node scripts/migrate-artifacts.js \
  --source ~/old-bmad-output \
  --dest ~/.openclaw/workspace/_bmad-output \
  --index  # Trigger auto-indexing
```

**Linear Historical Data:**
```bash
# Backfill Linear issues into knowledge graph
node scripts/backfill-linear.js \
  --team-id CRUZ \
  --since 2025-01-01 \
  --dry-run
```

**Validation Queries:**
```cypher
// Verify knowledge graph population
MATCH (p:Product)<-[:BELONGS_TO]-(e:Epic)<-[:IMPLEMENTS]-(i:Issue)
RETURN p.name, count(e) as epics, count(i) as issues
```

---

## Testing Strategy

### 1. Unit Testing

**Framework:** Vitest  
**Coverage Target:** 85%

**Test Structure:**
```typescript
// cruzbot-core/workflow-engine/__tests__/state-machine.test.ts
import { describe, it, expect } from 'vitest';
import { bmadWorkflowMachine } from '../state-machine';
import { interpret } from 'xstate';

describe('BMAD Workflow State Machine', () => {
  it('should transition from analyst → pm → architect', async () => {
    const service = interpret(bmadWorkflowMachine).start();

    // Mock agent responses
    service.send({
      type: 'ANALYST_COMPLETE',
      data: { output: 'product-brief.md' },
    });

    expect(service.state.value).toBe('planning.pm');
  });

  it('should fail gracefully on architect error', async () => {
    const service = interpret(bmadWorkflowMachine).start();

    service.send({ type: 'ARCHITECT_ERROR', error: 'Network timeout' });

    expect(service.state.value).toBe('failed');
    expect(service.state.context.errors).toContain('Network timeout');
  });
});
```

**Critical Test Coverage:**
- State machine transitions (all paths)
- Webhook signature verification
- Task queue enqueue/dequeue
- Memory indexing pipeline
- API authentication

### 2. Integration Testing

**Framework:** Playwright + Docker Compose  
**Scope:** Multi-service workflows

**Test Setup:**
```yaml
# docker-compose.test.yml
services:
  redis:
    image: redis:7
  postgres:
    image: postgres:16
  qdrant:
    image: qdrant/qdrant:v1.7.0
  neo4j:
    image: neo4j:5
  orchestrator:
    build: ./cruzbot-core
    environment:
      - NODE_ENV=test
    depends_on:
      - redis
      - postgres
```

**Integration Test Example:**
```typescript
// tests/integration/linear-webhook.test.ts
describe('Linear Webhook → Dev Agent Flow', () => {
  it('should spawn Dev Agent when issue moved to In Progress', async () => {
    // 1. Send webhook
    const response = await fetch('http://localhost:3000/webhooks/linear', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Linear-Signature': generateSignature(payload),
      },
      body: JSON.stringify({
        type: 'Issue',
        action: 'update',
        data: {
          id: 'test-issue-123',
          state: { name: 'In Progress' },
        },
      }),
    });

    expect(response.status).toBe(202);

    // 2. Wait for task processing
    await waitForCondition(
      () => checkSubagentSpawned('dev', 'test-issue-123'),
      5000
    );

    // 3. Verify subagent received correct context
    const subagent = await getSubagent('dev', 'test-issue-123');
    expect(subagent.task).toContain('test-issue-123');
  });
});
```

### 3. End-to-End Testing

**Scope:** Full BMAD lifecycle (planning → deployment)

**Test Scenario:**
```gherkin
Feature: Autonomous Feature Development
  Scenario: Ship feature from PRD to Production
    Given a new product brief for "Feature X"
    When the BMAD workflow is triggered
    Then the Analyst agent creates a product brief
    And the PM agent creates PRD + Epics
    And the Architect agent creates architecture document
    And the SM agent generates story files
    And stories are populated in Linear
    And a dev agent is spawned for the first story
    And the dev agent creates a PR
    And the PR is merged
    Then the Linear issue is marked "Done"
    And the knowledge graph contains decision nodes
```

**E2E Test Runner:**
```typescript
// tests/e2e/feature-delivery.test.ts
describe('Full Feature Delivery Lifecycle', () => {
  it('should complete BMAD workflow end-to-end', async () => {
    const startTime = Date.now();

    // 1. Start workflow
    const workflow = await startWorkflow({
      productBrief: 'Build a new dashboard widget',
    });

    // 2. Wait for planning phase
    await waitForPhase(workflow.id, 'devReady', 120000); // 2 min timeout

    // 3. Simulate Linear status change
    await simulateLinearUpdate(workflow.epicId, 'In Progress');

    // 4. Wait for PR creation
    const pr = await waitForPR(workflow.repoName, 60000); // 1 min timeout

    // 5. Merge PR
    await mergePR(pr.number);

    // 6. Verify Linear sync
    const issue = await getLinearIssue(workflow.issueId);
    expect(issue.state).toBe('Done');

    const duration = Date.now() - startTime;
    expect(duration).toBeLessThan(4 * 60 * 60 * 1000); // < 4 hours
  }, 300000); // 5-minute test timeout
});
```

### 4. Load Testing

**Tool:** k6  
**Target:** Webhook endpoint capacity

```javascript
// tests/load/webhook-load.js
import http from 'k6/http';
import { check } from 'k6';

export let options = {
  stages: [
    { duration: '1m', target: 10 },  // Ramp up to 10 RPS
    { duration: '3m', target: 50 },  // Ramp up to 50 RPS
    { duration: '1m', target: 100 }, // Spike to 100 RPS
    { duration: '2m', target: 0 },   // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<100'], // 95% under 100ms
    http_req_failed: ['rate<0.01'],   // <1% error rate
  },
};

export default function () {
  const payload = JSON.stringify({
    type: 'Issue',
    action: 'update',
    data: { id: `test-${__VU}-${__ITER}` },
  });

  const res = http.post('http://localhost:3000/webhooks/linear', payload, {
    headers: {
      'Content-Type': 'application/json',
      'Linear-Signature': generateSignature(payload),
    },
  });

  check(res, {
    'status is 202': (r) => r.status === 202,
    'response time < 100ms': (r) => r.timings.duration < 100,
  });
}
```

---

## Deployment & Operations

### Deployment Architecture

**Self-Hosted (Primary):**

```
┌─────────────────────────────────────────────────────────┐
│  Hardware: Mac Mini / Self-Hosted Server                │
│  OS: macOS / Ubuntu 22.04                                │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │  Docker Compose Stack                              │ │
│  │                                                    │ │
│  │  ┌──────────────┐  ┌──────────────┐              │ │
│  │  │ PostgreSQL   │  │ Redis        │              │ │
│  │  └──────────────┘  └──────────────┘              │ │
│  │  ┌──────────────┐  ┌──────────────┐              │ │
│  │  │ Qdrant       │  │ Neo4j        │              │ │
│  │  └──────────────┘  └──────────────┘              │ │
│  │                                                    │ │
│  │  ┌──────────────────────────────────────────────┐ │ │
│  │  │  CruzBot Orchestrator (PM2)                  │ │ │
│  │  │  - Workflow Engine                           │ │ │
│  │  │  - Integration Hub                           │ │ │
│  │  │  - Task Queue Workers                        │ │ │
│  │  └──────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │  OpenClaw Gateway (systemd/launchd)               │ │
│  │  - Agent Runtime                                   │ │
│  │  - Subagent Spawner                                │ │
│  │  - Tool Execution                                  │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

**Docker Compose Configuration:**

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: cruzbot
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "127.0.0.1:5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    ports:
      - "127.0.0.1:6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  qdrant:
    image: qdrant/qdrant:v1.7.0
    environment:
      QDRANT__SERVICE__API_KEY: ${QDRANT_API_KEY}
    volumes:
      - qdrant-data:/qdrant/storage
    ports:
      - "127.0.0.1:6333:6333"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:6333/health"]
      interval: 10s
      timeout: 5s
      retries: 5

  neo4j:
    image: neo4j:5-community
    environment:
      NEO4J_AUTH: neo4j/${NEO4J_PASSWORD}
      NEO4J_dbms_memory_heap_initial__size: 512m
      NEO4J_dbms_memory_heap_max__size: 2G
    volumes:
      - neo4j-data:/data
    ports:
      - "127.0.0.1:7474:7474"  # Web UI
      - "127.0.0.1:7687:7687"  # Bolt
    healthcheck:
      test: ["CMD-SHELL", "cypher-shell -u neo4j -p ${NEO4J_PASSWORD} 'RETURN 1'"]
      interval: 30s
      timeout: 10s
      retries: 5

  orchestrator:
    build:
      context: ./cruzbot-core
      dockerfile: Dockerfile
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/cruzbot
      REDIS_URL: redis://redis:6379
      QDRANT_URL: http://qdrant:6333
      NEO4J_URI: bolt://neo4j:7687
      OPENCLAW_GATEWAY_URL: ws://host.docker.internal:18789
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      qdrant:
        condition: service_healthy
      neo4j:
        condition: service_healthy
    restart: unless-stopped
    extra_hosts:
      - "host.docker.internal:host-gateway"

  dashboard:
    build:
      context: ./cruzbot-dashboard
      dockerfile: Dockerfile
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/cruzbot
      API_URL: http://orchestrator:3000
    ports:
      - "127.0.0.1:3001:3000"
    depends_on:
      - orchestrator
    restart: unless-stopped

volumes:
  postgres-data:
  redis-data:
  qdrant-data:
  neo4j-data:
```

### Process Management (PM2)

**PM2 Ecosystem File:**

```javascript
// ecosystem.config.js
module.exports = {
  apps: [
    {
      name: 'cruzbot-orchestrator',
      script: 'dist/index.js',
      cwd: './cruzbot-core',
      instances: 1,
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production',
      },
      error_file: 'logs/orchestrator-error.log',
      out_file: 'logs/orchestrator-out.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
      max_memory_restart: '1G',
      autorestart: true,
      watch: false,
    },
    {
      name: 'cruzbot-worker',
      script: 'dist/worker.js',
      cwd: './cruzbot-core',
      instances: 2, // 2 worker processes
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production',
      },
      autorestart: true,
      max_memory_restart: '512M',
    },
  ],
};
```

**Start/Stop Commands:**

```bash
# Start all services
pm2 start ecosystem.config.js

# Monitor
pm2 monit

# Logs
pm2 logs cruzbot-orchestrator --lines 100

# Restart
pm2 restart cruzbot-orchestrator

# Stop
pm2 stop all
```

### Backup Strategy

**Daily Automated Backups:**

```bash
#!/bin/bash
# scripts/backup.sh

BACKUP_DIR="/backups/cruzbot/$(date +%Y-%m-%d)"
mkdir -p "$BACKUP_DIR"

# PostgreSQL backup
docker exec cruzbot-postgres pg_dump -U cruzbot cruzbot | \
  gzip > "$BACKUP_DIR/postgres.sql.gz"

# Redis backup (RDB snapshot)
docker exec cruzbot-redis redis-cli SAVE
docker cp cruzbot-redis:/data/dump.rdb "$BACKUP_DIR/redis.rdb"

# Qdrant backup (collection snapshots)
curl -X POST "http://localhost:6333/collections/cruzbot-memory/snapshots" | \
  jq -r '.result.name' | \
  xargs -I {} curl "http://localhost:6333/collections/cruzbot-memory/snapshots/{}" \
    -o "$BACKUP_DIR/qdrant-{}.snapshot"

# Neo4j backup
docker exec cruzbot-neo4j neo4j-admin database dump neo4j \
  --to-path=/backups --verbose
docker cp cruzbot-neo4j:/backups/neo4j.dump "$BACKUP_DIR/neo4j.dump"

# Compress and upload to S3 (optional)
tar -czf "$BACKUP_DIR.tar.gz" "$BACKUP_DIR"
# aws s3 cp "$BACKUP_DIR.tar.gz" s3://cruzbot-backups/
```

**Cron Schedule:**

```cron
# Daily backup at 2 AM
0 2 * * * /path/to/scripts/backup.sh

# Weekly cleanup (retain 30 days)
0 3 * * 0 find /backups/cruzbot -type d -mtime +30 -exec rm -rf {} +
```

### Monitoring & Alerting

**Prometheus Scrape Config:**

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'cruzbot-orchestrator'
    static_configs:
      - targets: ['localhost:9090']
    metrics_path: '/metrics'

  - job_name: 'redis'
    static_configs:
      - targets: ['localhost:9121']

  - job_name: 'postgres'
    static_configs:
      - targets: ['localhost:9187']
```

**Grafana Dashboard (JSON config):**

```json
{
  "dashboard": {
    "title": "CruzBot 2.0 Operations",
    "panels": [
      {
        "title": "Workflow Success Rate",
        "targets": [
          {
            "expr": "rate(cruzbot_workflow_completed_total[5m]) / rate(cruzbot_workflow_started_total[5m])"
          }
        ]
      },
      {
        "title": "Task Queue Depth",
        "targets": [
          {
            "expr": "cruzbot_task_queue_depth"
          }
        ]
      },
      {
        "title": "Webhook Response Time (P95)",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{endpoint=\"/webhooks/*\"}[5m]))"
          }
        ]
      }
    ]
  }
}
```

**Alert Rules:**

```yaml
# alerts.yml
groups:
  - name: cruzbot
    interval: 30s
    rules:
      - alert: WorkflowFailureRate
        expr: rate(cruzbot_workflow_failed_total[5m]) > 0.1
        for: 5m
        annotations:
          summary: "High workflow failure rate"
          description: "{{ $value }}% of workflows failing"

      - alert: TaskQueueBacklog
        expr: cruzbot_task_queue_depth > 100
        for: 10m
        annotations:
          summary: "Task queue backlog"
          description: "{{ $value }} tasks pending"

      - alert: MemoryIndexingLag
        expr: (time() - cruzbot_memory_last_indexed_timestamp) > 600
        for: 5m
        annotations:
          summary: "Memory indexing stopped"
          description: "No indexing for {{ $value }}s"
```

---

## Epic-Level Technical Breakdown

### Epic 1: Foundational Orchestration Layer

**Technical Components:**

1. **BMAD State Machine** (`cruzbot-core/workflow-engine/state-machine.ts`)
   - XState v5 for type-safe state management
   - Event-driven transitions (agent completion → next phase)
   - Context preservation across phases
   - Error boundaries and retry logic

2. **Redis Task Queue** (`cruzbot-core/task-queue/`)
   - Redis Streams for persistent, ordered queue
   - Consumer groups for parallel processing
   - At-least-once delivery semantics
   - Priority queues (separate streams for high/normal/low)

3. **Agent/Model Router** (`cruzbot-core/workflow-engine/model-router.ts`)
   - Rule-based complexity analysis
   - Model selection: Opus (planning) / Gemini (large context) / Sonnet (dev) / Flash (simple)
   - Cost tracking and optimization

4. **OpenClaw Subagent Integration** (`cruzbot-core/openclaw-client.ts`)
   - `sessions_spawn` RPC calls to OpenClaw gateway
   - Completion announcement handling
   - Task injection with dependencies

**Implementation Order:**
1. State machine definition + unit tests
2. Redis queue producer/consumer + integration tests
3. Model router + complexity analysis
4. Subagent spawner + OpenClaw integration
5. E2E workflow test (mock agents)

**Success Criteria:**
- [ ] State machine executes full BMAD cycle (mocked agents)
- [ ] Task queue handles 100 tasks/second
- [ ] Model router selects correct model based on task type
- [ ] Subagents spawn and complete successfully

---

### Epic 2: Event-Driven Integrations (Linear & GitHub)

**Technical Components:**

1. **Linear API Client** (`cruzbot-core/integrations/linear-client.ts`)
   - SDK: `@linear/sdk`
   - Operations: create issue, update state, add comment
   - Retry logic with exponential backoff
   - Rate limiting (100 req/min)

2. **Linear Webhook Handler** (`cruzbot-core/integrations/linear-webhook.ts`)
   - Express middleware
   - HMAC signature verification
   - Event filtering (Issue update → status change)
   - Task queue enqueue (non-blocking)

3. **GitHub API Client** (`cruzbot-core/integrations/github-client.ts`)
   - SDK: `@octokit/rest`
   - Operations: create branch, commit, PR, add comment
   - Branch naming convention: `cruz-{issueNumber}-{slug}`

4. **GitHub Webhook Handler** (`cruzbot-core/integrations/github-webhook.ts`)
   - Webhook SDK: `@octokit/webhooks`
   - Events: `pull_request.opened`, `pull_request.closed` (merged)
   - Linear sync on PR events

5. **Bidirectional Sync Logic** (`cruzbot-core/integrations/sync-manager.ts`)
   - Mapping: Linear Issue ↔ GitHub PR
   - Idempotency: Duplicate webhooks don't create duplicate tasks
   - Conflict resolution: GitHub merge wins over Linear manual update

**Data Flow:**

```
Linear Issue (In Progress) 
  → Webhook 
  → Task Queue 
  → Dev Agent Spawn 
  → Branch + Commits 
  → PR Creation 
  → Linear Comment (PR link)

GitHub PR (Merged) 
  → Webhook 
  → Linear Issue Update (Done) 
  → Linear Comment (✅ merged)
```

**Implementation Order:**
1. Linear API client + unit tests
2. Linear webhook handler + signature verification
3. GitHub API client + unit tests
4. GitHub webhook handler
5. Sync manager + integration tests
6. E2E test (webhook → agent → PR → webhook → Linear)

**Success Criteria:**
- [ ] Linear webhook triggers Dev Agent in <500ms
- [ ] Dev Agent creates PR linked to Linear issue
- [ ] GitHub PR merge auto-updates Linear issue to "Done"
- [ ] Sync works bidirectionally with no data loss

---

### Epic 3: Vector Memory & Auto-Indexing

**Technical Components:**

1. **Qdrant Service Deployment** (`docker-compose.yml`)
   - Qdrant v1.7+ container
   - Persistent volume for index storage
   - API key authentication

2. **Vector Memory Store** (`cruzbot-core/memory/vector-store.ts`)
   - Embedding model: OpenAI `text-embedding-3-small`
   - Collection: `cruzbot-memory` (1536 dimensions, Cosine distance)
   - Operations: index document, semantic search, hybrid search

3. **Auto-Indexing Pipeline** (`cruzbot-core/memory/auto-indexer.ts`)
   - File watcher: `chokidar` (watches `_bmad-output/`)
   - Telegram listener: OpenClaw webhook integration
   - Batch processing: Index files in chunks of 10
   - Metadata extraction: product name, doc type, timestamp

4. **Search Interface** (`cruzbot-core/api/memory-search.ts`)
   - REST endpoint: `POST /api/memory/search`
   - Query parameters: text, filters, limit
   - Response: ranked results with scores and metadata

**Index Schema:**

```typescript
interface VectorDocument {
  id: string; // "file:<path>" or "telegram:<timestamp>"
  vector: number[]; // 1536-dim embedding
  payload: {
    text: string;
    source: 'telegram' | 'bmad-artifact' | 'linear' | 'github';
    timestamp: number;
    userId?: string;
    product?: string;
    type?: 'prd' | 'architecture' | 'epic' | 'story';
    filePath?: string;
  };
}
```

**Implementation Order:**
1. Qdrant deployment + health checks
2. Vector memory store wrapper + unit tests
3. Auto-indexer file watcher + integration tests
4. Telegram message indexing
5. Search API endpoint + E2E tests

**Success Criteria:**
- [ ] Qdrant service running and accessible
- [ ] File watcher indexes new BMAD artifacts within 1 minute
- [ ] Telegram messages indexed in real-time
- [ ] Semantic search returns relevant results (P95 < 2s)

---

### Epic 4: Knowledge Graph for Relational Reasoning

**Technical Components:**

1. **Neo4j Service Deployment** (`docker-compose.yml`)
   - Neo4j v5 community edition
   - Persistent volume for graph storage
   - Auth: username/password

2. **Knowledge Graph Schema** (`cruzbot-core/memory/schema.cypher`)
   - Nodes: Product, Epic, Issue, PullRequest, Decision, Person
   - Relationships: BELONGS_TO, IMPLEMENTS, CLOSES, RELATES_TO, MADE_BY, DEPENDS_ON
   - Constraints: Unique IDs for Epic, Issue, PR

3. **Graph Manager** (`cruzbot-core/memory/knowledge-graph.ts`)
   - Operations: record epic, issue, PR, decision
   - Query builder: LLM-assisted Cypher query generation
   - Temporal tracking: All nodes have `createdAt` timestamp

4. **Indexing Pipeline Extension** (`cruzbot-core/memory/auto-indexer.ts`)
   - Extract entities from BMAD artifacts (PRD → Epic nodes)
   - Extract decisions from architecture docs
   - Link entities via relationships

5. **"Why" Query Interface** (`cruzbot-core/api/knowledge-graph.ts`)
   - REST endpoint: `POST /api/memory/graph/query`
   - LLM-assisted: Parse natural language → Cypher query
   - Response synthesis: Graph traversal results → natural language answer

**Query Examples:**

```cypher
// Why was feature X built for product Y?
MATCH path = (i:Issue {title: "Feature X"})-[:IMPLEMENTS]->(e:Epic)-[:BELONGS_TO]->(p:Product {name: "CruzBot"})
MATCH (d:Decision)-[:RELATES_TO]->(e)
RETURN path, collect(d) as decisions

// What decisions influenced Epic X?
MATCH (e:Epic {id: "epic-123"})
MATCH (d:Decision)-[:RELATES_TO]->(e)
RETURN d.title, d.description, d.madeBy, d.timestamp

// Which PRs implemented Epic X?
MATCH (e:Epic {id: "epic-123"})<-[:IMPLEMENTS]-(i:Issue)<-[:CLOSES]-(pr:PullRequest)
RETURN pr.number, pr.title, pr.url, pr.mergedAt
```

**Implementation Order:**
1. Neo4j deployment + schema creation
2. Graph manager wrapper + unit tests
3. Entity extraction from BMAD artifacts
4. Auto-indexing pipeline integration
5. "Why" query interface + LLM integration
6. E2E test (create epic → record decision → query why)

**Success Criteria:**
- [ ] Neo4j service running with schema constraints
- [ ] Epics/Issues/PRs automatically indexed from Linear/GitHub
- [ ] "Why" queries return accurate answers (>90% accuracy in tests)
- [ ] Graph queries complete in <3s (P95)

---

### Epic 5: Web Dashboard (Single Pane of Glass)

**Technical Components:**

1. **Next.js 14 App** (`cruzbot-dashboard/`)
   - App Router with Server Components
   - SSR for initial page load
   - SSE for real-time updates

2. **Sprint Board View** (`app/sprint/page.tsx`)
   - Data source: Linear API (server component)
   - Columns: To Do, In Progress, In Review, Done
   - Drag-and-drop: Update Linear issue state
   - Real-time sync: SSE updates on webhook events

3. **Active Agent Monitor** (`app/agents/page.tsx`)
   - Data source: OpenClaw gateway (`/api/agents/list`)
   - Display: Running agents, current task, logs
   - Real-time updates: SSE from orchestrator

4. **SSE Endpoints** (`app/api/*/stream/route.ts`)
   - Sprint updates: Emit on Linear webhook
   - Agent updates: Emit on subagent spawn/complete
   - Keep-alive: Send heartbeat every 30s

5. **Authentication** (`lib/auth.ts`)
   - JWT tokens (signed with `AUTH_SECRET`)
   - Session middleware (Next-Auth v5)
   - Protected routes: All dashboard pages

**UI Components (shadcn/ui):**
- `Card` (agent status, issue cards)
- `Badge` (status indicators)
- `Button` (actions)
- `Skeleton` (loading states)

**Implementation Order:**
1. Next.js project setup + authentication
2. Sprint board view (static data)
3. SSE endpoints + real-time updates
4. Active agent monitor
5. Remote accessibility (Tailscale/Cloudflare Access)

**Success Criteria:**
- [ ] Dashboard accessible at `http://localhost:3001`
- [ ] Sprint board displays Linear issues correctly
- [ ] Real-time updates visible within 2 seconds of webhook
- [ ] Agent monitor shows running agents and logs
- [ ] Secure remote access configured

---

### Epic 6: VS Code Extension

**Technical Components:**

1. **Extension Manifest** (`package.json`)
   - Activation events: `onCommand:cruzbot.*`
   - Contributes: Commands, views, keybindings
   - Dependencies: `@vscode/extension-sdk`, `axios`

2. **Ask CruzBot Command** (`src/commands/ask.ts`)
   - Input: User question (input box)
   - API call: `POST /api/memory/search` + `/api/memory/graph/query`
   - Output: Webview panel with formatted answer

3. **Generate Tests Command** (`src/commands/generate-tests.ts`)
   - Input: Selected code (editor selection)
   - API call: OpenClaw agent (`sessions_send` with task)
   - Output: New `.test.ts` file with generated tests

4. **Context-Aware Actions** (`src/code-actions.ts`)
   - Code action provider: "Explain this code", "Generate unit tests"
   - Trigger: Right-click on selection
   - Integration: CruzBot memory API

5. **Settings** (`package.json` contributions)
   - `cruzbot.apiUrl`: Orchestrator API URL
   - `cruzbot.apiKey`: Authentication token

**Implementation Order:**
1. Extension scaffold + basic command
2. Ask CruzBot command + webview
3. Generate tests command
4. Code action provider
5. Settings + configuration
6. Package and publish (VSIX)

**Success Criteria:**
- [ ] Extension activates in VS Code
- [ ] Ask CruzBot returns answers from memory layer
- [ ] Generate tests creates valid test files
- [ ] Code actions appear in right-click menu
- [ ] Extension published to marketplace (optional)

---

## Risk Mitigation & Architectural Decision Records

### ADR-001: Use OpenClaw Subagents (Not Custom Agent Framework)

**Status:** Accepted  
**Date:** 2026-02-20

**Context:**  
CruzBot needs a runtime for executing BMAD phase agents. Two options:
1. Build custom agent framework from scratch
2. Use OpenClaw's subagent system

**Decision:**  
Use OpenClaw subagents via `sessions_spawn` tool.

**Rationale:**
- **Proven runtime**: OpenClaw's session management is battle-tested
- **Tool execution**: Subagents have access to full OpenClaw tool suite
- **Isolation**: Each subagent runs in isolated session
- **Completion announcements**: Push-based completion (no polling)
- **Non-forking design**: Maintains compatibility with OpenClaw upstream

**Consequences:**
- ✅ Faster time to market (no custom runtime)
- ✅ Reliable session management
- ✅ Access to OpenClaw ecosystem (tools, plugins)
- ⚠️ Dependency on OpenClaw release cycle
- ⚠️ Limited customization of subagent behavior

**Risks & Mitigations:**
- **Risk**: OpenClaw breaking changes in subagent API
- **Mitigation**: Pin OpenClaw version; test upgrades in staging

---

### ADR-002: Redis Streams for Task Queue (Not SQS/RabbitMQ)

**Status:** Accepted  
**Date:** 2026-02-20

**Context:**  
Need persistent, ordered task queue for webhook → agent coordination. Options:
1. Redis Streams
2. AWS SQS
3. RabbitMQ

**Decision:**  
Use Redis Streams.

**Rationale:**
- **Self-hosted requirement**: No cloud dependencies in v1
- **Persistence**: Redis AOF ensures no message loss
- **Ordering**: Streams guarantee FIFO within consumer group
- **Simplicity**: Single Redis instance (also used for caching/locks)
- **Performance**: 10k+ msgs/sec easily (exceeds current needs)

**Consequences:**
- ✅ Simple deployment (one Docker container)
- ✅ Low latency (<1ms enqueue/dequeue)
- ✅ Native Node.js client (`ioredis`)
- ⚠️ Single point of failure (mitigated by daily backups)
- ⚠️ No built-in dead-letter queue (implement manually)

---

### ADR-003: Hybrid Memory (Qdrant + Neo4j) Not Single Database

**Status:** Accepted  
**Date:** 2026-02-20

**Context:**  
Need memory system for both semantic search ("what was said") and relational reasoning ("why we decided"). Options:
1. Qdrant only (vector search)
2. Neo4j only (knowledge graph)
3. Hybrid: Qdrant + Neo4j

**Decision:**  
Use hybrid architecture: Qdrant for semantic search, Neo4j for knowledge graph.

**Rationale:**
- **Different strengths**: Qdrant excels at similarity search; Neo4j excels at graph traversal
- **Complementary queries**: "Find similar discussions" (Qdrant) vs. "Why was X decided?" (Neo4j)
- **Temporal tracking**: Neo4j relationships preserve decision timeline
- **Proven pattern**: Used by Graphiti, Zep, and other agent memory systems

**Consequences:**
- ✅ Best-of-breed tools for each use case
- ✅ Flexible querying (vector + graph)
- ⚠️ Operational complexity (2 databases to maintain)
- ⚠️ Data consistency (require indexing pipeline)

**Risks & Mitigations:**
- **Risk**: Qdrant and Neo4j data out of sync
- **Mitigation**: Atomic indexing transactions; daily consistency checks

---

### ADR-004: Next.js 14 for Dashboard (Not React SPA)

**Status:** Accepted  
**Date:** 2026-02-20

**Context:**  
Web dashboard needs real-time updates and server-side data fetching. Options:
1. React SPA (Create React App)
2. Next.js 14 (App Router)

**Decision:**  
Use Next.js 14 with App Router.

**Rationale:**
- **Server Components**: Fetch Linear/GitHub data server-side (no client API keys)
- **SSE integration**: Built-in streaming support
- **SEO-friendly**: SSR for initial page load
- **Developer experience**: Fast Refresh, TypeScript support

**Consequences:**
- ✅ Faster initial page load
- ✅ Secure API key handling (server-side only)
- ✅ Built-in routing and layouts
- ⚠️ Learning curve for App Router (newer API)

---

### Risk Register

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| **OpenClaw gateway downtime** | Low | High | PM2 auto-restart; health checks; alerting |
| **Linear/GitHub API rate limits** | Medium | Medium | Retry with exponential backoff; cache responses |
| **Redis data loss** | Low | High | Daily backups; AOF persistence; Redis Sentinel (future) |
| **Qdrant index corruption** | Low | Medium | Daily snapshots; rebuild from source files |
| **Neo4j query performance degradation** | Medium | Low | Index optimization; query result caching |
| **Webhook delivery failures** | Medium | High | Idempotency keys; manual replay endpoint |
| **Subagent spawn failures** | Low | High | Retry logic; fallback to manual execution |
| **Memory indexing lag** | Medium | Low | Monitoring alerts; backfill scripts |

---

## Conclusion

This architecture document defines a comprehensive technical blueprint for CruzBot 2.0 Evolution—a hybrid orchestration layer built on top of OpenClaw that transforms the current reactive agent into an autonomous development partner. By leveraging OpenClaw's proven runtime, event-driven integrations, dual-mode memory (vector + graph), and workflow-optimized interfaces, CruzBot 2.0 will achieve:

- **10x faster feature delivery** (days → hours)
- **95% automation** (eliminate manual coordination)
- **Deep contextual understanding** (answer "why" questions)
- **Full workflow visibility** (single-pane-of-glass dashboard)

The architecture is designed for **incremental deployment**, with parallel operation during migration and clear rollback paths. All critical systems include comprehensive monitoring, alerting, and backup strategies to ensure operational reliability.

**Next Steps:**
1. Epic 1 implementation (Orchestration Layer foundation)
2. Local testing with single product
3. Integration testing (E2E BMAD workflow)
4. Production deployment (incremental product migration)

---

**Document Control:**
- **Version:** 1.0
- **Last Updated:** February 20, 2026
- **Next Review:** After Epic 1 completion
- **Status:** **APPROVED FOR IMPLEMENTATION**
