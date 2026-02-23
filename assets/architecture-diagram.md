---
title: "System Architecture Diagram"
type: canonical
category: asset
diagram_type: system-map
related_docs:
  - narratives/architecture-overview.md
---

# System Architecture Diagram

## Overview

This document describes the Nexus system architecture at the component level. The diagram below illustrates the three planes (editing, collaboration, intelligence) and their interactions.

## Architecture Planes

### Client Plane

The client application is a React SPA (Single Page Application) with three primary modules:

| Module | Technology | Purpose |
|--------|-----------|---------|
| Editor | Monaco Editor (React wrapper) | Code editing, syntax highlighting, IntelliSense |
| Collaboration | Yjs client + awareness protocol | Real-time sync, cursor presence |
| Analytics | React + D3.js | Dashboard visualizations |

All client-to-server communication flows through two channels:
1. **HTTPS** — REST/GraphQL API calls for CRUD operations
2. **WebSocket (wss://)** — Real-time sync and presence updates

### Platform Plane

The platform plane consists of six microservices, connected via an Istio service mesh with mTLS:

```
                    ┌─────────────────────┐
                    │   API Gateway       │
                    │   (Kong)            │
                    └──────┬──────────────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
    ┌─────┴─────┐   ┌─────┴─────┐   ┌─────┴──────┐
    │ Auth      │   │ File      │   │ Analytics  │
    │ Service   │   │ Service   │   │ Service    │
    │ (Node.js) │   │ (Go)      │   │ (Python)   │
    └───────────┘   └───────────┘   └────────────┘

    ┌───────────┐   ┌───────────┐   ┌────────────┐
    │ Sync      │   │ User      │   │ AI Pipeline│
    │ Service   │   │ Service   │   │ Service    │
    │ (Node.js) │   │ (Node.js) │   │ (Python)   │
    └───────────┘   └───────────┘   └────────────┘
```

### Data Plane

| Component | Technology | Data Stored |
|-----------|-----------|-------------|
| Primary Database | PostgreSQL 16 | Users, teams, workspaces, permissions, audit logs |
| Cache Layer | Redis Cluster (6 nodes) | Sessions, hot files, CRDT state, pub/sub |
| Object Storage | MinIO (S3-compatible) | File content, Git objects, backups |
| Vector Database | pgvector (PostgreSQL ext.) | Code embeddings for AI features |
| Search Engine | Elasticsearch 8 | Full-text code search index |
| Message Queue | NATS JetStream | Event streaming, async job processing |

## Data Flow: Edit Operation

When a developer types in the editor, the following flow occurs:

1. **Keystroke** → Monaco editor processes locally (< 1ms)
2. **Local CRDT update** → Yjs applies change to local document state
3. **WebSocket broadcast** → Update sent to Sync Service
4. **Server-side merge** → Sync Service applies update to authoritative CRDT state
5. **Fan-out** → Update broadcast to all other connected clients in the room
6. **Persistence** → Debounced write to File Service (every 5 seconds or on explicit save)
7. **Search index** → Async update to Elasticsearch via NATS event

Total latency for steps 1-5 (visible to collaborators): **12ms P50, 28ms P95**.

## Infrastructure

### Kubernetes Cluster (EKS)

- **Region:** US-East-1 (primary), EU-West-1 (secondary)
- **Node pools:** 
  - `general`: 8x m6i.xlarge (API, Auth, User services)
  - `sync`: 4x c6i.2xlarge (Sync Service — CPU-optimized for WebSocket)
  - `ai`: 2x g5.xlarge (AI Pipeline — GPU instances, R3)
- **Autoscaling:** HPA based on CPU/memory + custom WebSocket connection count metric

### CDN & Edge

- **CloudFront** for static assets (editor bundle ~5MB, split across 12 chunks)
- **Edge functions** for geographic routing (user → nearest data plane)
- **Cache hit ratio:** 94% for static assets

### Observability

- **Metrics:** Datadog (custom dashboards per service)
- **Traces:** Datadog APM with distributed tracing across all services
- **Logs:** Structured JSON → Datadog Log Management
- **Alerts:** PagerDuty with escalation policies per service team
- **SLOs:** 99.9% availability, < 200ms P95 API latency, < 100ms P95 sync latency

## Security Architecture

Security follows a zero-trust model:

1. **Edge:** WAF (AWS WAF) + DDoS protection (CloudFront Shield)
2. **Network:** Service mesh mTLS, no service-to-service traffic in plaintext
3. **Application:** JWT validation at gateway, OPA policy enforcement per service
4. **Data:** AES-256-GCM encryption at rest, TLS 1.3 in transit
5. **Secrets:** HashiCorp Vault for all credentials and API keys
6. **Compliance:** SOC 2 Type II (in progress), GDPR data residency via region selection

## Related Documents

- [Architecture Overview (Narrative)](../narratives/architecture-overview.md)
- [WebSocket Technical Spike](../discovery/technical-spike-websockets.md)
- [Auth & Permissions](../backlog/auth-and-permissions.md)
