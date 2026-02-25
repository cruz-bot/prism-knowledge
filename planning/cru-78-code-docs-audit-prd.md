# PRD: CRU-78 — Code-level Documentation and Logging Audit

## Problem
Prism's codebase has inconsistent logging practices and gaps in JSDoc documentation:
- **6 files** use raw `console.log/error` instead of the structured `logger` from `src/lib/logger.ts`
- **4 exported functions** in `src/lib/` lack JSDoc comments
- Logger is well-designed but underutilized in auth and API modules

## Scope
### In Scope
1. Replace raw `console.log/warn/error` with `logger` calls in server-side code (`src/lib/`, `app/api/`)
2. Add missing JSDoc to exported functions in `src/lib/`
3. Leave client-side `console.*` (components) that are dev-guarded or where logger isn't available

### Out of Scope
- Client-side components using `console.warn` with `process.env.NODE_ENV` guards (Toast.tsx)
- `env-validation.ts` formatted console output (intentional dev UX)
- JSDoc comments inside code examples (not real missing docs)
- Refactoring the logger itself

## Files to Change
| File | Issue |
|---|---|
| `src/lib/auth/auth-config.ts` | 3× console.log → logger.info |
| `src/lib/auth/email-templates.ts` | console.error + console.log → logger |
| `src/lib/auth/rbac.ts` | console.error → logger.error |
| `app/api/admin/traces/route.ts` | console.error → logger.error |
| `src/lib/connections/connectors/github-connector.ts` | Missing JSDoc on 2 exports |
| `src/lib/connections/connectors/slack-connector.ts` | Missing JSDoc on 2 exports |
| `src/lib/connections/index.ts` | Missing JSDoc on 1 export |
| `src/lib/graph-summary.ts` | Missing JSDoc on 1 export |

## Success Criteria
- Zero raw `console.*` in server-side production code paths (excluding logger.ts itself)
- All exported functions in affected files have JSDoc
- `npx tsc --noEmit` passes
