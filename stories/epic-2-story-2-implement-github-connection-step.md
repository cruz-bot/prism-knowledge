---
epic: 2
story: 2
title: Implement GitHub Connection Step
size: M
---

### User Story

As a Type B user (new to GitHub), I want a simple, guided step to connect my GitHub account, so that Prism can create my workspace on my behalf.

### Acceptance Criteria

- **Given** a user is on the `/onboarding/setup` page,
- **When** the page loads,
- **Then** they are presented with the first step of a wizard, prompting them to connect their GitHub account.
- **And** the UI for this step displays the `GitHubAuthFlow.tsx` component to initiate the Device Flow.
- **And** the UI includes clear, non-technical instructions and a link for users who need to create a GitHub account first.
- **And** upon successful GitHub authentication, the wizard automatically transitions to the next step (Template Selection).

### Technical Approach

1.  **Wizard Page Creation:**
    -   Create the primary wizard file at `app/onboarding/setup/page.tsx`.
    -   This file must be a Client Component (`'use client'`) to handle state and user interaction.

2.  **State Management:**
    -   Use the `useState` hook to manage the current step of the wizard. For example: `const [step, setStep] = useState(1);`.

3.  **Component Structure:**
    -   Create a `WizardContainer` component that takes `currentStep` as a prop and displays the overall progress (e.g., "Step 1 of 3").
    -   Create a `GitHubConnectionStep.tsx` component that will be conditionally rendered by the main wizard page when `step === 1`.

4.  **Auth Flow Integration:**
    -   The `GitHubConnectionStep` component will be responsible for rendering the existing `GitHubAuthFlow.tsx` component.
    -   **Refactor `GitHubAuthFlow.tsx`:** Modify this existing component to accept a new prop, `onSuccess: () => void`. This callback function must be invoked when the component successfully retrieves a GitHub token.
    -   In the main wizard page (`page.tsx`), pass a function to this prop that updates the wizard's state: `<GitHubConnectionStep onSuccess={() => setStep(2)} />`.

5.  **User Guidance:**
    -   Within `GitHubConnectionStep.tsx`, add user-friendly text explaining why the connection is needed.
    -   Include a prominent, clearly labeled link for users who do not have a GitHub account: `<a href="https://github.com/signup" target="_blank" rel="noopener noreferrer">Don't have a GitHub account? Sign up here.</a>`.

### Testing Strategy

-   **Component Tests:**
    -   Write a test for `GitHubConnectionStep` to ensure it renders the instructional copy, the link to GitHub's signup page, and the `GitHubAuthFlow` component.
-   **Integration/E2E Test:**
    -   Testing the full OAuth flow end-to-end is complex. The strategy should focus on verifying the wizard's state transition:
        1.  Navigate to `/onboarding/setup`.
        2.  Assert that the `GitHubConnectionStep` is visible.
        3.  Mock the `GitHubAuthFlow` component to simulate a successful authentication and trigger the `onSuccess` callback.
        4.  Assert that the wizard's UI updates to show the content for Step 2 (e.g., the Template Picker is now visible).

### Definition of Done

-   [ ] The main stateful wizard page (`app/onboarding/setup/page.tsx`) is implemented.
-   [ ] The `GitHubConnectionStep` component is created and renders the auth flow.
-   [ ] `GitHubAuthFlow.tsx` is refactored to include and call the `onSuccess` callback.
-   [ ] The wizard UI provides clear guidance for users without a GitHub account.
-   [ ] On successful authentication, the wizard correctly transitions from step 1 to step 2.
-   [ ] Automated tests are written to verify the component rendering and state transition.
-   [ ] Code is reviewed and merged.
