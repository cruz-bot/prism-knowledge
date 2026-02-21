---
epic: 2
story: 1
title: Build Bifurcated Welcome Screen
size: S
---

### User Story

As a new user, I want to see a clear choice between connecting an existing project or starting fresh with a template, so that I can choose the onboarding path that's right for me.

### Acceptance Criteria

- **Given** a new user lands on the `/onboarding` page after signing up,
- **When** the page loads,
- **Then** they see two distinct, clearly explained options: "Connect an existing repository" and "Get started with a template."
- **And** clicking the "Get started with a template" option navigates the user to `/onboarding/setup`.
- **And** clicking the "Connect an existing repository" option navigates the user to the existing `/console/connect` flow.

### Technical Approach

1.  **Page Creation:**
    -   Create a new Next.js App Router page at `app/onboarding/page.tsx`.
    -   This page will be a Server Component.

2.  **Session Handling:**
    -   Inside the page component, use `getServerSession()` from NextAuth v5 to retrieve the authenticated user's session.

3.  **Component Implementation:**
    -   Create a new component `PathSelector` that will act as a container for the two choices.
    -   Create a new reusable component `PathOption` that accepts `title`, `description`, and `href` as props. This component should render a clickable card-like element.
    -   (Optional) Create a `WelcomeHeader` component to display a personalized greeting using the session information.

4.  **Structure:**
    -   The `app/onboarding/page.tsx` component will render the `WelcomeHeader` and the `PathSelector`.
    -   The `PathSelector` will contain two instances of `PathOption`:
        1.  `title`: "Connect Existing Repository", `description`: "For users who already have a knowledge base in GitHub", `href`: `/console/connect`.
        2.  `title`: "Get Started with a Template", `description`: "For users new to GitHub, we'll create one for you", `href`: `/onboarding/setup`.

### Testing Strategy

-   **Component Tests (e.g., with Storybook or Jest):**
    -   Create tests for the `PathOption` component to verify it renders the correct text and links to the correct `href`.
-   **End-to-End Test (e.g., with Playwright or Cypress):**
    -   Write a test that logs in a new user and navigates them to the `/onboarding` page.
    -   Assert that two choice cards are visible on the screen with the expected text content.
    -   Simulate a click on the "Get started with a template" card and assert that the URL changes to `/onboarding/setup`.
    -   Run a similar test for the "Connect an existing repository" card, asserting navigation to `/console/connect`.

### Definition of Done

-   [ ] The new page at `/onboarding` is created and handles user sessions.
-   [ ] The `PathSelector` and `PathOption` components are implemented and styled.
-   [ ] The page clearly presents the two onboarding choices.
-   [ ] Clicking the "Type B" path correctly navigates the user to `/onboarding/setup`.
-   [ ] Automated tests (component and/or E2E) are written and passing.
-   [ ] The UI is responsive and meets accessibility standards.
-   [ ] Code is reviewed and merged.
