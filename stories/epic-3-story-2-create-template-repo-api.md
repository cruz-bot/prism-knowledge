---
epic: 3
story: 2
title: Create Template-Based Repository API Endpoint
size: L
---

### User Story

As a developer, I want a `POST /api/sources/create-from-template` endpoint, so that the frontend can trigger the creation of a new workspace from a template.

### Acceptance Criteria

- **Given** a user is authenticated and has a valid GitHub token,
- **When** a POST request is sent to `/api/sources/create-from-template` with a valid `templateId`,
- **Then** the API creates a new **private** repository on that user's GitHub account.
- **And** the API creates a single initial commit in the new repository containing all files from the selected template.
- **And** a new `Source` record is created in the Prism database linking the user to the new repository.
- **And** the API returns a `201 Created` status with the new source's ID and URL.
- **And** the P95 latency for the entire API operation is under 5 seconds.

### Technical Approach

1.  **API Route Creation:**
    -   Create the Next.js API route handler at `app/api/sources/create-from-template/route.ts`.

2.  **Request Handling Flow:**
    -   **Authentication:** Begin the `POST` handler by calling `requireUserAsync(request)` to ensure the user is authenticated and to retrieve their `userId`, `tenantId`, and `githubAccessToken`.
    -   **Validation:** Parse the request body using a Zod schema to validate the presence and format of the `templateId`.
    -   **Service Initialization:** Instantiate the `GitHubService` with the user's `githubAccessToken`.

3.  **Core Logic (in a `try...catch` block):**
    -   **Parallel Fetch:** Use `Promise.all` to concurrently fetch the GitHub user's details (`githubService.getAuthenticatedUser()`) and the template files (`TemplateService.getTemplateFiles(templateId)`).
    -   **Repository Naming:** Generate a unique repository name using `TemplateService.generateRepoName(templateId)`.
    -   **Repository Creation:** Call `githubService.createRepository()`, passing the generated name and ensuring the `isPrivate` flag is set to `true`.
    -   **Initial Commit:** Call `githubService.createInitialCommit()` to push all template files to the new repository in a single, efficient commit using the Git Blob/Tree API strategy. This is critical for meeting the performance requirement.
    -   **Database Persistence:** Call the `createSourceFromTemplate()` function in the `source-service` to save the new repository's details (`repoFullName`, `repoUrl`, `githubRepoId`, etc.) as a `Source` record in the database.

4.  **Response Handling:**
    -   **On Success:** Return a `NextResponse.json()` with a `{ source: { id, repoUrl, ... } }` payload and a `{ status: 201 }`.
    -   **On Error:** In the `catch` block, log the detailed error and return a `NextResponse.json()` with a user-friendly error message and an appropriate status code (e.g., `500`).

5.  **Service Enhancement:**
    -   Add the `createSourceFromTemplate` function to `src/lib/services/source-service.ts` as specified in the architecture document.

### Testing Strategy

-   **Integration Testing (Primary):**
    -   This endpoint is best tested at the integration level due to its orchestration nature.
    -   **Setup:** Mock the Next.js `fetch` or API client, a valid user session, and the various services:
        -   Mock `TemplateService` to return a small, consistent set of virtual files.
        -   Mock all methods of `GitHubService` (`getAuthenticatedUser`, `createRepository`, `createInitialCommit`) to return successful dummy data without making real API calls.
        -   Mock the `source-service` to spy on the `createSourceFromTemplate` function.
    -   **Success Case:**
        -   Send a request to the endpoint.
        -   Assert that `createRepository` was called with `private: true`.
        -   Assert that `createInitialCommit` was called.
        -   Assert that `createSourceFromTemplate` was called with the correct repository details.
        -   Assert that the final API response is `201` and contains the expected data.
    -   **Failure Case:**
        -   Configure the `GitHubService` mock to throw an error on `createRepository`.
        -   Send a request and assert that the API response is `500` (or another appropriate error code) and contains a user-friendly error message.

### Definition of Done

-   [ ] The `POST /api/sources/create-from-template` endpoint is implemented.
-   [ ] The endpoint enforces authentication and validates incoming data.
-   [ ] The endpoint successfully orchestrates calls to the `TemplateService`, `GitHubService`, and `source-service`.
-   [ ] Repositories are created as **private**.
-   [ ] The efficient single-commit strategy (`createInitialCommit`) is used.
-   [ ] A `Source` record is correctly created in the database.
-   [ ] Comprehensive integration tests covering success and failure scenarios are written and passing.
-   [ ] Performance is considered and optimized to meet the <5s P95 target.
-   [ ] Code is reviewed and merged.
