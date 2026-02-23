---
title: "Technical Spike: WebSocket vs SSE for Real-Time Sync"
type: exploratory
category: discovery
status: complete
tags:
  - technical
  - spike
related_items:
  - real-time-sync
---

# Technical Spike: WebSocket vs SSE for Real-Time Sync

## Objective

Evaluate WebSocket and Server-Sent Events (SSE) as transport protocols for Nexus's real-time collaboration engine. Determine which protocol best supports bidirectional sync, presence awareness, and enterprise network constraints.

## Date

November 2024

## Participants

- Lead: Marcus Chen (Platform Architect)
- Review: Sarah Kim (Infrastructure), David Okafor (Frontend)

## Options Evaluated

### Option A: WebSocket (Custom Binary Protocol)

Full-duplex communication over a single TCP connection. Client and server can send messages independently at any time.

**Prototype implementation:**

```javascript
// Server-side WebSocket handler
wss.on('connection', (ws, req) => {
  const roomId = parseRoom(req.url);
  const doc = getOrCreateYDoc(roomId);
  
  // Sync protocol
  ws.on('message', (data) => {
    const msg = decodeMessage(data);
    switch (msg.type) {
      case MSG_SYNC_STEP1:
        handleSyncStep1(ws, doc, msg);
        break;
      case MSG_SYNC_STEP2:
        handleSyncStep2(ws, doc, msg);
        break;
      case MSG_UPDATE:
        applyUpdate(doc, msg.update);
        broadcastToRoom(roomId, msg, ws);
        break;
      case MSG_AWARENESS:
        broadcastAwareness(roomId, msg, ws);
        break;
    }
  });
});
```

### Option B: Server-Sent Events + POST

Unidirectional server-to-client stream via SSE, with client-to-server updates sent as HTTP POST requests.

**Prototype implementation:**

```javascript
// SSE endpoint
app.get('/sync/:roomId/events', (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
  });
  
  const roomId = req.params.roomId;
  subscribeToRoom(roomId, (event) => {
    res.write(`data: ${JSON.stringify(event)}\n\n`);
  });
});

// POST endpoint for client updates
app.post('/sync/:roomId/update', (req, res) => {
  applyUpdate(req.params.roomId, req.body);
  res.sendStatus(200);
});
```

## Evaluation Results

### Performance Benchmarks

| Metric | WebSocket | SSE + POST |
|--------|-----------|------------|
| Update latency (P50) | 12ms | 45ms |
| Update latency (P95) | 28ms | 120ms |
| Connection overhead | 1 TCP conn | 1 SSE + N HTTP |
| Bandwidth (1 min active editing) | 4.2 KB | 12.8 KB |
| Concurrent connections (per node) | 10,000 | 15,000 |
| Memory per connection | 2.4 KB | 1.8 KB |

### Enterprise Network Compatibility

| Scenario | WebSocket | SSE + POST |
|----------|-----------|------------|
| Standard corporate proxy | ✅ | ✅ |
| SSL-intercepting proxy | ✅ (wss://) | ✅ |
| Restrictive firewall (ports 80/443 only) | ✅ | ✅ |
| WebSocket-blocking proxy | ❌ | ✅ |
| HTTP/2 multiplexing | ❌ | ✅ |

**Key insight:** ~5% of enterprise environments block WebSocket upgrades. We need a fallback path.

### Developer Experience

- **WebSocket:** Natural fit for bidirectional sync. Yjs has native WebSocket provider. Single connection simplifies state management.
- **SSE:** Simpler server implementation. Better HTTP/2 compatibility. But bidirectional requires separate POST channel, complicating the sync protocol.

## Decision

**WebSocket as primary transport, with SSE + POST as automatic fallback.**

### Rationale

1. **Latency:** WebSocket's 12ms P50 vs SSE's 45ms P50 is significant for real-time editing feel
2. **Yjs compatibility:** Native WebSocket provider in Yjs; SSE would require custom adapter
3. **Bidirectional simplicity:** Single connection for both directions reduces complexity
4. **Fallback coverage:** SSE fallback handles the ~5% of enterprise environments that block WebSocket

### Implementation Plan

1. Primary: `y-websocket` provider with custom binary protocol
2. Fallback: SSE + POST provider activated on WebSocket connection failure
3. Detection: Client attempts WebSocket first, falls back to SSE after 5-second timeout
4. Monitoring: Track fallback activation rate to understand enterprise proxy landscape

## Risks Identified

1. **CRDT memory growth** — Both protocols affected; need garbage collection strategy regardless
2. **Connection counting** — WebSocket connections count against server file descriptor limits; plan for ulimit tuning
3. **Load balancer affinity** — WebSocket requires sticky sessions or room-aware routing; SSE is stateless-friendly

## Related Documents

- [Real-Time Sync backlog item](../backlog/real-time-sync.md)
- [R2 Collaboration Release](../roadmap/r2-collaboration.md)
- [Architecture Overview](../narratives/architecture-overview.md)
