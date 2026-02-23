---
title: "API Design Exploration: REST vs GraphQL"
type: exploratory
category: discovery
status: in-progress
tags:
  - api
  - technical
related_docs:
  - narratives/architecture-overview.md
---

# API Design Exploration: REST vs GraphQL

## Objective

Determine the optimal API paradigm for Nexus's public and internal APIs. This exploration is ongoing and will inform the API layer design for R2 and beyond.

## Context

Nexus currently uses a RESTful API for R1 services (auth, file operations, workspace management). As we add collaboration features (R2) and AI features (R3), the API surface grows significantly. We need to decide whether to continue with REST, migrate to GraphQL, or use a hybrid approach.

## Current State (REST)

The R1 API has 47 endpoints across 6 services:

```
Auth Service:     12 endpoints (login, token, roles, audit)
File Service:     15 endpoints (CRUD, Git operations, search)
Workspace:         8 endpoints (create, settings, members)
User Service:      7 endpoints (profile, preferences, teams)
Notification:      3 endpoints (list, mark-read, preferences)
Analytics:         2 endpoints (basic usage stats)
```

### REST Pain Points Emerging

1. **Over-fetching:** Dashboard views need data from 4+ endpoints; clients make multiple round trips
2. **Under-fetching:** Workspace detail page needs workspace + members + recent files + activity — 4 separate requests
3. **Versioning complexity:** API versioning via URL path (`/v1/`, `/v2/`) creates maintenance burden
4. **Documentation drift:** OpenAPI specs lag behind implementation

## GraphQL Evaluation

### Advantages for Nexus

- **Single request for complex views:** Dashboard can fetch workspace + members + files + activity in one query
- **Type safety:** Schema-first development with automatic TypeScript type generation
- **Subscriptions:** Native subscription model could complement WebSocket sync for UI updates
- **Self-documenting:** Schema IS the documentation; tools like GraphiQL provide exploration

### Example: Dashboard Query

```graphql
query DashboardView($workspaceId: ID!) {
  workspace(id: $workspaceId) {
    name
    members {
      id
      name
      role
      lastActive
    }
    recentFiles(limit: 10) {
      path
      lastModifiedBy { name }
      lastModifiedAt
    }
    activity(limit: 20) {
      type
      actor { name, avatar }
      target
      timestamp
    }
    stats {
      totalFiles
      activeMembersToday
      commitsThisWeek
    }
  }
}
```

This replaces 4 REST calls with a single query.

### Concerns

| Concern | Assessment |
|---------|-----------|
| **Learning curve** | Medium — team has REST experience, GraphQL requires schema design skills |
| **Caching** | Different from REST; requires Apollo Cache or similar client-side cache |
| **File uploads** | GraphQL doesn't handle file uploads natively; need multipart extension |
| **Performance** | N+1 query risk in resolvers; need DataLoader pattern |
| **Authorization** | Per-field authorization more complex than per-endpoint |
| **Rate limiting** | Query complexity analysis needed (vs. simple per-endpoint limits) |

## Hybrid Approach (Current Recommendation)

Based on evaluation so far, we're leaning toward a **hybrid approach:**

1. **GraphQL for read-heavy, composite queries** — Dashboard, search, workspace views
2. **REST for mutations with side effects** — File uploads, Git operations, auth flows
3. **WebSocket for real-time** — Sync engine (already decided)

This plays to each protocol's strengths:
- GraphQL excels at flexible data fetching
- REST excels at predictable, cacheable mutations
- WebSocket excels at bidirectional real-time

## Open Questions

1. Should we use **Apollo Federation** for a unified graph across microservices, or a single gateway that aggregates?
2. How do we handle **authorization at the field level** in GraphQL without duplicating RBAC logic?
3. What's the performance impact of GraphQL query parsing on the hot path?
4. Should the **public API** (for future plugin marketplace) be GraphQL, REST, or both?

## Next Steps

- [ ] Build prototype GraphQL gateway with 3 core types (Workspace, User, File)
- [ ] Benchmark against current REST endpoints for the dashboard use case
- [ ] Evaluate Apollo Federation vs. schema stitching
- [ ] Security review of GraphQL-specific attack vectors (query depth, batching)

## Related Documents

- [Architecture Overview](../narratives/architecture-overview.md)
- [Plugin Marketplace](../backlog/plugin-marketplace.md)
