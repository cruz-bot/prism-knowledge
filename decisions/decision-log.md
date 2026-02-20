---
title: Decision Log
date: 2026-02-20
---

# Decision Log

Architecture Decision Records (ADRs) and key product decisions for Prism.

## Format

Each decision follows this structure:
- **Date:** When decided
- **Decision:** What we chose
- **Context:** Why it mattered
- **Alternatives Considered:** What we rejected
- **Consequences:** What this implies
- **Status:** Active | Superseded | Deprecated

---

## D001: User-Owned Repos (Option B)

**Date:** 2026-02-19
**Decision:** When new users create a workspace via template, Prism creates the GitHub repository on the USER's own GitHub account, not on a Prism-managed organization.

**Context:** We had two options for onboarding non-GitHub users:
- **Option A (Prism-hosted):** Create repos in a Prism-managed GitHub org. Faster onboarding, higher friction to leave.
- **Option B (User-owned):** Guide users to create their own GitHub account, create repos on their account. More upfront friction, but aligns with mission.

**Alternatives Considered:**
- Option A was tempting for speed and simplicity, but contradicts the "open and durable" mission

**Rationale:** The "your data is yours forever" story is a genuine competitive moat against Notion. The onboarding friction is manageable if designed right. Option A makes Prism proprietary in the exact way we said we didn't want to be.

**Consequences:**
- Onboarding must guide GitHub account creation (adds ~30-60 seconds)
- We use GitHub Device Flow OAuth to get a token with `repo` scope
- Zero vendor lock-in — user can stop using Prism and keep their data
- Marketing story is stronger: "We're a layer, not a trap"

**Status:** Active

---

## D002: Private Workspace Default

**Date:** 2026-02-19
**Decision:** New workspaces created from templates are **private by default**.

**Context:** When a user selects a template and clicks "Create Workspace," should the repo be public or private?

**Rationale:** Company knowledge (vision, team docs, product plans) shouldn't be public by default. Users can always make it public later if they want, but defaulting to public would leak sensitive info.

**Alternatives Considered:** Public default (better for demos, worse for real use)

**Consequences:**
- API must pass `private: true` when creating GitHub repos
- UI should show a toggle for "Make this workspace public" but default to unchecked

**Status:** Active

---

## D003: Four Workspace Templates

**Date:** 2026-02-19
**Decision:** Launch with exactly four pre-built workspace templates:
1. 🏢 **Company OS** — values, decisions, team wiki, runbooks, processes
2. 📦 **Product Workspace** — PRDs, roadmap, releases, decision log
3. 📚 **Team Docs** — onboarding, how-we-work, runbooks, knowledge base
4. 🚀 **Startup OS** — lean version of all of the above

**Context:** Templates reduce onboarding friction. A blank repo is intimidating. We need enough variety to feel relevant, but not so many that choice becomes paralyzing.

**Rationale:** Four is enough to cover the main use cases without overwhelming. Each template has real, useful seed content (not placeholder text).

**Alternatives Considered:**
- Fewer (2-3) felt too limiting
- More (6+) felt like feature creep before MVP
- "Start from scratch" option is a post-MVP growth feature

**Consequences:**
- Template system is in `src/lib/workspace-templates/`
- Each template is a directory with Markdown files + metadata
- Templates must stay up-to-date (review quarterly)

**Status:** Active

---

## D004: GitHub Device Flow for Auth

**Date:** 2026-02-19
**Decision:** Use GitHub's Device Flow OAuth for authenticating users' GitHub accounts.

**Context:** We need a GitHub OAuth token with `repo` scope to create repos on the user's account. Device Flow is the recommended approach for apps where the user doesn't have direct access to a browser redirect.

**Rationale:** The existing codebase already has Device Flow implemented (`src/lib/github-auth.ts` + `GitHubAuthFlow.tsx` component). Reusing it is faster and proven.

**Alternatives Considered:**
- OAuth Web Flow (requires redirect handling, more complex)
- Personal Access Tokens (manual, bad UX)

**Consequences:**
- Onboarding wizard uses the existing `GitHubAuthFlow` component
- Token is stored temporarily for repo creation, then associated with the user's Prism account

**Status:** Active

---

## D005: Security Hardening as P0 Gate

**Date:** 2026-02-20
**Decision:** Epic 1 (Security Hardening) is a **hard prerequisite** for launching any part of the onboarding feature.

**Context:** Technical audit revealed three P0/SEV-1 security issues:
- `/api/agents/custom` has no authentication (publicly exploitable)
- In-memory trace store loses data on restart
- 7 legacy routes use global `source-persistence.ts` instead of user-scoped service

**Rationale:** Launching new onboarding features that handle user GitHub tokens and create repos while SEV-1 holes exist is unacceptable. Security gates development.

**Alternatives Considered:** Ship onboarding first, fix security later (rejected — this is how breaches happen)

**Consequences:**
- Epic 1 must be done and shipped before Epic 2-5 begin
- Linear epic dependencies must reflect this sequencing
- No shortcuts — 100% of security stories done before moving forward

**Status:** Active

---

## D006: No Git Jargon in User-Facing UI

**Date:** 2026-02-20
**Decision:** All user-facing copy must be free of developer jargon. No "repository", "commit", "branch", "merge", "clone", "pull request" in UI text, buttons, or error messages.

**Context:** Non-technical users (Persona: Brenda) are intimidated by Git terminology. The mission is "GitHub for everyone" — that requires speaking their language.

**Rationale:** The repo-ops authoring flow already uses mission language ("Save Draft", "Publish") and it works. Apply that everywhere.

**Consequences:**
- Epic 5 (Language Audit) is a dedicated cross-cutting story
- Copy must use analogies: "workspace" not "repository", "save" not "commit"
- Error messages must explain what went wrong in plain English
- QA must verify: grep for jargon before shipping

**Status:** Active

---

**Next Decision ID:** D007
