---
epic: 4
story: 1
title: Build Template Picker UI
size: S
---

### User Story

As a Type B user, I want to see and choose from a list of predefined workspace templates, so that I can bootstrap my knowledge base for a specific purpose.

### Acceptance Criteria

- **Given** a user is on the second step of the `/onboarding/setup` wizard,
- **When** the UI loads,
- **Then** a `TemplatePicker` component is displayed, showing four available templates with their names and descriptions.
- **And** the "Create Workspace" button/step is initially disabled or hidden.
- **And** upon selecting a template, the wizard proceeds to the next step, enabling the workspace creation process.

### Technical Approach

1.  **Wizard State Management:**
    -   In `app/onboarding/setup/page.tsx`, introduce a new state variable to hold the chosen template ID: `const [selectedTemplate, setSelectedTemplate] = useState<string | null>(null);`.

2.  **Conditional Rendering:**
    -   Update the wizard's conditional logic to render the `TemplatePicker` component when `step === 2`.

3.  **`TemplatePicker` Component:**
    -   Create a new client component at `src/components/onboarding/TemplatePicker.tsx`.
    -   This component will accept an `onSelect` prop: `onSelect: (templateId: string) => void`.
    -   Inside the component, define an array containing the metadata for the four templates (id, name, description, icon, color), as specified in the architecture document.
    -   Map over this array to render a `TemplateCard` for each template.

4.  **`TemplateCard` Sub-Component:**
    -   Create a smaller, reusable component for rendering a single template card.
    -   It should display the template's icon, name, and description.
    -   It will handle the `onClick` event, which will call the `onSelect` function passed down from the `TemplatePicker`, providing the template's ID.

5.  **Wizard Integration:**
    -   In `app/onboarding/setup/page.tsx`, pass a callback to the `TemplatePicker`'s `onSelect` prop. This callback will do two things:
        1.  `setSelectedTemplate(templateId);`
        2.  `setStep(3);`
    -   This ensures that selecting a template both captures the choice and advances the wizard to the final creation step.

### Testing Strategy

-   **Component Tests (Storybook or Jest):**
    -   Test `TemplateCard` to ensure it renders all props correctly.
    -   Test `TemplatePicker` to ensure it renders the correct number of cards.
    -   Simulate a click on a `TemplateCard` within the `TemplatePicker` test and assert that the `onSelect` callback is invoked with the correct template ID.
-   **End-to-End Test:**
    -   Extend the existing wizard E2E test.
    -   After mocking the GitHub authentication step, assert that the template picker UI is visible.
    -   Simulate a click on a template card.
    -   Assert that the UI transitions to the final "Creating Workspace..." step.

### Definition of Done

-   [ ] The `TemplatePicker.tsx` component is created and renders four distinct templates.
-   [ ] The `TemplateCard.tsx` sub-component is created and styled.
-   [ ] The picker is correctly integrated as Step 2 of the onboarding wizard.
-   [ ] Clicking a template updates the wizard's state and advances it to Step 3.
-   [ ] Automated component tests are written and passing.
-   [ ] The UI is responsive and visually polished.
-   [ ] Code is reviewed and merged.
