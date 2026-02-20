
# Prism Codebase Standards

This document outlines the key technical standards, patterns, and conventions for the Prism codebase. It is based on an analysis of the existing code and should be used as a guide for all new development.

## 1. Tech Stack & Versions

Based on `package.json`, the primary technologies and their versions are:

- **Next.js:** `^14.2.0`
- **React:** `^18.3.0`
- **TypeScript:** `^5.3.3`
- **Prisma:** `^7.1.0` (Client) / `^7.2.0` (Adapter)
- **Tailwind CSS:** `^3.4.1`
- **NextAuth.js:** `^5.0.0-beta.30`
- **Zod:** `^3.22.4`
- **Framer Motion:** `^12.23.25`
- **Testing Libraries:**
    - **Jest:** `^29.7.0`
    - **React Testing Library:** `^14.1.2`
    - **Playwright:** `^1.57.0`
- **Storybook:** `^10.1.4`

## 2. TypeScript Conventions

-   **Strict Mode:** TypeScript is configured in strict mode (`"strict": true` in `tsconfig.json`), which enforces strong type-checking rules. This is non-negotiable.
-   **Import Style:** Absolute paths are preferred using the `@/*` alias defined in `tsconfig.json`. Relative imports (`../`) should only be used for modules within the same directory.
    -   **Example:** `import { logger } from '@/src/lib/logger';`
-   **Type Patterns:**
    -   **`interface` vs `type`:** `interface` is predominantly used for defining the shape of objects and props (`AssistantPanelProps`). `type` is used for union types, primitives, and more complex type compositions (`LLMProvider`, `AuthorizeResult`).
    -   **Zod Schemas:** Zod is used for runtime validation, especially for agent inputs/outputs (`BaseAgent`) and API request bodies. Schemas should be defined and exported alongside the types they validate.
-   **Common Utility Types:** The codebase makes use of standard TypeScript utility types (`Omit`, `Partial`, etc.). Custom utility types are not prevalent; the preference is to use built-in or library-provided types.

## 3. Next.js App Router Patterns

-   **Route Handler Structure:** API routes are structured using the App Router convention with `route.ts` files inside `app/api/...`. Functions are exported for each HTTP method (e.g., `export async function GET(...)`).
-   **Auth Enforcement:**
    -   **Primary Method:** `requireUserAsync(request)` from `src/lib/auth/requireUser.ts` is the standard for protecting API routes. It uses NextAuth.js v5 sessions.
    -   The result is checked for success before proceeding:
        ```typescript
        const authResult = await requireUserAsync(request);
        if (!authResult.success) {
            return authResult.response; // Returns 401
        }
        const { userId, tenantId } = authResult.user;
        ```
    -   **Legacy/Test Method:** `requireUser(request)` is a synchronous version for tests relying on the `X-User-Id` header. **It should not be used in new production code.**
-   **Error Response Shape:** Standardized error responses are sent using `NextResponse.json()`. A common shape is:
    -   **401 Unauthorized:** `{ error: 'Authentication required', code: 'UNAUTHORIZED', message: '...' }`
    -   **404 Not Found:** `{ error: 'Source not found' }`
    -   **500 Server Error:** `{ error: 'Failed to...' }`
-   **Server vs. Client Components:**
    -   Pages and layouts that fetch data are primarily Server Components (`app/console/page.tsx`).
    -   Interactive components requiring hooks (`useState`, `useEffect`) or event listeners are Client Components, marked with `'use client';` (`AssistantPanel.tsx`, `AskBar.tsx`).
-   **Route Parameter Typing:** Route parameters are typed using an `interface` that defines the `params` object, as seen in `app/api/sources/[sourceId]/route.ts`.

## 4. Component Patterns

