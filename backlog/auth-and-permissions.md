---
title: "Authentication & Permissions"
type: canonical
category: backlog
strategic_role: Core
priority: high
release: R1
journey_phases:
  - 1
---

# Authentication & Permissions

## Summary

Authentication and role-based access control (RBAC) are the gatekeepers of Nexus. Every user interaction starts with auth, and every resource access is governed by permissions. This system must be invisible when it works and crystal-clear when it restricts.

## Problem Statement

Enterprise development teams need:

- **SSO integration** — Developers shouldn't manage yet another password
- **Granular permissions** — Not everyone should see every project
- **Audit compliance** — Regulated industries require access logs
- **Self-service** — Team leads should manage their own team access without IT tickets

Most developer tools get auth right at the individual level but fail at team and organization levels. Nexus must be enterprise-ready from day one.

## Requirements

### Must Have (R1)

- OAuth 2.0 / OpenID Connect authentication
- SAML 2.0 SSO for enterprise identity providers (Okta, Azure AD, Google Workspace)
- Role-based access control with four roles: Owner, Admin, Member, Viewer
- Workspace-level and project-level permission scoping
- Session management (JWT with 15-min access / 7-day refresh tokens)
- Full audit trail for all authentication and authorization events
- Multi-factor authentication (TOTP-based)

### Should Have (R1.1)

- File-level permissions (restrict access to sensitive config files)
- Permission templates (pre-configured role sets for common team structures)
- API key management for CI/CD and automation
- Temporary access grants with automatic expiry

### Could Have (Future)

- Attribute-based access control (ABAC) for complex policies
- Cross-organization collaboration (guest access with scoped permissions)
- Hardware security key support (WebAuthn/FIDO2)

## Technical Design

### Auth Flow

```
User ──► Nexus Login Page ──► Identity Provider (SSO)
                                      │
                              ┌───────┴───────┐
                              │ Auth Service   │
                              │ ┌───────────┐  │
                              │ │ JWT Issuer │  │
                              │ └─────┬─────┘  │
                              │       │        │
                              │ ┌─────┴─────┐  │
                              │ │ OPA Policy │  │
                              │ │ Engine     │  │
                              │ └───────────┘  │
                              └────────────────┘
                                      │
                              Access Token (JWT)
                                      │
                              ┌───────┴───────┐
                              │ API Gateway   │
                              │ (validates)    │
                              └───────────────┘
```

### RBAC Model

Permissions are defined as policies evaluated by Open Policy Agent (OPA):

```rego
# Allow workspace access based on role
allow {
    input.action == "workspace:read"
    role := user_role[input.user_id][input.workspace_id]
    role in {"owner", "admin", "member", "viewer"}
}

# Restrict write access to appropriate roles
allow {
    input.action == "workspace:write"
    role := user_role[input.user_id][input.workspace_id]
    role in {"owner", "admin", "member"}
}
```

### Security Considerations

- **Token storage:** Access tokens in memory only (never localStorage); refresh tokens in httpOnly cookies
- **Rate limiting:** 5 failed login attempts → 15-minute lockout
- **Session revocation:** Immediate revocation propagated via Redis pub/sub
- **Secrets:** All secrets in HashiCorp Vault; never in environment variables or code

## Success Metrics

- **SSO setup time:** < 30 minutes for a new identity provider
- **Auth latency:** < 100ms for token validation (P99)
- **Zero breaches:** No unauthorized access incidents
- **Self-service rate:** 90% of permission changes handled by team leads without IT

## Related Documents

- [R1 Foundation Release](../roadmap/r1-foundation.md)
- [User Journey — Stage 1](../narratives/user-journey.md)
- [Architecture Overview](../narratives/architecture-overview.md)
