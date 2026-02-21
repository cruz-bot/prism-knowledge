---
epic: 4
story: 2
title: Integrate Frontend Wizard with Backend API
size: M
---

### User Story

As a Type B user, I want to click "Create Workspace" and be taken directly to my new, functional workspace, so that my onboarding journey is fast and seamless.

### Acceptance Criteria

- **Given** a user has selected a template in the wizard,
- **When** the final creation step is initiated,
- **Then** the frontend makes a `POST` request to the `/api/sources/create-from-template` endpoint with the selected template ID.
- **And** a loading state is displayed to the user while the request is in progress.
- **And** upon a successful API response, the user is automatically redirected to their new workspace console page (e.g., `/console/[workspaceId]`).
- **And** if the API response is an error, a user-friendly error message is displayed.

### Technical Approach

1.  **Creation Step Component:**
    -   Create a new client component at `src/components/onboarding/WorkspaceCreationStep.tsx`.
    -   This component will accept the selected `templateId` as a prop.

2.  **Wizard Integration:**
    -   In `app/onboarding/setup/page.tsx`, conditionally render the `WorkspaceCreationStep` component when `step === 3` and a `selectedTemplate` is present.
    -   Pass the `selectedTemplate` ID to the component as a prop.

3.  **API Call Logic:**
    -   Inside `WorkspaceCreationStep`, use a `useEffect` hook with an empty dependency array `[]` so it runs once when the component mounts.
    -   Within the `useEffect`, define and call an async function that uses the `fetch` API to make a `POST` request to `/api/sources/create-from-template`.
    -   The request `body` should be a JSON string containing the `templateId`.
    -   The `headers` must include `'Content-Type': 'application/json'`.

4.  **State Management:**
    -   Use a state variable `const [status, setStatus] = useState<'loading' | 'success' | 'error'>('loading');` to track the API call's progress.
    -   Conditionally render UI based on this status:
        -   `'loading'`: Display a loading spinner or animation with text like "Creating your workspace...".
        -   `'success'`: Display a success message like "Workspace created! Redirecting...".
        -   `'error'`: Display an error message component.

5.  **Redirection and Error Handling:**
    -   Import `useRouter` from `next/navigation`.
    -   In the `try` block of your API call, if the response is successful (`response.ok`), parse the JSON to get the new `source.id`. Set status to `'success'` and then use `setTimeout` to delay for a second before calling `router.push(\`/console/${source.id}?firstRun=true\`)`.
    -   In the `catch` block, set status to `'error'` and store the error message to be displayed.

### Testing Strategy

-   **Component Tests:**
    -   Write tests for the `WorkspaceCreationStep` component.
    -   Mock the global `fetch` function.
    -   **Success Case:** Configure the `fetch` mock to resolve with a successful `201` response containing a dummy source ID. Assert that the component's state transitions from `loading` to `success` and that the mocked `useRouter().push` function is eventually called with the correct URL.
    -   **Error Case:** Configure the `fetch` mock to reject or resolve with a `500` error. Assert that the state transitions from `loading` to `error` and that an error message is rendered.
-   **End-to-End Test:**
    -   At the E2E level, mock the API endpoint itself using your test runner's network interception features (e.g., `cy.intercept()` or `page.route()`).
    -   Run the full onboarding flow. After the user clicks a template, the test should:
        1.  Wait for the API call to be intercepted.
        2.  Return a successful mock response.
        3.  Assert that the page URL eventually changes to the expected console URL.

### Definition of Done

-   [ ] The `WorkspaceCreationStep.tsx` component is implemented.
-   [ ] The component is correctly rendered as the final step of the wizard.
-   [ ] On mount, it triggers the `POST /api/sources/create-from-template` API call.
-   [ ] A user-facing loading state is shown during the API call.
-   [ ] On success, the user is redirected to the new workspace console, including the `?firstRun=true` query parameter.
-   [ ] On failure, a clear error message is displayed.
-   [ ] Automated component and E2E tests are written and passing.
-   [ ] Code is reviewed and merged.
