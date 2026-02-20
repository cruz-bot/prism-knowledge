---
project_name: 'Frictionless Onboarding: GitHub for Everyone'
user_name: 'BMAD PM Agent'
date: '2026-02-20'
---

# Epics: Frictionless Onboarding: GitHub for Everyone

**Author:** BMAD PM Agent
**Date:** 2026-02-20

---

## Overview

This document breaks down the "Frictionless Onboarding: GitHub for Everyone" feature into five distinct epics. These epics are designed to be delivered sequentially, with a critical security epic acting as a gate for the entire feature release. The breakdown is derived from the [Product Requirements Document](./prism-prd.md).

### Epic Summary

1.  **Security Hardening (P0 Gate):** Addresses critical, pre-existing security vulnerabilities that must be fixed before the new onboarding feature can be safely launched.
2.  **Welcome & GitHub Setup (Type B Entry):** Builds the core user-facing wizard that guides non-technical users through account connection and setup.
3.  **Workspace Template System:** Implements the backend infrastructure for creating new repositories from predefined templates.
4.  **Integration & Aha Moment:** Ties the frontend and backend together, ensuring a seamless end-to-end flow that culminates in the user's first successful interaction.
5.  **Language & Jargon Audit:** A cross-cutting epic focused on ensuring all user-facing language is clear, simple, and accessible to a non-technical audience.

---

## Epic 1: Security Hardening (P0 Gate)

**Goal:** Remediate all known P0 security vulnerabilities in the existing platform to ensure a safe environment for new user onboarding. This is a release-blocking epic.

### Stories

**Story 1.1: Secure Custom Agent API**
*As a* platform operator,
*I want* the `/api/agents/custom` endpoint to be protected by authentication and authorization,
*So that* only authenticated users can access it and prevent a SEV-1 vulnerability.

**Acceptance Criteria:**
*   **Given** a user is not authenticated,
*   **When** they attempt to POST to `/api/agents/custom`,
*   **Then** they receive a `401 Unauthorized` error.
*   **And** a valid, authenticated request from an authorized user succeeds.

**Story 1.2: Implement Persistent Trace Store**
*As a* developer,
*I want* agent traces to be persisted to a durable database,
*So that* trace history is not volatile and survives server restarts.

**Acceptance Criteria:**
*   **Given** an agent has completed a run,
*   **When** the server is restarted,
*   **Then** the full trace of the agent's run is still available for review.

**Story 1.3: Migrate Legacy Persistence Routes**
*As a* developer,
*I want* the 7 legacy API routes to be migrated to the current persistence layer,
*So that* we eliminate outdated code paths and ensure data consistency.

**Acceptance Criteria:**
*   **Given** the list of 7 legacy routes,
*   **When** each route is individually tested,
*   **Then** all data operations correctly use the new Prisma-based persistence layer.
*   **And** the old persistence layer code is removed.

---

## Epic 2: Welcome & GitHub Setup (Type B Entry)

**Goal:** Create the initial user-facing onboarding wizard that guides a "Type B" user (new to GitHub) from a welcome screen to successfully connecting their GitHub account.

### Stories

**Story 2.1: Build Bifurcated Welcome Screen**
*As a* new user,
*I want* to see a clear choice between connecting an existing project or starting fresh with a template,
*So that* I can choose the onboarding path that's right for me.

**Acceptance Criteria:**
*   **Given** a new user lands on `/onboarding` after signup,
*   **When** the page loads,
*   **Then** they see two distinct options: "Connect an existing repository" and "Get started with a template."
*   **And** clicking the "Type B" option navigates them to `/onboarding/setup`.

**Story 2.2: Implement GitHub Connection Step**
*As a* Type B user,
*I want* a simple, guided step to connect my GitHub account,
*So that* Prism can create my workspace on my behalf.

**Acceptance Criteria:**
*   **Given** a user is on the first step of the `/onboarding/setup` wizard,
*   **When** they initiate the connection process,
*   **Then** the `GitHubAuthFlow.tsx` component is displayed and initiates the Device Flow.
*   **And** the UI provides a clear link and instructions for users who need to create a GitHub account first.
*   **And** upon successful authentication, the wizard automatically proceeds to the next step.

