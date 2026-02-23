---
title: "Real-Time Sync Engine"
type: canonical
category: backlog
strategic_role: Core
priority: high
release: R2
journey_phases:
  - 3
  - 4
---

# Real-Time Sync Engine

## Summary

The real-time sync engine enables multiplayer editing in Nexus — multiple developers editing the same file simultaneously with instant conflict resolution. This is the core technology that transforms Nexus from a solo editor into a collaboration platform.

## Problem Statement

Code review and pair programming are essential practices, but current tools make them awkward:

- **Screen sharing** is passive — only one person can type
- **Live Share (VS Code)** requires both parties to use VS Code and has latency issues
- **Branch-based collaboration** introduces merge conflicts and delayed feedback

Real-time sync eliminates these friction points by making simultaneous editing the default mode.

## Requirements

### Must Have (R2)

- Simultaneous multi-user editing with < 100ms sync latency
- Cursor and selection presence for all active users
- User avatars and color-coded cursors
- Automatic conflict resolution (no manual merging)
- Support for 50 concurrent editors per file
- Graceful degradation on poor network connections
- Offline edit buffering with automatic sync on reconnect

### Should Have (R2)

- Follow mode (lock your viewport to another user's cursor)
- Awareness sidebar showing who's editing which files
- Voice chat integration (WebRTC) for pair programming sessions
- Undo/redo that respects multi-user context

### Could Have (Future)

- Replay mode (watch editing session playback)
- Branching within shared sessions (parallel exploration)

## Technical Design

### CRDT Implementation

We use **Yjs** as our CRDT library, chosen after evaluating Automerge, Diamond Types, and a custom implementation. See the [WebSocket technical spike](../discovery/technical-spike-websockets.md) for the full evaluation.

**Why Yjs:**
- Battle-tested in production (used by Notion, Jupyter)
- Excellent performance characteristics (sub-millisecond local operations)
- Rich awareness protocol for cursor/selection sharing
- Active maintenance and community

### Sync Protocol

```
Client                    Sync Server                  Client
  │                           │                           │
  │──── sync-step1 ──────────►│                           │
  │                           │◄──── sync-step1 ──────────│
  │◄──── sync-step2 ──────────│                           │
  │                           │──── sync-step2 ───────────►│
  │                           │                           │
  │──── update ──────────────►│──── update ───────────────►│
  │◄──── update ──────────────│◄──── update ──────────────│
  │                           │                           │
  │──── awareness ───────────►│──── awareness ────────────►│
```

### Scaling Strategy

- **Room-based sharding:** Each file is a "room" assigned to a sync node
- **Consistent hashing:** Room-to-node mapping via hash ring for predictable routing
- **Hot standby:** Each room replicated to a secondary node for failover
- **Connection limits:** 10,000 WebSocket connections per sync node (tested)

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| CRDT memory growth on large docs | High | Garbage collection protocol, doc size limits |
| WebSocket blocked by enterprise proxy | Medium | Long-polling fallback, CONNECT tunnel |
| Network partitions cause stale state | Medium | Heartbeat + reconnection protocol |
| Yjs library maintenance risk | Low | Abstraction layer allows CRDT swap |

## Success Metrics

- **Sync latency:** P95 < 100ms for updates between users
- **Adoption:** 40% of active users try multiplayer editing within first month
- **Reliability:** 99.95% sync uptime (separate from editor uptime)
- **Scale:** Support 500 concurrent rooms per sync node

## Related Documents

- [R2 Collaboration Release](../roadmap/r2-collaboration.md)
- [WebSocket Technical Spike](../discovery/technical-spike-websockets.md)
- [User Journey — Stages 3 & 4](../narratives/user-journey.md)
