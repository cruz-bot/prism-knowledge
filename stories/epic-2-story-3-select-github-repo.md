---
epic: 2
story: 2.3
title: Select GitHub Repository for Template Setup
size: M
---

### User Story

As a Developer who has just connected my GitHub account,
I want to see a list of my repositories,
So that I can select one to use as the source for my first Prism workspace template.

### Acceptance Criteria

*   **Given** a user has connected their GitHub account,
*   **When** they navigate to the repository selection page,
*   **Then** a list of their GitHub repositories is displayed.
*   **And** they can select one repository from the list.
*   **And** upon selection, they are advanced to the next step of onboarding.

### Technical Approach

(See Linear issue CRU-67 for detailed implementation)

### Testing Strategy

- Unit tests for core functionality
- Integration tests for API endpoints
- E2E tests for user flows

### Definition of Done

- [ ] Code written and reviewed
- [ ] All tests passing (100% pass rate)
- [ ] No regressions
- [ ] Documentation updated
- [ ] PR merged
