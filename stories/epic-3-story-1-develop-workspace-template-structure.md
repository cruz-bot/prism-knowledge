---
epic: 3
story: 1
title: Develop Workspace Template Structure
size: S
---

### User Story

As a developer, I want a file-based system at `src/lib/workspace-templates/` to define workspace templates, so that I can easily add, update, and manage the templates.

### Acceptance Criteria

- **Given** the four required templates are defined (Company OS, Product Workspace, Team Docs, Startup OS),
- **When** the `src/lib/workspace-templates/` directory is inspected,
- **Then** each template exists as a subdirectory (e.g., `company-os`).
- **And** each subdirectory contains a nested structure of Markdown files and folders representing the template's content.
- **And** a service exists that can read these files and their content.

### Technical Approach

1.  **Directory Structure:**
    -   Create the base directory: `src/lib/workspace-templates/`.
    -   Inside this directory, create the four template subdirectories: `company-os`, `product-workspace`, `team-docs`, and `startup-os`.
    -   Populate each subdirectory with a few placeholder Markdown files and folders as outlined in the architecture document. For example:
        ```
        src/lib/workspace-templates/
        └── company-os/
            ├── README.md
            └── team/
                └── org-chart.md
        ```

2.  **Template Service:**
    -   Create a new file: `src/lib/services/template-service.ts`.
    -   Inside this file, define and implement the `TemplateService` class.

3.  **Service Implementation:**
    -   Define a static `TEMPLATE_REGISTRY` object to store the metadata (id, name, description, repoNamePrefix) for each of the four templates.
    -   Implement the static method `getTemplateMetadata(templateId: string)`, which retrieves metadata from the registry.
    -   Implement the static async method `getTemplateFiles(templateId: string)`. This method will use Node's `fs/promises` API to recursively read the directory corresponding to the `templateId` and return an array of objects, each containing the file's relative `path` and `content`.
    -   Implement the static method `generateRepoName(templateId: string)` to create a unique repository name based on the template's prefix and a timestamp.

### Testing Strategy

-   **Unit Tests (for `TemplateService`):**
    -   Use a file system mocking library (like `memfs` or `jest.mock('fs/promises')`) to simulate the template directory structure in memory.
    -   Write a test for `getTemplateMetadata` to verify it returns correct data for a valid ID and throws an error for an invalid one.
    -   Write a test for `getTemplateFiles` that calls the function and asserts that the returned array of files and their content matches the mocked directory structure.
    -   Write a test for `generateRepoName` to check that it returns a string containing the correct prefix and is unique on subsequent calls.

### Definition of Done

-   [ ] The `src/lib/workspace-templates/` directory and its subdirectories are created and populated.
-   [ ] The `src/lib/services/template-service.ts` file is created with the `TemplateService` class.
-   [ ] All required static methods (`getTemplateMetadata`, `getTemplateFiles`, `generateRepoName`) are implemented correctly.
-   [ ] The `TemplateService` is fully unit tested, and all tests are passing.
-   [ ] Code is reviewed and merged.
