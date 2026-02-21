---
epic: 1
story: 1
title: Secure Custom Agent API
size: S
---

### User Story

As a platform operator, I want the `/api/agents/custom` endpoint to be protected by authentication and authorization, so that only authenticated users can access it and prevent a SEV-1 vulnerability.

### Acceptance Criteria

- **Given** a user is not authenticated,
- **When** they attempt to POST to `/api/agents/custom`,
- **Then** they receive a `401 Unauthorized` error.
- **And** a valid, authenticated request from an authorized user succeeds.

### Technical Approach

1.  **File to Modify:** `app/api/agents/custom/route.ts`
2.  **Import:** Import the `requireUserAsync` utility from `@/src/lib/auth/requireUser`.
3.  **Implementation:**
    -   At the beginning of the `POST` function, call `await requireUserAsync(request)`.
    -   Check if the result is unsuccessful (`!authResult.success`). If so, return `authResult.response` immediately.
    -   Use the `userId` and `tenantId` from the successful `authResult.user` to scope all subsequent logic and data access, ensuring a user can only interact with their own data.

### Testing Strategy

-   **Unit/Integration Test:**
    -   Create a test that simulates a `POST` request to the endpoint *without* a valid session token. Assert that the response status is `401`.
    -   Create a test that simulates a `POST` request *with* a valid session token. Assert that the response status is `200` or another appropriate success code.
-   **Manual Test:**
    -   Using `curl` or a REST client, send a `POST` request to a local running instance of the application without any authentication cookies. Verify a `401` response.
    -   Log in to the application in a browser, then use the browser's developer tools or a REST client with the session cookie to send a valid `POST` request. Verify a successful response.

### Definition of Done

-   [ ] `requireUserAsync` check is implemented in the `POST` handler of `/api/agents/custom`.
-   [ ] Automated tests are written to cover both unauthenticated and authenticated cases.
-   [ ] All tests are passing.
-   [ ] Manual verification confirms the endpoint is secure.
-   [ ] Code is reviewed and merged to the main branch.
