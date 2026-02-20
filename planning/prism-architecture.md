# Architecture Document: Frictionless Onboarding: GitHub for Everyone

**Author:** BMAD Architect Agent
**Date:** 2026-02-20
**Version:** 1.0
**Status:** DRAFT

## 1. System Overview

This document outlines the technical architecture for the "Frictionless Onboarding: GitHub for Everyone" feature. The primary goal is to provide a seamless, wizard-based experience for non-technical users ("Type B") to create a Prism workspace backed by a repository on their own GitHub account, taking them from sign-up to their first AI-powered answer in under two minutes.

The architecture leverages existing Prism patterns for authentication, data access, and component design, while introducing a new, dedicated onboarding flow and backend service.

### High-Level Architecture Diagram

```ascii
+----------------------+      +-----------------------------+      +------------------------+
|   Prism Frontend     |      |       Prism Backend         |      |    External Services   |
| (Next.js App Router) |      |        (Next.js API)        |      |                        |
+----------------------+      +-----------------------------+      +------------------------+
           |                             |                                  |
           | 1. User signs up/logs in    |                                  |
           |---------------------------->|                                  |
           |                             |                                  |
           | 2. Redirect to /onboarding  |                                  |
           |<----------------------------|                                  |
           |                             |                                  |
           | 3. User chooses "Start      |                                  |
           |    with Template"           |                                  |
           |                             |                                  |
           | 4. /onboarding/setup wizard |                                  |
           |    - Step 1: GitHub Auth    |                                  |
           |      (Initiates Device Flow)|                                  |
           |--------------------------------------------------------------->| [GitHub OAuth]
           |                             |                                  |   (Device Flow)
           |      (User authorizes)      |                                  |
           |<---------------------------------------------------------------|
           |                             | 5. GitHub Token stored securely  |
           |                             |    in `Account` table (NextAuth) |
           |                             |                                  |
           |    - Step 2: Template Select|                                  |
           |                             |                                  |
           |    - Step 3: Create         | 6. POST /api/sources/create-from-template |
           |---------------------------->|   (with templateId)             |
           |                             |                                  |
           |                             | 7. RepoCreatorService           |
           |                             |    - Fetches user's GH token    |
           |                             |    - Reads template from        |
           |                             |      /src/lib/workspace-templates |
           |                             |    - Uses Octokit to:           |
           |                             |      a. Create private repo     |-----> [GitHub API]
           |                             |      b. Commit template files   |-----> [GitHub API]
           |                             |                                  |
           |                             | 8. Creates `Source` record in DB|-----> [Prisma/DB]
           |                             |                                  |
           |                             | 9. Returns { success: true, ...}|
           |<----------------------------|                                  |
           |                             |                                  |
           | 10. Redirect to new         |                                  |
           |    /console/[workspaceId]   |                                  |
           |                             |                                  |
           | 11. User asks first question|                                  |
           |    in `AskPrism` component  |                                  |
           +-----------------------------+----------------------------------+
```

## 2. Component Breakdown

This feature introduces a new onboarding wizard and a backend service for repository creation.

### Frontend Components (`'use client'`)

-   **`OnboardingWelcome.tsx`** (`/onboarding`)
    -   **Responsibility:** Presents the bifurcated choice between the existing "Type A" flow (Connect existing repo) and the new "Type B" flow (Start with a template).
    -   **Interface:** Receives no props. Contains two links pointing to the respective flows.
-   **`OnboardingWizard.tsx`** (`/onboarding/setup`)
    -   **Responsibility:** A multi-step container component that manages the state of the "Type B" onboarding flow. It orchestrates the display of the child step components.
    -   **Interface:** Manages internal state for the current step and user selections.
-   **`StepConnectGitHub.tsx`** (Step 1 of Wizard)
    -   **Responsibility:** Integrates the existing `GitHubAuthFlow.tsx` component. Provides clear UI text guiding users, including a link to `github.com` for those who need to create an account.
    -   **Interface:** Props: `onSuccess: () => void`. Calls `onSuccess` when the GitHub authentication completes.
