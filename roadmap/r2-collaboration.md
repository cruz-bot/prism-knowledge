---
title: "Release 2: Collaboration — Real-Time Teamwork"
type: canonical
category: roadmap
release: R2
status: in-progress
related_items:
  - real-time-sync
  - smart-notifications
---

# Release 2: Collaboration — Real-Time Teamwork

## Overview

Release 2 transforms Nexus from a powerful individual editor into a real-time collaboration platform. This release introduces multiplayer editing, presence awareness, and context-aware notifications — the features that differentiate Nexus from traditional IDEs.

R2 development began in January 2025 and is targeting a Q2 2025 GA release. The real-time sync engine is currently in closed beta with 20 teams.

## Goals

1. **Multiplayer editing** — Multiple users editing the same file simultaneously with conflict-free resolution.
2. **Presence & awareness** — See who is online, what files they're viewing, and where their cursors are.
3. **Smart notifications** — Context-aware alerts that reduce noise and surface relevant activity.
4. **Performance at scale** — Support 50 concurrent editors per workspace with < 100ms sync latency.

## Key Capabilities

### Real-Time Sync Engine

The sync engine is built on CRDTs (Conflict-free Replicated Data Types) using the Yjs library, communicated over a custom WebSocket protocol. This approach was chosen after a thorough [technical spike comparing WebSocket and SSE approaches](../discovery/technical-spike-websockets.md).

**Architecture highlights:**

- **CRDT-based conflict resolution** — No operational transform complexity; convergence is guaranteed mathematically.
- **Awareness protocol** — Cursor positions, selections, and user presence broadcast via lightweight awareness channels.
- **Offline support** — Local changes buffered and synced on reconnect; CRDT merge handles divergence.
- **Room-based scaling** — Each workspace file is a "room" with independent sync state; rooms are load-balanced across sync nodes.

```
Client A ──WebSocket──► Sync Server ◄──WebSocket── Client B
                           │
                      ┌────┴────┐
                      │  Yjs    │
                      │  CRDT   │
                      │  Store  │
                      └─────────┘
```

### Smart Notifications

Notifications in Nexus are context-aware, meaning they consider what the user is currently working on before deciding whether and how to alert them:

- **Focus mode** — Notifications batched and delivered after a focus session ends
- **Relevance scoring** — Changes to files you've recently edited score higher
- **Channel preferences** — In-app, email digest, Slack integration, or webhook
- **@mentions** — Direct mentions always break through notification filters

## Current Status

| Milestone | Target Date | Status |
|-----------|-------------|--------|
| Sync engine alpha | 2025-01-15 | ✅ Complete |
| Beta with 20 teams | 2025-02-15 | ✅ Complete |
| Notification system MVP | 2025-03-01 | 🔄 In progress |
| Performance optimization | 2025-04-01 | 📋 Planned |
| GA release | 2025-06-01 | 📋 Planned |

## Risks

1. **WebSocket connection limits** — Enterprise proxies and firewalls may interfere with persistent WebSocket connections. Fallback to long-polling is implemented but degrades experience.
2. **CRDT memory overhead** — Large documents (10k+ lines) show increased memory usage on the client. Investigation ongoing.
3. **Notification fatigue** — If the relevance scoring model isn't tuned well, users may disable notifications entirely, defeating the purpose.

## Related Documents

- [Real-Time Sync backlog item](../backlog/real-time-sync.md)
- [Smart Notifications backlog item](../backlog/smart-notifications.md)
- [WebSocket Technical Spike](../discovery/technical-spike-websockets.md)
- [Product Vision](../narratives/product-vision.md)
