---
epic: 5
story: 1
title: Review and Refine All UI Copy
size: S
---

### User Story

As a non-technical user, I want to understand all the text, buttons, and instructions in the onboarding flow without feeling confused, so that I can complete the setup process confidently.

### Acceptance Criteria

- **Given** all UI components for the onboarding flow are functionally complete,
- **When** a user matching the non-technical persona reviews the end-to-end flow,
- **Then** they report no points of confusion related to technical jargon (e.g., "repository," "commit," "branch").
- **And** all user-facing copy has been updated to use simpler, more intuitive language (e.g., "workspace," "version," "save").

### Technical Approach

1.  **Component Audit:**
    -   Compile a list of all new or modified UI components involved in the onboarding flow that contain user-facing text. This includes, but is not limited to:
        -   `app/onboarding/page.tsx`
        -   `src/components/onboarding/GitHubConnectionStep.tsx`
        -   `src/components/onboarding/TemplatePicker.tsx`
        -   `src/components/onboarding/WorkspaceCreationStep.tsx` (and its loading/error states)
        -   `src/components/console/FirstRunEmptyState.tsx`

2.  **Initial Review:**
    -   Perform a developer-led review of the text in all identified components.
    -   Replace any obvious developer-centric terms with their user-friendly analogues as defined in the project's glossary.
    -   Examples:
        -   `Repository` → `Workspace`
        -   `Main Branch` → `Main Version`
        -   `Commit` → `Save` / `Update`

3.  **User Acceptance Testing (UAT):**
    -   Schedule a review session with a non-technical stakeholder or test user.
    -   Have them go through the entire onboarding flow from start to finish.
    -   Encourage them to "think aloud" and point out any words, phrases, or instructions that are unclear or intimidating.
    -   Document all feedback meticulously.

4.  **Implementation:**
    -   Based on the UAT feedback, perform a final pass on all components and update the copy to address the points of confusion.

### Testing Strategy

-   **User Acceptance Testing (UAT):** This is the primary validation method for this story. The success of the story is measured by the positive feedback from a non-technical user.
-   **Snapshot Testing:** If the project uses Jest snapshot testing, these snapshots must be updated after the copy changes are finalized. This helps prevent accidental regressions in the future.
-   **Manual Review:** The product manager or designer should perform a final review of the entire flow to sign off on the language and tone.

### Definition of Done

-   [ ] A comprehensive audit of all user-facing text in the onboarding flow has been completed.
-   [ ] An initial pass to remove obvious jargon has been performed.
-   [ ] At least one UAT session with a non-technical user has been conducted and feedback has been collected.
-   [ ] All required copy changes based on the feedback have been implemented across all relevant components.
-   [ ] Jest snapshots (if applicable) have been updated.
-   [ ] The final flow is confirmed to be clear, simple, and accessible for the target "Brenda" persona.
-   [ ] Code is reviewed and merged.
