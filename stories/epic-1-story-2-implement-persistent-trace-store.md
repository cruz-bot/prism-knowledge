---
epic: 1
story: 2
title: Implement Persistent Trace Store
size: M
---

### User Story

As a developer, I want agent traces to be persisted to a durable database, so that trace history is not volatile and survives server restarts.

### Acceptance Criteria

- **Given** an agent has completed a run,
- **When** the server is restarted,
- **Then** the full trace of the agent's run is still available for review in the persistent database.

### Technical Approach

1.  **Prisma Schema:**
    -   Add the `AgentTrace` model to the `prisma/schema.prisma` file as defined in the architecture document. Include all fields (`id`, `userId`, `tenantId`, `agentType`, `input`, `output`, `status`, `startedAt`, `completedAt`, `duration`, `error`, `metadata`) and relations to `User` and `Tenant`.
    -   Add the specified `@@index` annotations for performance.
    -   Run `npx prisma migrate dev --name add_agent_traces` to apply the changes to the database.
    -   Run `npx prisma generate` to update the Prisma client.

2.  **Trace Service:**
    -   Create a new service file at `src/lib/services/trace-service.ts`.
    -   Implement the following async functions:
        -   `createAgentTrace`: Creates a new record in the `AgentTrace` table with `status: 'running'`.
        -   `completeAgentTrace`: Updates an existing trace record with the final status (`success` or `error`), output, error message, and duration.
        -   `getAgentTracesForUser`: Retrieves a list of traces for a given user.

3.  **Base Agent Integration:**
    -   Modify the `execute` method in the core agent file `agents/core/agent-base.ts`.
    -   Remove the existing in-memory `Map` used for storing traces.
    -   At the start of the `execute` method, call `createAgentTrace`.
    -   Wrap the `executeInternal` call in a `try...catch...finally` block.
    -   On success (in `try`), call `completeAgentTrace` with the successful output and `status: 'success'`.
    -   On failure (in `catch`), call `completeAgentTrace` with the error details and `status: 'error'`.

### Testing Strategy

-   **Unit Tests:**
    -   For `trace-service.ts`, write unit tests using a mocked Prisma client to verify that `create`, `update`, and `findMany` are called with the correct parameters.
-   **Integration Test:**
    -   Write an integration test that runs a mock agent that successfully completes. Verify that a corresponding record exists in the `AgentTrace` database table with `status: 'success'`.
    -   Write another integration test for a mock agent that throws an error. Verify that the `AgentTrace` record exists with `status: 'error'` and the correct error message.
-   **Manual Test:**
    -   Run the application locally.
    -   Trigger an agent to run.
    -   Use a database GUI or `prisma studio` to confirm the trace record was created.
    -   Stop the Next.js server (`Ctrl+C`).
    -   Restart the server (`npm run dev`).
    -   Check the database again to ensure the record is still present.

### Definition of Done

-   [ ] `AgentTrace` model is successfully migrated into the database schema.
-   [ ] `trace-service.ts` is implemented and unit tested.
-   [ ] `agents/core/agent-base.ts` is refactored to use the new service.
-   [ ] The old in-memory trace storage mechanism is completely removed.
-   [ ] Integration tests are passing, confirming traces are persisted correctly for both success and error cases.
-   [ ] Manual testing confirms traces survive a server restart.
-   [ ] Code is reviewed and merged.
