---
title: "Nexus Architecture Overview"
type: canonical
category: narrative
tags:
  - architecture
  - technical
related_docs:
  - discovery/technical-spike-websockets.md
---

# Nexus Architecture Overview

## System Design Philosophy

Nexus follows a **microservices architecture** with a clear separation between the editing plane, the collaboration plane, and the intelligence plane. Each plane can scale independently and evolve at its own pace.

## High-Level Architecture

```
┌─────────────────────────────────────────────────┐
│                   Client Layer                   │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │  Editor   │  │ Collab   │  │  Analytics    │  │
│  │  (Monaco) │  │ (Yjs)    │  │  (React)      │  │
│  └────┬─────┘  └────┬─────┘  └──────┬────────┘  │
└───────┼──────────────┼───────────────┼───────────┘
        │ HTTPS        │ WebSocket     │ HTTPS
┌───────┼──────────────┼───────────────┼───────────┐
│       ▼              ▼               ▼           │
│  ┌─────────┐  ┌──────────┐  ┌──────────────┐    │
│  │ API      │  │ Sync     │  │ Analytics    │    │
│  │ Gateway  │  │ Service  │  │ Service      │    │
│  └────┬─────┘  └────┬─────┘  └──────┬───────┘   │
│       │              │               │           │
│  ┌────┴──────────────┴───────────────┴────┐      │
│  │           Service Mesh (Istio)          │      │
│  └────┬──────────────┬───────────────┬────┘      │
│       ▼              ▼               ▼           │
│  ┌─────────┐  ┌──────────┐  ┌──────────────┐    │
│  │ Auth     │  │ File     │  │ AI Pipeline  │    │
│  │ Service  │  │ Service  │  │ Service      │    │
│  └─────────┘  └──────────┘  └──────────────┘    │
│                  Platform Layer                   │
└──────────────────────────────────────────────────┘
```

## Service Breakdown

### API Gateway

The API Gateway handles all HTTP traffic, providing:

- **Rate limiting** — Per-user and per-team quotas
- **Authentication** — JWT validation via the Auth Service
- **Routing** — Path-based routing to downstream services
- **Caching** — Edge caching for static assets and read-heavy endpoints

Built on **Kong** with custom Lua plugins for Nexus-specific routing logic.

### Sync Service

The Sync Service manages all real-time collaboration features. This was the subject of a detailed [technical spike on WebSocket vs SSE](../discovery/technical-spike-websockets.md).

- **Protocol:** Custom binary WebSocket protocol over TLS
- **State management:** Yjs CRDT documents stored in Redis for hot state, PostgreSQL for persistence
- **Scaling:** Horizontal via room-based sharding; each sync node handles ~10,000 concurrent connections
- **Failover:** Active-passive with automatic promotion; client reconnects transparently

### Auth Service

Handles identity, authentication, and authorization:

- **Identity providers:** Google, GitHub, Okta (SAML), Azure AD
- **Token format:** JWT with 15-minute access tokens, 7-day refresh tokens
- **RBAC engine:** Policy-based using OPA (Open Policy Agent)
- **Audit logging:** Every auth event written to immutable audit log

### File Service

Manages workspace file storage and Git operations:

- **Storage:** Object storage (S3-compatible) with Git-native operations
- **Versioning:** Full Git history preserved; every save is a commit
- **Caching:** Hot files cached in Redis; LRU eviction based on access patterns
- **Limits:** 100MB per file, 10GB per workspace (configurable for enterprise)

### AI Pipeline Service

Orchestrates AI-powered features (R3):

- **Embedding generation:** Code and document embeddings stored in pgvector
- **RAG pipeline:** Retrieval-augmented generation for code review
- **Model routing:** Supports multiple LLM providers with fallback chains
- **Isolation:** Strict tenant isolation; no cross-tenant data in any pipeline

## Data Architecture

| Store | Technology | Purpose |
|-------|-----------|---------|
| Primary DB | PostgreSQL 16 | Users, teams, workspaces, permissions |
| Cache | Redis Cluster | Session state, hot files, CRDT sync |
| Object Store | MinIO (S3-compat) | File content, Git objects |
| Vector DB | pgvector extension | Code embeddings for AI features |
| Search | Elasticsearch | Full-text code search |
| Queue | NATS JetStream | Event streaming, async processing |

## Infrastructure

Nexus runs on **Kubernetes** (EKS) with:

- **Regions:** US-East, EU-West, APAC (Singapore)
- **CDN:** CloudFront for static assets and editor bundles
- **Observability:** Datadog (metrics, traces, logs) + PagerDuty (alerting)
- **CI/CD:** GitHub Actions → ArgoCD → Kubernetes

## Security Model

Security is layered and follows zero-trust principles:

1. **Network:** Service mesh with mTLS between all services
2. **Application:** RBAC + row-level security in PostgreSQL
3. **Data:** AES-256 encryption at rest, TLS 1.3 in transit
4. **Compliance:** SOC 2 Type II (in progress), GDPR data residency support

## Related Documents

- [API Design Exploration](../discovery/api-design-exploration.md)
- [Architecture Diagram](../assets/architecture-diagram.md)
- [WebSocket Technical Spike](../discovery/technical-spike-websockets.md)