-   **`StepTemplatePicker.tsx`** (Step 2 of Wizard)
    -   **Responsibility:** Fetches and displays the available workspace templates. Allows the user to select one.
    -   **Interface:** Props: `onSelect: (templateId: string) => void`. State: `selectedTemplate`.
-   **`StepCreateWorkspace.tsx`** (Step 3 of Wizard)
    -   **Responsibility:** Displays the selected template and a "Create Workspace" button. Handles the API call to the backend, displaying loading and error states.
    -   **Interface:** Props: `templateId: string`. `onSuccess: (workspaceId: string) => void`. Makes a `POST` request to `/api/sources/create-from-template`.
-   **`FirstAskEmptyState.tsx`** (`/console/[workspaceId]`)
    -   **Responsibility:** A new variant of the workspace empty state, specifically designed to guide a freshly onboarded user to immediately use the "Ask Prism" feature.
    -   **Interface:** Displayed conditionally when a workspace is newly created.

### Backend Components (Server-Side)

-   **`POST /api/sources/create-from-template/route.ts`**
    -   **Responsibility:** The main API endpoint for this feature. It orchestrates the entire workspace creation process.
    -   **Interface:** Accepts a `POST` request with a JSON body: `{ templateId: string }`.
    -   **Logic:**
        1.  Authenticates the user using `requireUserAsync`.
        2.  Validates input using Zod.
        3.  Calls the `RepoCreatorService` to perform the core logic.
        4.  Returns a `201 Created` or an appropriate error response.
-   **`RepoCreatorService.ts`** (`src/lib/workspace/repo-creator-service.ts`)
    -   **Responsibility:** A new, dedicated service that encapsulates all logic for creating a GitHub repository from a template. This isolates the logic from the route handler and makes it testable.
    -   **Interface:** `createFromTemplate(userId: string, templateId: string): Promise<{ source: Source }>`
    -   **Dependencies:** `prisma`, `Octokit` (GitHub client).
-   **`WorkspaceTemplateProvider.ts`** (`src/lib/workspace/workspace-template-provider.ts`)
    -   **Responsibility:** Provides access to the file-based workspace templates stored in `src/lib/workspace-templates/`. It reads the directory structure and file contents.
    -   **Interface:** `getTemplate(templateId: string): Promise<{ files: { path: string; content: string }[] }>`, `listTemplates(): Promise<{ id: string; name: string; description: string }[]>`

## 3. Data Models

### Database Schema (Prisma)

No schema changes are required. The feature will use the existing `Account` and `Source` models.

-   **`Account`:** The NextAuth.js Prisma adapter already stores the GitHub OAuth `access_token` in this table. This token will be retrieved to authorize GitHub API calls.
-   **`Source`:** A new record will be created in this table to represent the user's new repository, linking it to their `userId`.

### API Contracts (Zod)

-   **Request Body for `POST /api/sources/create-from-template`:**
    ```typescript
    // src/lib/workspace/schemas.ts
    import { z } from 'zod';

    export const CreateFromTemplateRequestSchema = z.object({
      templateId: z.string().regex(/^[a-z0-9-]+$/, 'Invalid template ID format'),
    });
    ```
-   **Success Response (201 Created):**
    ```typescript
    // src/lib/workspace/schemas.ts
    export const CreateFromTemplateResponseSchema = z.object({
      id: z.string().uuid(),
      name: z.string(),
      url: z.string().url(),
    });
    ```

## 4. Integration Points

-   **Authentication:** The new flow seamlessly integrates with the existing NextAuth.js setup. The GitHub provider configuration and the `Account` model are used directly. Authentication is enforced on the new API endpoint using the standard `requireUserAsync` pattern.
-   **Source Management:** The new repository, once created, is registered as a standard `Source` in the database. From that point on, it is treated like any other user-connected repository by the rest of the Prism application (e.g., for indexing, querying, etc.).
-   **UI Flow:** The `OnboardingWelcome.tsx` component acts as the new entry hub, directing users to either the existing `RepoConnector` flow (for "Type A" users) or the new `OnboardingWizard.tsx` (for "Type B" users). This ensures a non-disruptive integration with the current user experience.

