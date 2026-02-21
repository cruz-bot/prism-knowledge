---
epic: 4
story: 3
title: Implement "First Ask" Empty State
size: S
---

### User Story

As a new user, I want to be guided to ask my first question immediately upon entering my new workspace, so that I can experience the "Aha!" moment right away.

### Acceptance Criteria

- **Given** a user is redirected to their new workspace console page,
- **When** the URL contains the query parameter `?firstRun=true`,
- **Then** a special "First Run Empty State" component is displayed.
- **And** this component welcomes the user and prominently guides them to use the "Ask Prism" chat interface, potentially showing example questions.
- **And** the "Ask Prism" chat interface is immediately functional for the newly created repository.

### Technical Approach

1.  **Empty State Component:**
    -   Create a new client component at `src/components/console/FirstRunEmptyState.tsx`.
    -   This component will contain:
        -   A large, friendly heading (e.g., "Welcome to your workspace! 🎉").
        -   A short paragraph explaining that the workspace is ready and they should try asking a question.
        -   An `ExampleQuestions` sub-component that renders several clickable, predefined questions relevant to a new workspace (e.g., "What's in this workspace?").

2.  **Console Page Integration:**
    -   The main console page (likely `app/console/[workspaceId]/page.tsx`) must be a client component (or have a client component wrapper) to access query parameters.
    -   Use the `useSearchParams` hook from `next/navigation` to read the URL's query string.
    -   Check if the `firstRun` parameter is equal to `'true'`.
    -   Use a state variable `const [showFirstRun, setShowFirstRun] = useState(searchParams.get('firstRun') === 'true');`.
    -   Conditionally render the `FirstRunEmptyState` component if `showFirstRun` is true. Otherwise, render the standard console view.

3.  **Dismissal Logic:**
    -   Pass a dismiss function to the `FirstRunEmptyState` component (e.g., `onDismiss={() => setShowFirstRun(false)}`).
    -   This function should be called when the user interacts with the page in a way that moves beyond the initial state, such as clicking an example question or focusing the main chat input bar. This provides a smooth transition to the standard UI.

### Testing Strategy

-   **Component Tests (Storybook or Jest):**
    -   Write a test for `FirstRunEmptyState` to ensure it renders the welcome message and example questions correctly.
    -   Simulate a click on an example question and assert that the `onDismiss` or `onQuestionClick` callback is invoked.
-   **End-to-End Test:**
    -   This will be the final assertion in the full onboarding E2E test flow.
    -   After the final redirection to `/console/[id]?firstRun=true`, assert that the `FirstRunEmptyState` component is visible.
    -   Verify the welcome text is present.
    -   (Optional) Test the dismissal by simulating a click and asserting the component is removed from the DOM.
    -   Navigate to the same URL without the query parameter and assert that the `FirstRunEmptyState` component is *not* visible.

### Definition of Done

-   [ ] The `FirstRunEmptyState.tsx` component is implemented and styled.
-   [ ] The main console page is updated to use `useSearchParams` and conditionally render the component.
-   [ ] The empty state is only shown when the `?firstRun=true` parameter is present in the URL.
-   [ ] The component provides clear guidance to the user.
-   [ ] A dismissal mechanism is in place.
-   [ ] Automated tests are written for the component and the E2E flow.
-   [ ] Code is reviewed and merged.
