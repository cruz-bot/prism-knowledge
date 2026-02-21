---
epic: 1
story: 3
title: Migrate Legacy Persistence Routes
size: M
---

### User Story

As a developer, I want the 7 legacy API routes to be migrated to the current persistence layer, so that we eliminate outdated code paths and ensure data consistency.

### Acceptance Criteria

- **Given** the list of 7 legacy API operations,
- **When** each operation is individually tested via its API endpoint,
- **Then** all data operations correctly use the new Prisma-based persistence layer.
- **And** the old persistence layer code (`legacy-persistence.ts`) is deleted.

### Technical Approach

1.  **Route Identification:**
    -   Identify the 7 route handlers located within the `app/api/legacy/` directory structure.
    -   Primary routes to address:
        -   `GET /api/legacy/sources`
        -   `POST /api/legacy/sources`
        -   `GET /api/legacy/sources/[id]`
        -   `PATCH /api/legacy/sources/[id]`
        -   `DELETE /api/legacy/sources/[id]`
        -   `GET /api/legacy/queries`
        -   `POST /api/legacy/queries`

2.  **Migration Process (for each route):**
    -   Relocate the route handler file from the `app/api/legacy/` directory to its canonical location (e.g., `app/api/sources/route.ts`).
    -   Replace any calls to the `legacyPersistence` module with calls to the modern, Prisma-backed services (e.g., `source-service.ts`, `query-service.ts`).
    -   Ensure that `requireUserAsync` is called at the start of each handler to enforce authentication.
    -   Verify that all data access is scoped to the authenticated user (`userId` and `tenantId`).

3.  **Code Decommissioning:**
    -   After all routes are migrated and tests are passing, delete the `src/lib/legacy-persistence.ts` file.
    -   Delete the now-empty `app/api/legacy/` directory.

### Testing Strategy

-   **Integration/Regression Testing:**
    -   For each of the 7 migrated API operations, write a dedicated integration test.
    -   Each test should:
        1.  Set up the necessary authenticated user session.
        2.  Call the API endpoint.
        3.  Directly query the PostgreSQL database using the Prisma client to assert that the expected change occurred (e.g., a record was created, updated, or deleted).
        4.  Assert that the API response is correct.
-   **Authentication Testing:**
    -   For each endpoint, add a test case to ensure that an unauthenticated request is rejected with a `401 Unauthorized` status code.

### Definition of Done

-   [ ] All 7 identified legacy API operations are refactored to use the Prisma-based service layer.
-   [ ] All 7 endpoints enforce authentication using `requireUserAsync`.
-   [ ] The `src/lib/legacy-persistence.ts` file has been deleted.
-   [ ] The `app/api/legacy/` directory has been deleted.
-   [ ] A comprehensive suite of integration tests is implemented for the migrated endpoints, and all tests are passing.
-   [ ] The application functions correctly with the legacy routes removed.
-   [ ] Code is reviewed and merged.