---

## Epic 3: Workspace Template System

**Goal:** Build the backend system for managing and creating new workspaces from a set of predefined templates.

### Stories

**Story 3.1: Develop Workspace Template Structure**
*As a* developer,
*I want* a file-based system at `src/lib/workspace-templates/` to define workspace templates,
*So that* I can easily add, update, and manage the templates.

**Acceptance Criteria:**
*   **Given** the four required templates (Company OS, Product Workspace, Team Docs, Startup OS),
*   **When** I inspect the `src/lib/workspace-templates/` directory,
*   **Then** each template exists as a subdirectory containing its full content structure.

**Story 3.2: Create Template-Based Repository API Endpoint**
*As a* developer,
*I want* a `POST /api/sources/create-from-template` endpoint,
*So that* the frontend can trigger the creation of a new workspace from a template.

**Acceptance Criteria:**
*   **Given** a user is authenticated and has a valid GitHub token,
*   **When** the frontend sends a POST request with a `templateId`,
*   **Then** the API creates a new **private** repository on the user's GitHub account.
*   **And** the API commits all files from the corresponding template directory into the new repository.
*   **And** a new `Source` is created in the Prism database.
*   **And** the API returns a success status and the new workspace URL.
*   **And** the P95 latency for the entire operation is under 5 seconds.

---

## Epic 4: Integration & Aha Moment

**Goal:** Integrate the frontend wizard with the backend API to create a seamless, end-to-end user experience that culminates in the "Aha!" moment of asking a question.

### Stories

**Story 4.1: Build Template Picker UI**
*As a* Type B user,
*I want* to see and choose from a list of predefined workspace templates,
*So that* I can bootstrap my knowledge base for a specific purpose.

**Acceptance Criteria:**
*   **Given** a user has connected their GitHub account in the wizard,
*   **When** they land on the second step of `/onboarding/setup`,
*   **Then** a `TemplatePicker` component displays the four available templates with names and descriptions.
*   **And** selecting a template enables the "Create Workspace" button.

**Story 4.2: Integrate Frontend Wizard with Backend API**
*As a* Type B user,
*I want* to click "Create Workspace" and be taken directly to my new, functional workspace,
*So that* my onboarding journey is fast and seamless.

**Acceptance Criteria:**
*   **Given** a user has selected a template in the wizard,
*   **When** they click "Create Workspace",
*   **Then** the frontend calls the `POST /api/sources/create-from-template` endpoint.
*   **And** a loading state is displayed during creation.
*   **And** upon a successful API response, the user is automatically redirected to their new console page.

**Story 4.3: Implement "First Ask" Empty State**
*As a* new user,
*I want* to be guided to ask my first question immediately upon entering my new workspace,
*So that* I can experience the "Aha!" moment right away.

**Acceptance Criteria:**
*   **Given** a user has just been redirected to their new workspace,
*   **When** the console page loads,
*   **Then** an updated empty state component is shown that prompts the user to try the "Ask Prism" feature.
*   **And** the "Ask Prism" chat is immediately functional and ready to query the new repository.

---

## Epic 5: Language & Jargon Audit

**Goal:** Ensure all user-facing text in the onboarding flow is simple, clear, and free of technical jargon to provide an accessible experience for non-technical users.

### Stories

**Story 5.1: Review and Refine All UI Copy**
*As a* non-technical user,
*I want* to understand all the text, buttons, and instructions in the onboarding flow without feeling confused,
*So that* I can complete the setup process confidently.

**Acceptance Criteria:**
*   **Given** all UI components for the onboarding flow are built,
*   **When** a non-technical test user reviews the complete flow,
*   **Then** they report no points of confusion related to technical jargon (e.g., "repository," "commit," "main branch").
*   **And** all copy has been updated based on this feedback to use simpler analogues (e.g., "workspace," "save," "main version").
