# PRD: CRU-77 — UI Text and Copy Audit

## Overview
Audit all user-facing text in Prism's UI to ensure consistent tone, eliminate placeholder copy, fix capitalization issues, and improve clarity of CTAs and error messages.

## Problem
The codebase has accumulated inconsistencies in UI copy:
1. **`alert()` calls** used instead of toast notifications (PipelineBuilder.tsx)
2. **Inconsistent "Coming soon" labels** — mixed casing and phrasing across login, workspace switcher, onboarding setup, and views
3. **Placeholder/stub text** in onboarding step 3 ("Coming soon: Configure your workspace")
4. **Inconsistent error message phrasing** — some use "Failed to..." others use different patterns
5. **Minor punctuation inconsistencies** in placeholder text (trailing ellipsis vs. none)

## Scope
- Replace `alert()` calls with toast notifications
- Standardize "Coming soon" badge text to consistent casing
- Clean up placeholder stub text in onboarding
- Standardize error message patterns
- Ensure consistent placeholder text style (trailing ellipsis)

## Success Criteria
- Zero `alert()` calls in component code
- Consistent "Coming soon" casing across all instances
- All error messages follow pattern: "Unable to [action]. Please try again."
- No raw placeholder/stub text visible to users
- TypeScript compiles cleanly