## 5. Security Considerations

Security is paramount, especially as this feature handles user authentication tokens and creates resources on their behalf. Epic 1 is a hard prerequisite.

1.  **P0 Security Hardening (Epic 1):** The three critical vulnerabilities identified in the Epics document **must be remediated before this feature is released to production.**
    -   `/api/agents/custom` must be authenticated.
    -   Traces must be moved to a persistent store.
    -   Legacy persistence routes must be migrated.
2.  **Authentication:** The `POST /api/sources/create-from-template` endpoint **must** be protected by `requireUserAsync`. This is non-negotiable and aligns with **Standard #5 (Auth & Authorization Pattern)**.
3.  **Authorization & Scoping:**
    -   The `RepoCreatorService` will retrieve the GitHub OAuth token specifically for the authenticated `userId`.
    -   The resulting `Source` record will be explicitly tied to the `userId`, adhering to the critical **Standard #5 (User Scoping Pattern)**. There is no risk of one user creating a repository for another.
4.  **GitHub Token Handling:** The `access_token` is managed by `NextAuth.js` and stored encrypted in the database. The service will retrieve it only when needed and hold it in memory only for the duration of the API call. It will never be exposed to the client.
5.  **Repository Visibility:** Repositories created on the user's behalf **must default to private**. This is a critical requirement from the PRD to protect user data by default. The `Octokit` call will explicitly set `private: true`.
6.  **Input Validation:** The API endpoint will use the `CreateFromTemplateRequestSchema` (Zod) to validate the incoming `templateId`, preventing any form of injection or unexpected input. This follows **Standard #2 (Type Patterns)**.

## 6. Technology Stack

The technology stack for this feature is consistent with the existing Prism codebase as defined in **Standard #1 (Tech Stack & Versions)**.

-   **Frontend:** Next.js 14 (App Router), React 18, TypeScript, Tailwind CSS
-   **Backend:** Next.js 14 (Route Handlers), NextAuth.js v5
-   **Database:** Prisma
-   **GitHub Integration:** `octokit` library for interacting with the GitHub REST API.

No new technologies are being introduced.

## 7. API Design

-   **Endpoint:** `POST /api/sources/create-from-template`
-   **Description:** Creates a new private GitHub repository on the user's account from a predefined template and registers it as a Source in Prism.
-   **Authentication:** Required (via `requireUserAsync`).
-   **Request Body:**
    ```json
    {
      "templateId": "company-os"
    }
    ```
-   **Success Response (201 Created):**
    ```json
    {
      "id": "clt4o5j3g0000c8s9b2k1a1f1",
      "name": "prism-company-os",
      "url": "https://github.com/user/prism-company-os"
    }
    ```
-   **Error Responses:**
    -   `400 Bad Request`: If `templateId` is missing or invalid. `{ "error": "Invalid templateId" }`
    -   `401 Unauthorized`: If the user is not authenticated.
    -   `404 Not Found`: If the `templateId` does not correspond to an existing template.
    -   `500 Internal Server Error`: For failures during GitHub API interaction or database operations. `{ "error": "Failed to create workspace." }`

## 8. Frontend Architecture

-   **Routing:** The feature utilizes the Next.js App Router. New routes are `/onboarding` and `/onboarding/setup`. This aligns with **Standard #3 (Next.js App Router Patterns)**.
-   **State Management:** For the wizard flow, state will be managed locally within the `OnboardingWizard.tsx` parent component using `useState`. This is sufficient for the short-lived, self-contained nature of the onboarding form and avoids the need for a global state manager.
-   **Component Hierarchy:** A clear parent-child hierarchy is used, with `OnboardingWizard` controlling the flow and rendering the appropriate step component (`StepConnectGitHub`, `StepTemplatePicker`, etc.).
-   **Data Fetching:** Data fetching from the new API endpoint will be handled using the standard browser `fetch` API within a client-side function, triggered by a user event (button click).