-   **File Naming:** Components use **PascalCase** and `.tsx` extension (e.g., `AssistantPanel.tsx`).
-   **Component Export Style:** **Named exports** are standard (e.g., `export function Button(...)`). A default export is sometimes included for Next.js pages or Storybook.
-   **Props Typing:** Props are defined using an `interface` with the convention `ComponentNameProps` (e.g., `AssistantPanelProps`).
-   **Tailwind Usage:**
    -   Classes are applied directly in the `className` attribute.
    -   There is **no `cn()` utility** observed; classes are composed with template literals.
    -   Custom styles are defined in `tailwind.config.js`, including an extensive color palette, font sizes, and animations.
-   **Framer Motion:** Used for animations. Keyframes and animation utilities are defined in `tailwind.config.js` and applied as Tailwind classes (e.g., `animate-fade-in`).
-   **Storybook:** The project is configured for Storybook, implying a pattern of creating `.stories.tsx` files for components, although none were analyzed in this scope.
-   **Component Testing:** Component tests use React Testing Library, with mocks configured in `jest.setup.ts`.

## 5. Auth & Authorization Pattern

-   **`requireUser` / `requireUserAsync`:** These functions act as an authentication boundary. `requireUserAsync` is the modern, preferred function that checks for a NextAuth.js session. It returns a user context (`{ userId, tenantId }`) on success or a `NextResponse` with a 401 status on failure.
-   **`getSourceForUser`:** This function from `source-service.ts` is the correct way to fetch a data source. It is user-scoped, meaning it will only return a source if it exists **and** is owned by the provided `userId`. This prevents data leakage between users.
-   **Session Access (Server-Side):** `getServerSession()` from `src/lib/auth/server-session.ts` is used in Server Components to get the current user session.
-   **User Scoping Pattern:** **All database access must be scoped by `userId` and/or `tenantId`.** This is a critical security pattern. The `source-service.ts` provides a clear example of this, where every database query includes a `where` clause for `userId`.

## 6. Database / Prisma Pattern

-   **Prisma Client Instantiation:** A singleton instance of `PrismaClient` is created in `src/lib/db/client.ts`. This file handles the logic to prevent connection pool exhaustion during development hot-reloading. The client is exported as `prisma`.
    -   **Import:** `import { prisma } from '@/src/lib/db/client';`
-   **Transaction Pattern:** Transactions are not explicitly shown in the analyzed files, but the standard Prisma pattern (`prisma.$transaction([...])`) should be used for atomic operations.
-   **Error Handling:** Database operations are wrapped in `try...catch` blocks in API routes and services. Errors are logged and a generic 500 error response is returned to the client.
-   **Migration Workflow:** The `package.json` scripts define the Prisma migration workflow:
    -   `db:migrate`: `prisma migrate dev` (for development)
    -   `db:migrate:deploy`: `prisma migrate deploy` (for production)
    -   `db:generate`: `prisma generate` (after schema changes)

## 7. Agent Pattern (`BaseAgent`)

-   **Extension:** Agents extend the `BaseAgent` class from `agents/core/agent-base.ts`.
-   **Required Signatures:** Each agent must implement:
    -   `getInputSchema(): z.ZodSchema<TInput>`: Defines the Zod schema for input validation.
    -   `getOutputSchema(): z.ZodSchema<TOutput>`: Defines the Zod schema for output validation.
    -   `executeInternal(...)`: Contains the core logic of the agent.
    -   `selectMemoryContext(...)`: Determines which memory nodes to fetch.
-   **Trace Logging:** Traces are captured automatically by the `BaseAgent.execute` method. It builds a comprehensive `AgentTrace` object and logs it using the provided `traceLogger`.
-   **Input/Output Schema:** Zod schemas are mandatory for validating the inputs and outputs of every agent, ensuring type safety at runtime.
-   **Memory Adapter:** The `MemoryAdapter` is passed to `executeInternal`, providing a safe, scoped view of the `MemoryGraph` for the agent to use.

## 8. LLM Routing Pattern

