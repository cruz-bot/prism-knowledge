---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11]
inputDocuments: ['C:\Users\cruzb\.openclaw\workspace\_bmad-output\planning-artifacts\prism-product-brief.md']
workflowType: 'prd'
lastStep: 11
project_name: 'Frictionless Onboarding: GitHub for Everyone'
user_name: 'BMAD PM Agent'
date: '2026-02-20'
---

# Product Requirements Document: Frictionless Onboarding: GitHub for Everyone

**Author:** BMAD PM Agent
**Date:** 2026-02-20
**Version:** 1.0

## 1. Introduction & Background

Prism is a UX layer designed to make GitHub accessible to a broader audience, transforming it into an AI-powered operating system for businesses. The "Frictionless Onboarding: GitHub for Everyone" feature is a critical initiative to bridge the gap between technical and non-technical users. It addresses the primary adoption hurdle for non-developers: the complexity and intimidation of the native GitHub experience.

This feature will guide new, non-technical users (referred to as "Type B" users) through a seamless wizard. The goal is to take them from initial sign-up to their first AI-powered answer from their own knowledge base—stored in a brand new repository on their own GitHub account—in under two minutes. This reinforces Prism's core principle of user data ownership and open standards, while abstracting away the underlying complexity of Git and GitHub.

## 2. Problem Statement

GitHub is the standard for software development, but its developer-centric design makes it a walled garden for non-technical team members. Business knowledge (product specs, marketing plans, company policies) becomes siloed in separate, often proprietary systems like Notion, Confluence, or Google Docs. This fragmentation leads to:

-   **Inefficiency:** Constant context switching and searching for the "right" document.
-   **Misalignment:** Discrepancies between what's in the code and what's in the business documents.
-   **Vendor Lock-In:** Critical business knowledge is trapped in closed ecosystems, making it difficult to migrate or integrate.
-   **Collaboration Barriers:** Non-technical users are dependent on developers to access or update information stored alongside the code.

Prism aims to solve this by providing an accessible on-ramp to GitHub, not a replacement for it.

## 3. Target Audience & User Personas

This feature primarily targets the **Non-Technical Newcomer**, enabling them to collaborate with the **Technical Adopter** within a unified knowledge ecosystem.

### Persona 1: Brenda (The Non-Technical Newcomer)

-   **Role:** Head of Operations, Product Manager, or other non-developer business user.
-   **Goals:**
    -   To be an empowered, self-sufficient contributor to the company's knowledge base.
    -   A single, reliable place to find answers and manage documents without needing technical help.
-   **Frustrations:**
    -   Feels disconnected from the core knowledge base stored in GitHub.
    -   Intimidated by developer tools and jargon.
    -   Worries that her documents in Notion/GDocs are out of sync with the engineering team's reality.

### Persona 2: Alex (The Technical Adopter)

-   **Role:** CTO, Engineering Lead, or technical founder.
-   **Goals:**
    -   Establish a single source of truth for the entire company within the GitHub ecosystem.
    -   Empower non-technical colleagues to collaborate directly with the knowledge base.
-   **Frustrations:**
    -   The "knowledge divide" between technical and non-technical teams.
    -   Information is fragmented across GDocs, Notion, and Slack, creating sync issues.

## 4. Goals and Success Metrics

### Product Goals

-   **Deliver an immediate "Aha!" moment:** Users should experience the core value of Prism within two minutes of signing up.
-   **Maximize onboarding conversion:** Create a frictionless path for new users to successfully set up a workspace.
-   **Drive long-term engagement:** Ensure the initial positive experience translates into sustained, active use.
-   **Reinforce data ownership:** Uphold the principle that users' data lives in their own accounts, in an open format.

### Key Success Metrics

| Metric | Description | Target |
| :--- | :--- | :--- |
| **Time to First Answer** | Time from signup completion to the user's first successful AI-powered answer. | **< 2 minutes** |
| **Onboarding Completion Rate**| % of users who start the wizard and successfully create a GitHub-backed workspace. | **> 60%** |
| **Template Adoption Rate**| % of new users who bootstrap their workspace from a template. | **> 30%** |
| **Day-7 Retention**| % of new users who return to use the product within the first 7 days. | **> 40%** |

## 5. Functional Requirements

### FR1: Bifurcated Onboarding Entry Point

-   **FR1.1:** After signing up and logging in, new users shall be directed to a welcome screen at `/onboarding`.
-   **FR1.2:** The welcome screen shall present two paths:
    -   **Path A (Existing GitHub Users):** "Connect an existing repository." (This path utilizes the existing RepoConnector wizard).
    -   **Path B (New to GitHub):** "Get started with a template." (This path initiates the new onboarding wizard).