## 9. Standards Applied

This architecture explicitly adheres to the `prism-standards.md` document.

-   **Auth:** `requireUserAsync` is used for API auth (**Standard #5**).
-   **Database:** The `prisma` singleton is used, and all new data (`Source`) is scoped by `userId` (**Standard #6, #5**).
-   **Components:** Components use PascalCase, `.tsx`, named exports, and `interface` for props (**Standard #4**).
-   **TypeScript:** Strict mode is maintained, and Zod is used for API validation (**Standard #2**).
-   **API Design:** Error responses follow the established JSON shape (**Standard #10**).
-   **Anti-Patterns:** The design explicitly avoids the documented anti-patterns, such as using `requireUser` or direct DB access without user scoping (**Standard #11**).

## 10. Implementation Risks

-   **GitHub API Rate Limits:** The process of creating a repository and committing multiple files involves several API calls. While unlikely to be an issue for a single user, this could become a bottleneck at scale.
    -   **Mitigation:** Use a single tree commit (`git/trees` and `git/commits` APIs) instead of multiple file creation calls. This is more complex but far more efficient. The initial implementation will use the simpler multi-commit approach and will be benchmarked.
-   **Complex Error Handling:** The "happy path" is straightforward, but errors can occur at multiple stages (GitHub auth fails, token expires, repo name conflict, commit fails).
    -   **Mitigation:** The `RepoCreatorService` will be designed with robust `try...catch` blocks for each major step. Clear, user-friendly error messages will be mapped to specific failure modes and displayed in the UI.
-   **Performance Target for API:** The `< 5s P95` target for repository creation is aggressive. Large templates could slow down the file commit process.
    -   **Mitigation:** As above, use the Git Trees API for bulk commits. Additionally, keep initial templates small and focused. Monitor performance metrics from day one.

## 11. Performance Considerations

-   **End-to-End Onboarding (< 2 minutes):** Most of this time is user-dependent (time taken to auth with GitHub). The critical path controlled by our system is the API call.
-   **API Performance (< 5s P95):**
    -   **Caching:** The `WorkspaceTemplateProvider` can cache template file content in memory to avoid repeated file system reads.
    -   **API Call Optimization:** As mentioned in Risks, using the GitHub Git Data API (trees/blobs/commits) instead of the Contents API will significantly reduce the number of HTTP requests needed to populate the repository, from N (one per file) to ~3.
    -   **Database Operations:** The database inserts are single-row operations and will be extremely fast.
-   **Frontend Load:** The new components are lightweight. The use of server components where possible and minimal client-side state will ensure a fast initial load of the onboarding pages.

## 12. Testing Strategy

-   **Unit Tests (Jest & React Testing Library):**
    -   Each new React component (`OnboardingWelcome`, `StepTemplatePicker`, etc.) will have unit tests to verify rendering and basic interactions.
    -   The `RepoCreatorService` and `WorkspaceTemplateProvider` will be unit-tested with mocks for `prisma` and `Octokit`.
-   **Integration Tests (Jest):**
    -   The `POST /api/sources/create-from-template` route handler will be tested to ensure it correctly calls the service, handles authentication, and returns appropriate responses. The test will use a mocked `RepoCreatorService`.
-   **End-to-End Tests (Playwright):**
    -   A new E2E test suite will be created to simulate the full "Type B" user journey.
    -   **Scenario:**
        1.  Programmatically log in as a new user.
        2.  Navigate to `/onboarding`.
        3.  Click the "Start with a template" button.
        4.  Mock the GitHub OAuth flow to return a valid (test) token.
        5.  Select a template from the picker.
        6.  Click "Create Workspace" and wait for the API call (mocked at the network level to avoid actual GitHub calls) to complete.
        7.  Verify redirection to the new workspace URL.
        8.  Verify the "First Ask" empty state is present.
    -   This E2E test is the most critical for verifying the overall success of the feature and its performance targets.
-   **Jargon Audit (Manual):** As per Epic 5, a manual review of all user-facing text by a non-technical stakeholder will be a required step in the QA process.