-   **LLM Calls:** All LLM calls are made through the `routeRequest` or `routeRequestStreaming` functions in `src/lib/llm/router.ts`. Direct calls to provider SDKs are an anti-pattern.
-   **Model Selection:** The model is specified in the `LLMRequest` object. `BaseAgent` uses the model defined in its constructor config but can be overridden per call.
-   **Streaming vs. Non-streaming:** The router has separate functions for streaming (`routeRequestStreaming`) and non-streaming (`routeRequest`) calls. The `useAssistant` hook handles streaming on the client-side.
-   **Error Handling:** The router returns a discriminated union `LLMResult` which is either a success or a structured `LLMError` object, containing a `code`, `message`, and `retryable` flag.

## 9. Testing Conventions

-   **Test File Location:** Tests are co-located with the code they test in `__tests__` directories (e.g., `src/lib/auth/__tests__/requireUser.test.ts`).
-   **Mock Patterns:**
    -   Global mocks and polyfills are configured in `jest.setup.ts`.
    -   `jest.mock('next/navigation', ...)` is used to mock Next.js router hooks.
    -   `fetch` is globally mocked, but can be overridden per test.
-   **Auth Mocking:** In the test environment (`NODE_ENV=test`), auth is mocked by setting the `X-User-Id` header in test requests. The `requireUser` helper is designed to respect this header only in test mode.
-   **Jest Config:** While `jest.config.js` was not provided, `jest.setup.ts` indicates the use of `@testing-library/jest-dom` and `ts-jest`.

## 10. Error Handling

-   **API Error Response Shape:** Errors are returned as JSON with a `{ error: 'message' }` shape, often accompanied by a `code` and more detailed `message` for auth errors. Status codes (401, 404, 500) are used appropriately.
-   **Client-Side Error Handling:**
    -   The `useAssistant` hook exposes an `error` state, which is displayed in components like `AssistantPanel`.
    -   UI components provide retry mechanisms where applicable (e.g., the "Retry" button in `AssistantPanel`).
    -   Data fetching in components uses `try...catch` and sets an error state for display.
-   **Logging:** A central `logger` is used (`src/lib/logger.ts`), creating child instances for specific modules (e.g., `log = logger.child('[api:agents]')`). Errors are logged with context.

## 11. Key Anti-Patterns (Do NOT do these)

-   **Using `requireUser` in new code.** **DO NOT** use the synchronous `requireUser`. **ALWAYS** use `requireUserAsync` for proper session-based authentication in new code.
-   **Directly accessing the database without user scoping.** Every query that touches user data **MUST** have a `where` clause that includes `userId` and/or `tenantId`.
-   **Hardcoding a fallback user ID.** There should be no "default user" or any fallback mechanism for authentication. If a user is not authenticated, they should be denied access.
-   **Using `getPersistedSource()` (legacy pattern).** The function `getSourceForUser(userId, sourceId)` is the correct, user-scoped way to retrieve a source. Any older, unscoped functions are an anti-pattern and a security risk.

## 12. File Structure Rules

-   **New API Routes:** Go in `app/api/` following the App Router folder structure. (e.g., a route for `/api/users/[userId]/posts` would be `app/api/users/[userId]/posts/route.ts`).
-   **New Components:**
    -   **UI Primitives:** Reusable, unstyled, or lightly styled components go in `src/components/ui/` (e.g., `Button.tsx`, `Card.tsx`).
    -   **Feature Components:** Complex components tied to a specific feature go into a subdirectory in `src/components/` (e.g., `src/components/assistant/`, `src/components/source-home/`).
-   **New Lib Modules:** Utility functions and services go in `src/lib/`, often grouped by domain (e.g., `src/lib/auth/`, `src/lib/db/`, `src/lib/llm/`).
-   **New Agents:** Go in the `agents/` directory, typically grouped by category (e.g., `agents/analysis/`, `agents/generation/`). All agents must extend `BaseAgent`.