-   **FR1.3:** The UI shall clearly explain the benefits of each path to help users choose correctly.

### FR2: Guided GitHub Setup Wizard (Type B Flow)

-   **FR2.1:** The wizard shall consist of three steps presented at `/onboarding/setup`.
-   **FR2.2: Step 1: Connect GitHub Account.**
    -   The UI will prompt the user to connect their GitHub account.
    -   It will use the existing GitHub Device Flow OAuth (`src/lib/github-auth.ts`) and the `GitHubAuthFlow.tsx` component.
    -   The flow must handle users who do not have a GitHub account by providing clear, in-context instructions and a link to sign up for GitHub.
-   **FR2.3: Step 2: Choose a Workspace Template.**
    -   The UI will display a `TemplatePicker` component.
    -   The user can select from four pre-defined templates: "Company OS," "Product Workspace," "Team Docs," and "Startup OS."
    -   Each template will have a name, a short description, and an icon/visual.
-   **FR2.4: Step 3: Create Workspace.**
    -   Upon template selection, a "Create Workspace" button is enabled.
    -   Clicking this button triggers the repository creation process.
    -   A loading/progress indicator must be shown while the backend process is running.

### FR3: Workspace Template System

-   **FR3.1:** A new system shall be implemented at `src/lib/workspace-templates/` to manage workspace templates.
-   **FR3.2:** Each template shall be a directory containing a set of Markdown files and sub-folders that constitute the initial repository content.
-   **FR3.3:** The system must be able to retrieve the contents of a specified template to be sent to the API.

### FR4: Backend API for Template-Based Repository Creation

-   **FR4.1:** A new API endpoint shall be created: `POST /api/sources/create-from-template`.
-   **FR4.2:** The endpoint will require authentication and accept a `templateId` in the request body.
-   **FR4.3:** Using the user's stored GitHub OAuth token, the endpoint will perform the following actions via the GitHub API:
    -   Create a new **private** repository on the authenticated user's GitHub account. The repository name can be generated from the template name (e.g., "prism-company-os").
    -   Commit the full file and directory structure of the selected template to the `main` branch of the new repository.
-   **FR4.4:** After successfully creating and populating the repository, the endpoint will create a new `Source` record in the Prism database, linking the user to their new repository.
-   **FR4.5:** The API response shall indicate success or failure and provide the URL of the newly created workspace.

### FR5: Post-Creation Flow & Empty State

-   **FR5.1:** Upon a successful response from the `create-from-template` API, the user shall be automatically redirected to their new workspace console page (e.g., `/console/[workspaceId]`).
-   **FR5.2:** The console for a newly created workspace will feature an updated "Empty State" or welcome component that guides the user to their first action: using the "Ask Prism" chat interface.
-   **FR5.3:** The "Ask Prism" interface should be immediately functional for the new repository.

## 6. Non-Functional Requirements (NFRs)

| Category | Requirement | Metric/Target |
| :--- | :--- | :--- |
| **Performance** | End-to-end onboarding flow duration. | **< 2 minutes** |
| | API latency for workspace creation. | **P95 < 5 seconds** for `POST /api/sources/create-from-template` |
| **Reliability** | GitHub API interaction success rate. | **> 99.5%** |
| **Security** | Zero regressions in authentication or authorization. | **0 regressions** |
| | Repository default visibility. | **Must be Private** |
| **Testability** | Code coverage for new components and APIs. | **> 85%** |
| **Usability** | Jargon and language must be accessible to non-technical users. | Pass a "jargon audit" |

## 7. Out of Scope (for this version)

-   **"Start from Scratch" / Blank Template:** Users must select one of the four provided templates.
-   **GitHub Organization Support:** Repositories will only be created on the user's personal GitHub account. Organization support is a fast-follow.
-   **Custom User Templates:** Users cannot provide their own templates.
-   **Content Importers:** No importing from Notion, Google Docs, etc.
-   **Onboarding for Existing GitHub Users (Type A):** This feature is exclusively for the "Type B" flow. Type A users continue to use the existing manual connection method.
-   **Advanced Error Recovery:** The MVP will focus on the "happy path." Extensive error handling for edge cases in the GitHub signup or auth flow is deferred.

## 8. Security Considerations

-   The pre-existing P0 security issues (`/api/agents/custom` auth, in-memory trace store, legacy persistence routes) are not part of this feature's scope but must be addressed by a separate, parallel effort as a P0 gate for release.
-   All new API endpoints (`/api/sources/create-from-template`) must have robust authentication and authorization checks.
-   GitHub OAuth tokens must be stored securely.
-   Repositories created on the user's behalf must default to **private** to protect user data.
