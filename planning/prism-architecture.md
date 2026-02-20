# Technical Architecture: Frictionless Onboarding

**Project:** Prism - Frictionless Onboarding: GitHub for Everyone  
**Author:** BMAD Architect Agent  
**Date:** 2026-02-20  
**Version:** 1.0

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [System Overview](#system-overview)
3. [Architecture Principles](#architecture-principles)
4. [Component Architecture](#component-architecture)
5. [Data Architecture](#data-architecture)
6. [API Architecture](#api-architecture)
7. [Security Architecture](#security-architecture)
8. [GitHub Integration Architecture](#github-integration-architecture)
9. [Performance Architecture](#performance-architecture)
10. [Technology Decisions & ADRs](#technology-decisions--adrs)
11. [Implementation Roadmap](#implementation-roadmap)
12. [Risk Mitigation](#risk-mitigation)

---

## Executive Summary

This document defines the technical architecture for Prism's Frictionless Onboarding feature, which enables non-technical users to go from signup to their first AI-powered answer in under 2 minutes. The architecture prioritizes **security**, **user data ownership**, **performance**, and **maintainability** while implementing five key epics:

1. **Security Hardening (P0)**: Critical security fixes as a release gate
2. **Welcome & GitHub Setup**: User-facing onboarding wizard
3. **Workspace Template System**: Backend infrastructure for template-based repo creation
4. **Integration & Aha Moment**: End-to-end user experience
5. **Language & Jargon Audit**: Accessible, non-technical UX copy

**Key Architectural Decisions:**
- User-owned GitHub repositories (private by default)
- Reuse existing GitHub Device Flow OAuth
- NextAuth v5 for session management with encrypted JWT tokens
- Template-based repository creation via GitHub REST API
- Strict user/tenant isolation at the database layer
- Sub-2-minute onboarding via optimized API patterns

---

## System Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Browser                            │
│  ┌────────────────┐  ┌─────────────────┐  ┌──────────────────┐ │
│  │ Onboarding     │  │ Template        │  │ Workspace        │ │
│  │ Wizard (UI)    │  │ Picker (UI)     │  │ Console (UI)     │ │
│  └────────┬───────┘  └────────┬────────┘  └────────┬─────────┘ │
└───────────┼──────────────────┼────────────────────┼────────────┘
            │                  │                    │
            ▼                  ▼                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Next.js 14 App Router                        │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              App Pages & Server Components               │  │
│  │  /onboarding  │  /onboarding/setup  │  /console/[id]    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                  API Route Handlers                       │  │
│  │  /api/auth/*  │  /api/sources/create-from-template       │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────┬────────────────────┬───────────────────┬──────────┘
             │                    │                   │
             ▼                    ▼                   ▼
┌──────────────────┐  ┌────────────────────┐  ┌─────────────────┐
│  NextAuth v5     │  │  GitHub Service    │  │  Source Service │
│  (Auth Layer)    │  │  (API Client)      │  │  (Data Layer)   │
└────────┬─────────┘  └──────────┬─────────┘  └────────┬────────┘
         │                       │                      │
         │                       ▼                      │
         │            ┌──────────────────┐             │
         │            │  GitHub REST API │             │
         │            │  (External)      │             │
         │            └──────────────────┘             │
         │                                             │
         └─────────────────────┬───────────────────────┘
                               ▼
                    ┌─────────────────────┐
                    │  PostgreSQL + Prisma│
                    │  (User Data)        │
                    └─────────────────────┘
```

### Data Flow: Onboarding Journey

```
1. User Sign-up → NextAuth (Google OAuth)
2. Welcome Screen → Choose "Type B" path
3. GitHub Connection → Device Flow OAuth → Store encrypted token
4. Template Selection → Send templateId to API
5. Repository Creation → GitHub API (create repo + initial commit)
6. Source Record Creation → Prisma (link user to repo)
7. Redirect to Workspace → User sees "Ask Prism" empty state
8. First Question → AI-powered answer (< 2 minutes total)
```

---

## Architecture Principles

### 1. **Security First**
- All security fixes (Epic 1) MUST be complete before shipping onboarding
- No endpoint exposes data without authentication and user-scoping
- Sensitive tokens never exposed to client-side code

### 2. **User Data Ownership**
- Repositories created on user's GitHub account, not Prism organization
- Private by default to protect user data
- Users can export/access data directly via GitHub

### 3. **Performance by Design**
- Target: < 2 minutes from signup to first answer
- Optimize for the critical path (onboarding flow)
- Async operations where possible, synchronous where necessary for UX

### 4. **Fail Safely**
- Graceful degradation for GitHub API errors
- Retry logic with exponential backoff
- User-friendly error messages (no technical jargon)

### 5. **Standards Adherence**
- Follow existing Prism patterns (from `prism-standards.md`)
- Use `requireUserAsync` for auth, `getSourceForUser` for data access
- Maintain consistency with existing codebase

---

## Component Architecture

### Frontend Components

#### 1. **Onboarding Welcome Screen**
**Location:** `app/onboarding/page.tsx`  
**Type:** Server Component  
**Responsibility:** Bifurcated entry point for Type A vs Type B users

```tsx
// app/onboarding/page.tsx
export default async function OnboardingPage() {
  const session = await getServerSession();
  
  return (
    <div>
      <WelcomeHeader user={session.user} />
      <PathSelector>
        <PathOption
          type="A"
          title="Connect Existing Repository"
          description="I already have a GitHub repository"
          href="/console/connect"
        />
        <PathOption
          type="B"
          title="Get Started with a Template"
          description="I'm new to GitHub"
          href="/onboarding/setup"
        />
      </PathSelector>
    </div>
  );
}
```

**Key Props:**
- `session: Session` - Current user session (from NextAuth)

**Dependencies:**
- `getServerSession()` (NextAuth v5)
- `WelcomeHeader` component
- `PathSelector` component (new)

---

#### 2. **Setup Wizard**
**Location:** `app/onboarding/setup/page.tsx`  
**Type:** Client Component (interactive multi-step form)  
**Responsibility:** 3-step wizard for GitHub connection, template selection, workspace creation

```tsx
// app/onboarding/setup/page.tsx
'use client';

export default function SetupWizard() {
  const [step, setStep] = useState(1);
  const [githubConnected, setGithubConnected] = useState(false);
  const [selectedTemplate, setSelectedTemplate] = useState<string | null>(null);
  
  return (
    <WizardContainer currentStep={step} totalSteps={3}>
      {step === 1 && (
        <GitHubConnectionStep
          onSuccess={() => {
            setGithubConnected(true);
            setStep(2);
          }}
        />
      )}
      {step === 2 && (
        <TemplateSelectionStep
          onSelect={(templateId) => {
            setSelectedTemplate(templateId);
            setStep(3);
          }}
        />
      )}
      {step === 3 && selectedTemplate && (
        <WorkspaceCreationStep templateId={selectedTemplate} />
      )}
    </WizardContainer>
  );
}
```

**Sub-Components:**
- `GitHubConnectionStep`: Wraps existing `GitHubAuthFlow.tsx` component
- `TemplateSelectionStep`: Renders `TemplatePicker` component
- `WorkspaceCreationStep`: Calls API, shows loading state, redirects on success

---

#### 3. **GitHub Auth Flow** (Existing, Reused)
**Location:** `src/components/onboarding/GitHubAuthFlow.tsx`  
**Type:** Client Component  
**Responsibility:** GitHub Device Flow OAuth UI

**Modifications Required:**
- Add prop to customize callback URL (redirect to `/onboarding/setup?step=2` instead of default)
- Emit event on successful auth for parent wizard to consume

---

#### 4. **Template Picker**
**Location:** `src/components/onboarding/TemplatePicker.tsx`  
**Type:** Client Component  
**Responsibility:** Visual selection UI for workspace templates

```tsx
// src/components/onboarding/TemplatePicker.tsx
'use client';

interface TemplatePickerProps {
  onSelect: (templateId: string) => void;
}

export function TemplatePicker({ onSelect }: TemplatePickerProps) {
  const templates = [
    {
      id: 'company-os',
      name: 'Company OS',
      description: 'Everything your startup needs: team docs, processes, and playbooks',
      icon: '🏢',
      color: 'blue',
    },
    {
      id: 'product-workspace',
      name: 'Product Workspace',
      description: 'PRDs, roadmaps, and product strategy documents',
      icon: '🚀',
      color: 'purple',
    },
    {
      id: 'team-docs',
      name: 'Team Docs',
      description: 'Meeting notes, processes, and team knowledge base',
      icon: '📚',
      color: 'green',
    },
    {
      id: 'startup-os',
      name: 'Startup OS',
      description: 'Comprehensive startup playbook with OKRs, hiring, and fundraising',
      icon: '💡',
      color: 'orange',
    },
  ];

  return (
    <div className="grid grid-cols-2 gap-6">
      {templates.map((template) => (
        <TemplateCard
          key={template.id}
          template={template}
          onClick={() => onSelect(template.id)}
        />
      ))}
    </div>
  );
}
```

**Styling:**
- Cards with hover effects (Tailwind + Framer Motion animations)
- Large icons, clear typography (non-technical descriptions)

---

#### 5. **Workspace Creation Loading State**
**Location:** `src/components/onboarding/WorkspaceCreationStep.tsx`  
**Type:** Client Component  
**Responsibility:** API call, loading state, error handling, redirect

```tsx
// src/components/onboarding/WorkspaceCreationStep.tsx
'use client';

interface WorkspaceCreationStepProps {
  templateId: string;
}

export function WorkspaceCreationStep({ templateId }: WorkspaceCreationStepProps) {
  const router = useRouter();
  const [status, setStatus] = useState<'loading' | 'error' | 'success'>('loading');
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    async function createWorkspace() {
      try {
        const response = await fetch('/api/sources/create-from-template', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ templateId }),
        });

        if (!response.ok) {
          const errorData = await response.json();
          throw new Error(errorData.error || 'Failed to create workspace');
        }

        const { source } = await response.json();
        setStatus('success');
        
        // Redirect after brief success message
        setTimeout(() => {
          router.push(`/console/${source.id}?firstRun=true`);
        }, 1500);
      } catch (err) {
        setStatus('error');
        setError(err instanceof Error ? err.message : 'Unknown error');
      }
    }

    createWorkspace();
  }, [templateId, router]);

  if (status === 'loading') {
    return <LoadingIndicator message="Creating your workspace..." />;
  }

  if (status === 'error') {
    return <ErrorDisplay error={error} onRetry={() => setStatus('loading')} />;
  }

  return <SuccessMessage message="Workspace created! Redirecting..." />;
}
```

**Error Handling:**
- Network errors: "Connection issue. Please try again."
- GitHub API errors: "GitHub is temporarily unavailable. Please try again in a moment."
- Rate limit errors: "Too many requests. Please wait a moment and try again."

---

#### 6. **First Run Empty State**
**Location:** `src/components/console/FirstRunEmptyState.tsx`  
**Type:** Client Component  
**Responsibility:** Guide new users to ask their first question

```tsx
// src/components/console/FirstRunEmptyState.tsx
'use client';

export function FirstRunEmptyState({ onDismiss }: { onDismiss: () => void }) {
  return (
    <div className="flex flex-col items-center justify-center h-full">
      <h2 className="text-2xl font-bold mb-4">
        Welcome to your workspace! 🎉
      </h2>
      <p className="text-gray-600 mb-6 max-w-md text-center">
        Your knowledge base is ready. Try asking a question to see how Prism works.
      </p>
      <ExampleQuestions
        questions={[
          "What's in this workspace?",
          "Show me the team structure",
          "How do I add new documents?",
        ]}
        onQuestionClick={(question) => {
          // Trigger the ask bar with pre-filled question
          onDismiss();
          // ... trigger ask bar
        }}
      />
    </div>
  );
}
```

**Trigger:**
- Show when `?firstRun=true` query param is present
- Dismiss after first question is asked (store in localStorage)

---

### Backend Services

#### 1. **GitHub Service**
**Location:** `src/lib/services/github-service.ts`  
**Responsibility:** Centralized GitHub API client with error handling and rate limiting

```typescript
// src/lib/services/github-service.ts
import { Octokit } from '@octokit/rest';
import { z } from 'zod';

const GitHubRepoSchema = z.object({
  id: z.number(),
  name: z.string(),
  full_name: z.string(),
  html_url: z.string(),
  private: z.boolean(),
  default_branch: z.string(),
});

export type GitHubRepo = z.infer<typeof GitHubRepoSchema>;

export class GitHubService {
  private octokit: Octokit;

  constructor(accessToken: string) {
    this.octokit = new Octokit({ auth: accessToken });
  }

  /**
   * Create a new repository on the authenticated user's account
   */
  async createRepository(options: {
    name: string;
    description: string;
    isPrivate: boolean;
  }): Promise<GitHubRepo> {
    try {
      const { data } = await this.octokit.repos.createForAuthenticatedUser({
        name: options.name,
        description: options.description,
        private: options.isPrivate,
        auto_init: false, // We'll create the initial commit manually
      });

      return GitHubRepoSchema.parse(data);
    } catch (error) {
      throw this.handleGitHubError(error);
    }
  }

  /**
   * Create or update a file in a repository
   * Used for creating initial template files
   */
  async createOrUpdateFile(options: {
    owner: string;
    repo: string;
    path: string;
    content: string;
    message: string;
    branch?: string;
  }): Promise<void> {
    try {
      const contentBase64 = Buffer.from(options.content, 'utf-8').toString('base64');
      
      await this.octokit.repos.createOrUpdateFileContents({
        owner: options.owner,
        repo: options.repo,
        path: options.path,
        message: options.message,
        content: contentBase64,
        branch: options.branch || 'main',
      });
    } catch (error) {
      throw this.handleGitHubError(error);
    }
  }

  /**
   * Create multiple files in a repository using a single commit
   * More efficient than createOrUpdateFile for multiple files
   */
  async createInitialCommit(options: {
    owner: string;
    repo: string;
    files: Array<{ path: string; content: string }>;
    message: string;
  }): Promise<void> {
    try {
      // Create blobs for all files
      const blobs = await Promise.all(
        options.files.map(async (file) => {
          const { data: blob } = await this.octokit.git.createBlob({
            owner: options.owner,
            repo: options.repo,
            content: Buffer.from(file.content, 'utf-8').toString('base64'),
            encoding: 'base64',
          });
          return { path: file.path, sha: blob.sha };
        })
      );

      // Create tree
      const { data: tree } = await this.octokit.git.createTree({
        owner: options.owner,
        repo: options.repo,
        tree: blobs.map((blob) => ({
          path: blob.path,
          mode: '100644',
          type: 'blob',
          sha: blob.sha,
        })),
      });

      // Create commit
      const { data: commit } = await this.octokit.git.createCommit({
        owner: options.owner,
        repo: options.repo,
        message: options.message,
        tree: tree.sha,
        parents: [], // No parents for initial commit
      });

      // Update main branch reference
      await this.octokit.git.createRef({
        owner: options.owner,
        repo: options.repo,
        ref: 'refs/heads/main',
        sha: commit.sha,
      });
    } catch (error) {
      throw this.handleGitHubError(error);
    }
  }

  /**
   * Get authenticated user info
   */
  async getAuthenticatedUser() {
    try {
      const { data } = await this.octokit.users.getAuthenticated();
      return {
        id: data.id,
        login: data.login,
        name: data.name || data.login,
      };
    } catch (error) {
      throw this.handleGitHubError(error);
    }
  }

  /**
   * Centralized error handling for GitHub API
   */
  private handleGitHubError(error: unknown): Error {
    if (error && typeof error === 'object' && 'status' in error) {
      const status = (error as any).status;
      const message = (error as any).message || 'GitHub API error';

      switch (status) {
        case 401:
          return new Error('GitHub authentication failed. Please reconnect your account.');
        case 403:
          if (message.includes('rate limit')) {
            return new Error('GitHub rate limit exceeded. Please try again in a moment.');
          }
          return new Error('GitHub access denied. Please check your permissions.');
        case 404:
          return new Error('GitHub resource not found.');
        case 422:
          return new Error('Invalid GitHub request. The repository may already exist.');
        default:
          return new Error(`GitHub API error: ${message}`);
      }
    }

    return new Error('Unknown GitHub API error');
  }
}

/**
 * Factory function to create GitHubService from user session
 */
export async function createGitHubServiceFromSession(): Promise<GitHubService> {
  const session = await getServerSession();
  
  if (!session?.user?.githubAccessToken) {
    throw new Error('No GitHub access token found in session');
  }

  return new GitHubService(session.user.githubAccessToken);
}
```

**Key Design Decisions:**
- **Octokit SDK**: Use official GitHub SDK for type safety and maintainability
- **Zod validation**: Validate GitHub API responses for runtime type safety
- **Error mapping**: User-friendly error messages (no GitHub-specific jargon)
- **Rate limit handling**: Centralized error detection for rate limits
- **Efficient commit creation**: Use Git blob/tree API for multi-file commits

---

#### 2. **Template Service**
**Location:** `src/lib/services/template-service.ts`  
**Responsibility:** Manage workspace templates and their file contents

```typescript
// src/lib/services/template-service.ts
import { readdir, readFile } from 'fs/promises';
import path from 'path';
import { z } from 'zod';

const TemplateFileSchema = z.object({
  path: z.string(),
  content: z.string(),
});

const TemplateMetadataSchema = z.object({
  id: z.string(),
  name: z.string(),
  description: z.string(),
  repoNamePrefix: z.string(),
});

export type TemplateFile = z.infer<typeof TemplateFileSchema>;
export type TemplateMetadata = z.infer<typeof TemplateMetadataSchema>;

const TEMPLATES_DIR = path.join(process.cwd(), 'src/lib/workspace-templates');

const TEMPLATE_REGISTRY: Record<string, TemplateMetadata> = {
  'company-os': {
    id: 'company-os',
    name: 'Company OS',
    description: 'Everything your startup needs',
    repoNamePrefix: 'company-os',
  },
  'product-workspace': {
    id: 'product-workspace',
    name: 'Product Workspace',
    description: 'Product strategy and roadmaps',
    repoNamePrefix: 'product-workspace',
  },
  'team-docs': {
    id: 'team-docs',
    name: 'Team Docs',
    description: 'Team knowledge base',
    repoNamePrefix: 'team-docs',
  },
  'startup-os': {
    id: 'startup-os',
    name: 'Startup OS',
    description: 'Comprehensive startup playbook',
    repoNamePrefix: 'startup-os',
  },
};

export class TemplateService {
  /**
   * Get metadata for a template
   */
  static getTemplateMetadata(templateId: string): TemplateMetadata {
    const metadata = TEMPLATE_REGISTRY[templateId];
    if (!metadata) {
      throw new Error(`Template not found: ${templateId}`);
    }
    return metadata;
  }

  /**
   * Get all files for a template
   */
  static async getTemplateFiles(templateId: string): Promise<TemplateFile[]> {
    const templateDir = path.join(TEMPLATES_DIR, templateId);
    
    try {
      const files = await this.readDirectoryRecursive(templateDir);
      
      return Promise.all(
        files.map(async (filePath) => {
          const content = await readFile(filePath, 'utf-8');
          const relativePath = path.relative(templateDir, filePath);
          
          return TemplateFileSchema.parse({
            path: relativePath,
            content,
          });
        })
      );
    } catch (error) {
      throw new Error(`Failed to load template files: ${templateId}`);
    }
  }

  /**
   * Generate a unique repository name for a template
   */
  static generateRepoName(templateId: string): string {
    const metadata = this.getTemplateMetadata(templateId);
    const timestamp = Date.now().toString(36);
    return `${metadata.repoNamePrefix}-${timestamp}`;
  }

  /**
   * Recursively read all files in a directory
   */
  private static async readDirectoryRecursive(dir: string): Promise<string[]> {
    const entries = await readdir(dir, { withFileTypes: true });
    const files = await Promise.all(
      entries.map(async (entry) => {
        const fullPath = path.join(dir, entry.name);
        return entry.isDirectory()
          ? this.readDirectoryRecursive(fullPath)
          : fullPath;
      })
    );
    return files.flat();
  }
}
```

**Template Directory Structure:**
```
src/lib/workspace-templates/
├── company-os/
│   ├── README.md
│   ├── team/
│   │   ├── org-chart.md
│   │   └── processes.md
│   ├── product/
│   │   └── roadmap.md
│   └── operations/
│       └── okrs.md
├── product-workspace/
│   ├── README.md
│   ├── prds/
│   │   └── template.md
│   └── roadmap/
│       └── quarterly.md
├── team-docs/
│   ├── README.md
│   └── meeting-notes/
│       └── template.md
└── startup-os/
    ├── README.md
    ├── hiring/
    │   └── playbook.md
    └── fundraising/
        └── deck-outline.md
```

---

#### 3. **Source Service** (Enhanced)
**Location:** `src/lib/services/source-service.ts`  
**Responsibility:** Database operations for Source records

**New Function:**
```typescript
// src/lib/services/source-service.ts

/**
 * Create a new Source record for a template-generated repository
 */
export async function createSourceFromTemplate(
  userId: string,
  tenantId: string,
  options: {
    repoFullName: string;
    repoUrl: string;
    templateId: string;
    githubRepoId: number;
  }
): Promise<Source> {
  const source = await prisma.source.create({
    data: {
      userId,
      tenantId,
      type: 'github',
      repoFullName: options.repoFullName,
      repoUrl: options.repoUrl,
      metadata: {
        templateId: options.templateId,
        createdViaOnboarding: true,
        githubRepoId: options.githubRepoId,
      },
      status: 'active',
    },
  });

  return source;
}
```

---

## Data Architecture

### Database Schema Updates

**Existing `Source` Model (No Changes Required):**
```prisma
model Source {
  id            String   @id @default(cuid())
  userId        String
  tenantId      String
  type          String   // 'github'
  repoFullName  String   // e.g., "username/prism-company-os-abc123"
  repoUrl       String   // e.g., "https://github.com/username/prism-company-os-abc123"
  metadata      Json?    // Store templateId, githubRepoId, etc.
  status        String   // 'active', 'archived', etc.
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  user          User     @relation(fields: [userId], references: [id])
  tenant        Tenant   @relation(fields: [tenantId], references: [id])

  @@index([userId])
  @@index([tenantId])
  @@index([repoFullName])
}
```

**Metadata JSON Structure:**
```json
{
  "templateId": "company-os",
  "createdViaOnboarding": true,
  "githubRepoId": 123456789
}
```

---

**Existing `User` Model (Enhancement Required):**
```prisma
model User {
  id                String   @id @default(cuid())
  email             String   @unique
  name              String?
  githubLogin       String?  // GitHub username (from OAuth)
  githubId          Int?     // GitHub user ID (from OAuth)
  // ... other fields

  sources           Source[]
  
  @@index([githubId])
}
```

---

**NextAuth Session Enhancement:**

NextAuth v5 stores session data in encrypted JWT cookies. We need to extend the session to include the GitHub access token (server-side only).

```typescript
// src/lib/auth/config.ts (NextAuth config)
import NextAuth from 'next-auth';
import GitHubProvider from 'next-auth/providers/github';

export const { auth, handlers, signIn, signOut } = NextAuth({
  providers: [
    GitHubProvider({
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
      authorization: {
        params: {
          scope: 'repo user',
        },
      },
    }),
  ],
  session: {
    strategy: 'jwt',
    maxAge: 30 * 24 * 60 * 60, // 30 days
  },
  callbacks: {
    async jwt({ token, account, user }) {
      // On first sign-in, persist GitHub access token
      if (account?.provider === 'github') {
        token.githubAccessToken = account.access_token;
        token.githubLogin = user?.login;
        token.githubId = user?.id;
      }
      return token;
    },
    async session({ session, token }) {
      // Make GitHub access token available in session (server-side only)
      // This is encrypted in the JWT and NOT exposed to client
      session.user.githubAccessToken = token.githubAccessToken as string;
      session.user.githubLogin = token.githubLogin as string;
      session.user.githubId = token.githubId as number;
      return session;
    },
  },
});
```

**Security Note:**  
The GitHub access token is stored in the encrypted JWT session token and is **only accessible server-side**. It is never sent to the client or exposed in the `/api/auth/session` endpoint.

---

## API Architecture

### API Endpoint: `POST /api/sources/create-from-template`

**Location:** `app/api/sources/create-from-template/route.ts`  
**Method:** POST  
**Authentication:** Required (NextAuth session)  
**Rate Limit:** None (user-initiated action)

**Request Schema:**
```typescript
const CreateFromTemplateRequestSchema = z.object({
  templateId: z.enum(['company-os', 'product-workspace', 'team-docs', 'startup-os']),
});
```

**Response Schema (Success):**
```typescript
const CreateFromTemplateResponseSchema = z.object({
  source: z.object({
    id: z.string(),
    repoUrl: z.string(),
    repoFullName: z.string(),
  }),
});
```

**Response Schema (Error):**
```typescript
{
  error: string;       // User-friendly error message
  code: string;        // Error code (e.g., 'UNAUTHORIZED', 'GITHUB_API_ERROR')
  message?: string;    // Technical details (for debugging)
}
```

**Implementation:**
```typescript
// app/api/sources/create-from-template/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { requireUserAsync } from '@/src/lib/auth/requireUser';
import { GitHubService } from '@/src/lib/services/github-service';
import { TemplateService } from '@/src/lib/services/template-service';
import { createSourceFromTemplate } from '@/src/lib/services/source-service';
import { logger } from '@/src/lib/logger';
import { z } from 'zod';

const log = logger.child('[api:sources:create-from-template]');

const RequestSchema = z.object({
  templateId: z.enum(['company-os', 'product-workspace', 'team-docs', 'startup-os']),
});

export async function POST(request: NextRequest) {
  try {
    // Step 1: Authenticate user
    const authResult = await requireUserAsync(request);
    if (!authResult.success) {
      return authResult.response; // 401
    }
    const { userId, tenantId, githubAccessToken } = authResult.user;

    // Step 2: Validate request body
    const body = await request.json();
    const validation = RequestSchema.safeParse(body);
    
    if (!validation.success) {
      return NextResponse.json(
        { error: 'Invalid request', code: 'VALIDATION_ERROR' },
        { status: 400 }
      );
    }

    const { templateId } = validation.data;

    log.info(`Creating workspace from template: ${templateId} for user: ${userId}`);

    // Step 3: Initialize GitHub service
    const githubService = new GitHubService(githubAccessToken);
    
    // Get GitHub user info for repo owner
    const githubUser = await githubService.getAuthenticatedUser();

    // Step 4: Generate unique repo name
    const repoName = TemplateService.generateRepoName(templateId);
    const templateMetadata = TemplateService.getTemplateMetadata(templateId);

    // Step 5: Create repository on GitHub
    log.info(`Creating GitHub repository: ${repoName}`);
    const repo = await githubService.createRepository({
      name: repoName,
      description: templateMetadata.description,
      isPrivate: true, // CRITICAL: Always private by default
    });

    // Step 6: Load template files
    const templateFiles = await TemplateService.getTemplateFiles(templateId);

    // Step 7: Create initial commit with all template files
    log.info(`Creating initial commit with ${templateFiles.length} files`);
    await githubService.createInitialCommit({
      owner: githubUser.login,
      repo: repo.name,
      files: templateFiles,
      message: `Initial commit: ${templateMetadata.name} template`,
    });

    // Step 8: Create Source record in database
    log.info(`Creating Source record for repo: ${repo.full_name}`);
    const source = await createSourceFromTemplate(userId, tenantId, {
      repoFullName: repo.full_name,
      repoUrl: repo.html_url,
      templateId,
      githubRepoId: repo.id,
    });

    // Step 9: Success response
    log.info(`Workspace created successfully: ${source.id}`);
    return NextResponse.json(
      {
        source: {
          id: source.id,
          repoUrl: source.repoUrl,
          repoFullName: source.repoFullName,
        },
      },
      { status: 201 }
    );
  } catch (error) {
    log.error('Failed to create workspace from template', { error });

    // User-friendly error messages
    if (error instanceof Error) {
      return NextResponse.json(
        { error: error.message, code: 'CREATION_FAILED' },
        { status: 500 }
      );
    }

    return NextResponse.json(
      { error: 'Failed to create workspace', code: 'UNKNOWN_ERROR' },
      { status: 500 }
    );
  }
}
```

**Performance Optimization:**
- Target: P95 < 5 seconds
- Parallel operations where possible (file loading + GitHub user info)
- Use efficient Git blob/tree API for multi-file commits (single commit vs. multiple)
- Minimal database operations (single INSERT for Source record)

**Error Handling:**
- Authentication errors → 401 with clear message
- GitHub API errors → Mapped to user-friendly messages (see `GitHubService.handleGitHubError`)
- Template errors → 400 with validation details
- Unexpected errors → 500 with generic message (log full details server-side)

---

## Security Architecture

### Epic 1: Security Hardening (P0)

This epic addresses three critical security vulnerabilities that MUST be fixed before shipping the onboarding feature.

#### Issue 1.1: `/api/agents/custom` Authentication

**Current State:**  
The `/api/agents/custom` endpoint is **unprotected**, allowing unauthenticated access. This is a SEV-1 vulnerability.

**Fix:**
```typescript
// app/api/agents/custom/route.ts
import { requireUserAsync } from '@/src/lib/auth/requireUser';

export async function POST(request: NextRequest) {
  // Add authentication check
  const authResult = await requireUserAsync(request);
  if (!authResult.success) {
    return authResult.response; // 401
  }
  const { userId, tenantId } = authResult.user;

  // Proceed with existing logic, now user-scoped
  // ...
}
```

**Validation:**
- [ ] Unauthenticated requests return 401
- [ ] Authenticated requests succeed
- [ ] User can only access their own agent runs

---

#### Issue 1.2: Persistent Trace Store

**Current State:**  
Agent traces are stored in-memory (`Map`), which is volatile and does not survive server restarts.

**Fix:**  
Create a new `AgentTrace` Prisma model for persistent storage.

**Schema Addition:**
```prisma
// prisma/schema.prisma
model AgentTrace {
  id          String   @id @default(cuid())
  userId      String
  tenantId    String
  agentType   String   // e.g., 'custom', 'analysis', etc.
  input       Json     // Agent input
  output      Json?    // Agent output (nullable if failed)
  status      String   // 'running', 'success', 'error'
  startedAt   DateTime @default(now())
  completedAt DateTime?
  duration    Int?     // Duration in milliseconds
  error       String?  // Error message if failed
  metadata    Json?    // Additional context

  user        User     @relation(fields: [userId], references: [id])
  tenant      Tenant   @relation(fields: [tenantId], references: [id])

  @@index([userId])
  @@index([tenantId])
  @@index([agentType])
  @@index([status])
}
```

**Migration:**
```typescript
// src/lib/services/trace-service.ts
import { prisma } from '@/src/lib/db/client';

export async function createAgentTrace(data: {
  userId: string;
  tenantId: string;
  agentType: string;
  input: Record<string, unknown>;
}) {
  return prisma.agentTrace.create({
    data: {
      userId: data.userId,
      tenantId: data.tenantId,
      agentType: data.agentType,
      input: data.input,
      status: 'running',
    },
  });
}

export async function completeAgentTrace(
  traceId: string,
  result: {
    status: 'success' | 'error';
    output?: Record<string, unknown>;
    error?: string;
    duration: number;
  }
) {
  return prisma.agentTrace.update({
    where: { id: traceId },
    data: {
      status: result.status,
      output: result.output,
      error: result.error,
      completedAt: new Date(),
      duration: result.duration,
    },
  });
}

export async function getAgentTracesForUser(
  userId: string,
  options?: { limit?: number; agentType?: string }
) {
  return prisma.agentTrace.findMany({
    where: {
      userId,
      ...(options?.agentType && { agentType: options.agentType }),
    },
    orderBy: { startedAt: 'desc' },
    take: options?.limit || 50,
  });
}
```

**Integration with `BaseAgent`:**
```typescript
// agents/core/agent-base.ts
import { createAgentTrace, completeAgentTrace } from '@/src/lib/services/trace-service';

export abstract class BaseAgent<TInput, TOutput> {
  async execute(
    input: TInput,
    context: AgentContext
  ): Promise<{ success: true; output: TOutput } | { success: false; error: string }> {
    const startTime = Date.now();
    
    // Create persistent trace record
    const trace = await createAgentTrace({
      userId: context.userId,
      tenantId: context.tenantId,
      agentType: this.name,
      input: input as Record<string, unknown>,
    });

    try {
      const output = await this.executeInternal(input, context);
      const duration = Date.now() - startTime;

      // Update trace with success
      await completeAgentTrace(trace.id, {
        status: 'success',
        output: output as Record<string, unknown>,
        duration,
      });

      return { success: true, output };
    } catch (error) {
      const duration = Date.now() - startTime;

      // Update trace with error
      await completeAgentTrace(trace.id, {
        status: 'error',
        error: error instanceof Error ? error.message : 'Unknown error',
        duration,
      });

      return { success: false, error: error instanceof Error ? error.message : 'Unknown error' };
    }
  }
}
```

**Validation:**
- [ ] Traces persist across server restarts
- [ ] All agent executions create trace records
- [ ] Traces are user-scoped (userId/tenantId)

---

#### Issue 1.3: Legacy Persistence Routes

**Current State:**  
7 legacy API routes use an outdated persistence layer that bypasses Prisma.

**Identified Routes:**
1. `/api/legacy/sources` (GET, POST)
2. `/api/legacy/sources/[id]` (GET, PATCH, DELETE)
3. `/api/legacy/queries` (GET, POST)

**Fix:**  
Migrate each route to use Prisma-based services (`source-service.ts`, etc.).

**Example Migration:**
```typescript
// Before (app/api/legacy/sources/route.ts)
import { legacyPersistence } from '@/src/lib/legacy-persistence';

export async function GET(request: NextRequest) {
  const sources = await legacyPersistence.getSources(userId);
  // ...
}

// After (app/api/sources/route.ts)
import { getSourcesForUser } from '@/src/lib/services/source-service';
import { requireUserAsync } from '@/src/lib/auth/requireUser';

export async function GET(request: NextRequest) {
  const authResult = await requireUserAsync(request);
  if (!authResult.success) {
    return authResult.response;
  }

  const sources = await getSourcesForUser(authResult.user.userId);
  return NextResponse.json({ sources });
}
```

**Validation:**
- [ ] All 7 routes use Prisma-based services
- [ ] Legacy persistence code is deleted
- [ ] All routes have authentication checks
- [ ] All database queries are user-scoped

---

### OAuth Token Security

**Storage:**
- GitHub access tokens are stored in encrypted JWT session tokens (NextAuth v5)
- JWT encryption uses `AUTH_SECRET` environment variable (minimum 32 characters)
- Tokens are **server-side only** and never exposed to client

**Transmission:**
- All API calls use HTTPS (enforced in production)
- Cookies are `httpOnly`, `secure` (HTTPS only), `sameSite: 'lax'`

**Rotation:**
- GitHub tokens do not expire by default (user can revoke manually)
- For future enhancement: Implement refresh token flow (GitHub OAuth supports it)

---

### CSRF Protection

**Built-in Protection:**  
Next.js Server Actions automatically verify the `Origin` header matches the `Host` header, preventing CSRF attacks.

**Additional Measures:**
- All state-changing operations use POST/PATCH/DELETE (not GET)
- NextAuth v5 has built-in CSRF protection for auth routes

---

### Rate Limiting

**GitHub API Rate Limits:**
- Authenticated users: 5,000 requests/hour
- Our onboarding flow: ~10-15 requests per user (repo creation + file commits)
- Risk: Low for normal usage, but could be hit by malicious actors

**Mitigation:**
- Catch rate limit errors in `GitHubService.handleGitHubError`
- Return user-friendly error message: "Too many requests. Please try again in a moment."
- For future: Implement server-side rate limiting per user (e.g., max 5 workspace creations per hour)

---

## GitHub Integration Architecture

### GitHub Device Flow OAuth

**Existing Implementation:** `src/lib/github-auth.ts`, `GitHubAuthFlow.tsx`

**Flow:**
1. User clicks "Connect GitHub"
2. Frontend calls GitHub Device Flow endpoint (`POST https://github.com/login/device/code`)
3. User shown device code + verification URL
4. User navigates to URL, enters code, authorizes app
5. Frontend polls GitHub token endpoint until authorization complete
6. Access token stored in NextAuth session (encrypted JWT)

**No Changes Required:**  
The existing Device Flow implementation is secure and meets our needs. We will reuse it as-is.

---

### Repository Creation Strategy

**Decision:** Use GitHub REST API (not GraphQL)

**Rationale:**
- REST API is more stable and widely documented
- Octokit SDK provides excellent TypeScript support
- GraphQL `createCommitOnBranch` mutation is newer and less battle-tested
- REST API blob/tree approach is proven for multi-file commits

**Initial Commit Strategy:**

**Option A: Multiple API Calls (Simple, Inefficient)**
```typescript
for (const file of templateFiles) {
  await octokit.repos.createOrUpdateFileContents({
    owner, repo, path: file.path,
    content: base64(file.content),
    message: `Add ${file.path}`,
  });
}
```
- **Pros:** Simple, easy to debug
- **Cons:** Slow (N API calls for N files), creates N commits

**Option B: Git Blob/Tree API (Efficient, Complex)**
```typescript
// 1. Create blobs for all files
const blobs = await Promise.all(files.map(file => 
  octokit.git.createBlob({ owner, repo, content: base64(file.content) })
));

// 2. Create tree with all blobs
const tree = await octokit.git.createTree({
  owner, repo,
  tree: blobs.map((blob, i) => ({
    path: files[i].path,
    mode: '100644',
    type: 'blob',
    sha: blob.sha,
  })),
});

// 3. Create commit
const commit = await octokit.git.createCommit({
  owner, repo,
  message: 'Initial commit',
  tree: tree.sha,
  parents: [], // No parents for initial commit
});

// 4. Create main branch reference
await octokit.git.createRef({
  owner, repo,
  ref: 'refs/heads/main',
  sha: commit.sha,
});
```
- **Pros:** Single commit, efficient (4 API calls regardless of file count)
- **Cons:** More complex, harder to debug

**Chosen Strategy: Option B (Git Blob/Tree API)**

**Justification:**
- Performance is critical for <2min onboarding target
- Template sizes: ~5-20 files per template
- Option A would take 5-20 API calls vs. 4 for Option B
- Cleaner Git history (1 commit instead of N)

---

### GitHub API Error Handling

**Error Categories:**

| Status Code | Meaning | User Message | Retry Strategy |
|-------------|---------|--------------|----------------|
| 401 | Unauthorized | "GitHub authentication failed. Please reconnect." | No retry, re-auth required |
| 403 (rate limit) | Rate limit exceeded | "Too many requests. Please try again in a moment." | Retry after delay (use `retry-after` header) |
| 403 (other) | Permission denied | "GitHub access denied. Check permissions." | No retry |
| 404 | Not found | "GitHub resource not found." | No retry |
| 422 | Validation error | "Repository name already exists. Please try again." | Retry with new repo name |
| 500+ | Server error | "GitHub is temporarily unavailable. Please try again." | Retry with exponential backoff |

**Retry Logic:**
```typescript
async function retryWithExponentialBackoff<T>(
  fn: () => Promise<T>,
  maxRetries = 3,
  baseDelay = 1000
): Promise<T> {
  let lastError: Error | undefined;

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error instanceof Error ? error : new Error('Unknown error');

      // Don't retry on client errors (4xx except 429)
      if (error && typeof error === 'object' && 'status' in error) {
        const status = (error as any).status;
        if (status >= 400 && status < 500 && status !== 429) {
          throw lastError;
        }
      }

      // Wait before retry (exponential backoff)
      const delay = baseDelay * Math.pow(2, attempt);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }

  throw lastError;
}
```

---

## Performance Architecture

### Performance Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| Time to First Answer | < 2 minutes | E2E user flow tracking |
| API Latency (P95) | < 5 seconds | `/api/sources/create-from-template` |
| GitHub API Success Rate | > 99.5% | Error tracking & monitoring |
| Template Load Time | < 500ms | File system operations |
| Database Query Time | < 100ms | Prisma query logging |

---

### Optimization Strategies

#### 1. **Parallel Operations**

**Sequential (Slow):**
```typescript
const githubUser = await githubService.getAuthenticatedUser(); // 200ms
const templateFiles = await TemplateService.getTemplateFiles(templateId); // 300ms
const repo = await githubService.createRepository(...); // 1000ms
// Total: 1500ms
```

**Parallel (Fast):**
```typescript
const [githubUser, templateFiles] = await Promise.all([
  githubService.getAuthenticatedUser(), // 200ms
  TemplateService.getTemplateFiles(templateId), // 300ms
]);
const repo = await githubService.createRepository(...); // 1000ms
// Total: 1300ms (200ms saved)
```

---

#### 2. **Efficient Git Commit Strategy**

**Key Decision:** Use Git blob/tree API for single-commit file uploads

**Performance Impact:**
- **Option A (N commits):** ~500ms per file × 20 files = **10 seconds**
- **Option B (1 commit):** ~500ms (blob creation) + ~1000ms (tree + commit + ref) = **1.5 seconds**
- **Savings: 8.5 seconds** ⚡

---

#### 3. **Template File Caching**

**Current:** Read template files from disk on every request

**Optimization:** Cache template files in memory after first load

```typescript
// src/lib/services/template-service.ts
const templateCache = new Map<string, TemplateFile[]>();

export class TemplateService {
  static async getTemplateFiles(templateId: string): Promise<TemplateFile[]> {
    // Check cache first
    if (templateCache.has(templateId)) {
      return templateCache.get(templateId)!;
    }

    // Load from disk
    const files = await this.loadTemplateFilesFromDisk(templateId);
    
    // Cache for future requests
    templateCache.set(templateId, files);
    
    return files;
  }
}
```

**Performance Impact:**
- **First request:** 300ms (disk read)
- **Subsequent requests:** <1ms (memory read)
- **Savings: 299ms per request after first** ⚡

**Trade-off:** Memory usage (~100KB per template × 4 templates = ~400KB)  
**Verdict:** Acceptable trade-off for production deployment

---

#### 4. **Database Query Optimization**

**User-Scoped Indexes:**
```prisma
model Source {
  // ... fields
  @@index([userId])
  @@index([tenantId])
  @@index([repoFullName])
}

model AgentTrace {
  // ... fields
  @@index([userId])
  @@index([tenantId])
  @@index([agentType])
  @@index([status])
}
```

**Query Performance:**
- Without index: O(N) table scan (~1000ms for 100k records)
- With index: O(log N) index lookup (~10ms for 100k records)

---

#### 5. **Frontend Optimizations**

**Code Splitting:**
```tsx
// Lazy load heavy components
const TemplatePicker = dynamic(() => import('@/src/components/onboarding/TemplatePicker'), {
  loading: () => <LoadingSpinner />,
});
```

**Prefetching:**
```tsx
// Prefetch workspace page during template selection
useEffect(() => {
  if (selectedTemplate) {
    router.prefetch(`/console/workspace-preview`);
  }
}, [selectedTemplate]);
```

---

### Performance Monitoring

**Instrumentation Points:**
1. API endpoint timing (built into Next.js via `performance.now()`)
2. GitHub API latency (log each request duration)
3. Database query timing (Prisma query events)
4. Template loading time (file system operations)

**Logging Example:**
```typescript
const startTime = performance.now();
const result = await githubService.createRepository(...);
const duration = performance.now() - startTime;
log.info('GitHub API call completed', { operation: 'createRepository', duration });
```

---

## Technology Decisions & ADRs

### ADR-001: Use GitHub REST API (Not GraphQL)

**Status:** Accepted  
**Date:** 2026-02-20

**Context:**  
GitHub offers both REST API and GraphQL API for repository operations. GraphQL has a newer `createCommitOnBranch` mutation that simplifies multi-file commits.

**Decision:**  
Use GitHub REST API with Git blob/tree/commit primitives.

**Rationale:**
- REST API is more mature and stable
- Octokit SDK provides excellent TypeScript support
- More documentation and community examples for REST API
- GraphQL `createCommitOnBranch` is newer (2021) and less battle-tested
- REST API gives us more control over commit structure

**Consequences:**
- More complex code for initial commit (blob → tree → commit → ref)
- Better long-term maintainability due to maturity
- Easier to debug (can inspect each API call)

---

### ADR-002: Private Repositories by Default

**Status:** Accepted  
**Date:** 2026-02-20

**Context:**  
Repositories can be created as public or private. User data ownership is a core principle, but we need to decide on default visibility.

**Decision:**  
All template-generated repositories MUST be created as private by default.

**Rationale:**
- **Security:** Users may add sensitive company data to templates
- **Privacy:** Non-technical users may not understand public vs. private
- **Reversibility:** Easy to make private → public, hard to reverse
- **Compliance:** Safer default for enterprise users (GDPR, etc.)

**Consequences:**
- Users must manually change to public if desired
- Aligns with "user data ownership" principle
- No risk of accidental data leakage

---

### ADR-003: Template File Caching Strategy

**Status:** Accepted  
**Date:** 2026-02-20

**Context:**  
Template files are loaded from disk on every workspace creation request. This adds latency (~300ms) to the critical path.

**Decision:**  
Cache template files in memory after first load. Clear cache on server restart only (no TTL).

**Rationale:**
- **Performance:** 300ms → <1ms per request (after first)
- **Memory:** ~400KB for all 4 templates (negligible)
- **Stale Data:** Templates rarely change in production; cache invalidation not needed

**Consequences:**
- Template changes require server restart to take effect
- For development: Add cache-busting logic (check file modification time)
- Production: Accept restart requirement for template updates

---

### ADR-004: GitHub Device Flow (Reuse Existing Implementation)

**Status:** Accepted  
**Date:** 2026-02-20

**Context:**  
The codebase has an existing GitHub Device Flow OAuth implementation. We could rebuild it or reuse it.

**Decision:**  
Reuse existing `GitHubAuthFlow.tsx` component and `github-auth.ts` library.

**Rationale:**
- **DRY Principle:** Don't repeat yourself
- **Battle-tested:** Already in production, proven to work
- **Consistency:** Maintains UX consistency with existing Type A flow

**Consequences:**
- Minor refactoring needed to customize callback URL for wizard
- Existing implementation is secure and follows best practices

---

### ADR-005: NextAuth v5 for Session Management

**Status:** Accepted  
**Date:** 2026-02-20

**Context:**  
The project uses NextAuth v5 beta. We need to decide if this is production-ready for the onboarding feature.

**Decision:**  
Continue using NextAuth v5 with JWT-based sessions.

**Rationale:**
- **Already integrated:** Migrating to another auth solution is too costly
- **Encryption:** JWT sessions are encrypted (JWE) and secure
- **Performance:** No database lookup required for session validation
- **Maturity:** Beta is stable enough for production (many companies use it)

**Consequences:**
- Monitor for breaking changes as NextAuth v5 moves to stable
- GitHub access token stored in encrypted JWT (server-side only)
- Future: Consider database sessions if JWT size becomes an issue

---

### ADR-006: Zod for Runtime Validation

**Status:** Accepted  
**Date:** 2026-02-20

**Context:**  
TypeScript provides compile-time type safety, but we need runtime validation for API requests and external data (GitHub API responses).

**Decision:**  
Use Zod for all runtime validation (API request bodies, GitHub API responses, template data).

**Rationale:**
- **Already adopted:** Codebase standard (see `prism-standards.md`)
- **Type inference:** Zod schemas generate TypeScript types automatically
- **Error messages:** User-friendly validation error messages
- **Consistency:** Follows existing patterns in the codebase

**Consequences:**
- All new API endpoints MUST define Zod request/response schemas
- GitHub API responses validated with Zod before use
- Template data validated for correctness

---

### ADR-007: Single-Commit Template Initialization

**Status:** Accepted  
**Date:** 2026-02-20

**Context:**  
We can create initial template files with either N commits (one per file) or 1 commit (all files).

**Decision:**  
Use single-commit approach via Git blob/tree/commit API.

**Rationale:**
- **Performance:** ~8.5 seconds faster for 20-file template
- **Clean history:** Single "Initial commit" instead of N commits
- **GitHub best practice:** Follows official Git workflow

**Consequences:**
- More complex implementation code
- Higher upfront API calls (4 calls regardless of file count)
- Better user experience (cleaner Git history)

---

## Implementation Roadmap

### Phase 1: Security Hardening (Epic 1) - **P0 GATE**

**Timeline:** Week 1  
**Owner:** Dev team  
**Status:** MUST COMPLETE BEFORE PHASE 2

**Tasks:**
- [ ] 1.1: Add authentication to `/api/agents/custom`
- [ ] 1.2: Create `AgentTrace` Prisma model + migration
- [ ] 1.3: Migrate 7 legacy routes to Prisma
- [ ] 1.4: Delete legacy persistence code
- [ ] 1.5: Write tests for all security fixes (85% coverage)

**Validation Criteria:**
- All API endpoints have authentication checks
- All database queries are user-scoped
- All agent traces persist across server restarts
- No regressions in existing functionality

---

### Phase 2: Backend Infrastructure (Epic 3)

**Timeline:** Week 2  
**Owner:** Backend dev  
**Dependencies:** Phase 1 complete

**Tasks:**
- [ ] 2.1: Create template directory structure (`src/lib/workspace-templates/`)
- [ ] 2.2: Populate template files (Company OS, Product Workspace, Team Docs, Startup OS)
- [ ] 2.3: Implement `TemplateService` with file loading + caching
- [ ] 2.4: Implement `GitHubService` with Octokit SDK
- [ ] 2.5: Implement `createSourceFromTemplate` in `source-service.ts`
- [ ] 2.6: Create `/api/sources/create-from-template` endpoint
- [ ] 2.7: Write tests for all services (85% coverage)

**Validation Criteria:**
- Template files load correctly from disk
- GitHub API integration works (create repo + initial commit)
- Source records created correctly in database
- API endpoint returns 201 on success, appropriate errors on failure

---

### Phase 3: Frontend Components (Epic 2 & 4)

**Timeline:** Week 3  
**Owner:** Frontend dev  
**Dependencies:** Phase 2 complete (for API integration testing)

**Tasks:**
- [ ] 3.1: Create `/onboarding` welcome page
- [ ] 3.2: Create `/onboarding/setup` wizard page
- [ ] 3.3: Build `TemplatePicker` component
- [ ] 3.4: Build `WorkspaceCreationStep` component (API integration)
- [ ] 3.5: Enhance `GitHubAuthFlow` for wizard integration
- [ ] 3.6: Create `FirstRunEmptyState` component
- [ ] 3.7: Update `/console/[id]` to show empty state on `?firstRun=true`
- [ ] 3.8: Write Storybook stories for all components
- [ ] 3.9: Write component tests (React Testing Library)

**Validation Criteria:**
- End-to-end flow works: welcome → GitHub auth → template picker → workspace creation → redirect
- Loading states display correctly
- Error states display with retry functionality
- Empty state guides user to first question

---

### Phase 4: Language & UX Polish (Epic 5)

**Timeline:** Week 4  
**Owner:** Product + Design  
**Dependencies:** Phase 3 complete

**Tasks:**
- [ ] 4.1: Audit all UI copy for jargon (use non-technical alternatives)
- [ ] 4.2: User testing with 5 non-technical users (record friction points)
- [ ] 4.3: Revise copy based on feedback
- [ ] 4.4: Design review for visual consistency
- [ ] 4.5: Accessibility audit (WCAG 2.1 AA compliance)
- [ ] 4.6: Final polish: animations, micro-interactions, etc.

**Validation Criteria:**
- Non-technical users report "easy to understand" (qualitative feedback)
- No confusing jargon identified in user testing
- WCAG 2.1 AA compliance (keyboard navigation, screen reader support)

---

### Phase 5: Testing & Launch Prep

**Timeline:** Week 5  
**Owner:** QA + DevOps  
**Dependencies:** Phases 1-4 complete

**Tasks:**
- [ ] 5.1: E2E testing with Playwright (full onboarding flow)
- [ ] 5.2: Performance testing (measure time to first answer)
- [ ] 5.3: Load testing (simulate 100 concurrent onboarding flows)
- [ ] 5.4: Security audit (OWASP Top 10 checks)
- [ ] 5.5: Monitoring setup (error tracking, performance dashboards)
- [ ] 5.6: Rollback plan (feature flag for onboarding flow)

**Validation Criteria:**
- E2E tests pass (100% success rate)
- Time to first answer < 2 minutes (P95)
- API latency < 5 seconds (P95)
- No security vulnerabilities identified
- Monitoring dashboards operational

---

### Phase 6: Staged Rollout

**Timeline:** Week 6  
**Owner:** Product + DevOps  
**Strategy:** Feature flag-based rollout

**Rollout Plan:**
1. **Internal beta (Day 1-2):** Enable for internal users only
2. **Alpha users (Day 3-5):** Enable for 10% of new signups
3. **Beta expansion (Day 6-10):** Enable for 50% of new signups
4. **Full launch (Day 11+):** Enable for 100% of new signups

**Monitoring:**
- Track onboarding completion rate (target: >60%)
- Track template adoption rate (target: >30%)
- Track Day-7 retention (target: >40%)
- Track time to first answer (target: <2 min)

**Rollback Criteria:**
- Onboarding completion rate drops below 40%
- Critical bugs reported (SEV-1 or SEV-2)
- GitHub API success rate drops below 95%

---

## Risk Mitigation

### Risk 1: GitHub API Rate Limiting

**Probability:** Medium  
**Impact:** High (blocks onboarding)  
**Mitigation:**
- Monitor rate limit headers on every GitHub API call
- Implement exponential backoff retry logic
- Show user-friendly error message if rate limit hit
- Alert engineering team if rate limit exhaustion detected

**Fallback:**  
If rate limits become a persistent issue, implement server-side queueing (defer repo creation to background job).

---

### Risk 2: GitHub Service Outage

**Probability:** Low  
**Impact:** High (blocks onboarding)  
**Mitigation:**
- Implement retry logic with exponential backoff (3 retries)
- Show clear error message: "GitHub is temporarily unavailable. Please try again in a moment."
- Monitor GitHub status page (https://www.githubstatus.com/) for incidents

**Fallback:**  
Display maintenance mode message if GitHub is down for extended period (>30 minutes).

---

### Risk 3: Template Files Corruption

**Probability:** Low  
**Impact:** Medium (users get broken templates)  
**Mitigation:**
- Validate template files on server startup (check all files exist and are readable)
- Write automated tests that verify template structure
- Version control templates (track changes in Git)

**Fallback:**  
If template loading fails, return graceful error: "Template temporarily unavailable. Please try again or contact support."

---

### Risk 4: Database Performance Degradation

**Probability:** Medium (as user base grows)  
**Impact:** Medium (slow onboarding)  
**Mitigation:**
- Add database indexes on `userId`, `tenantId`, `repoFullName`
- Monitor database query performance (Prisma query logging)
- Set up alerts for slow queries (>100ms)

**Fallback:**  
Scale database vertically (increase resources) or horizontally (read replicas).

---

### Risk 5: NextAuth v5 Breaking Changes

**Probability:** Low  
**Impact:** High (auth stops working)  
**Mitigation:**
- Pin NextAuth version in `package.json` (don't auto-upgrade)
- Monitor NextAuth release notes for breaking changes
- Test thoroughly before upgrading NextAuth versions

**Fallback:**  
If NextAuth v5 has critical issues, roll back to stable v4 (requires code changes).

---

### Risk 6: User Data Loss (Repo Deletion)

**Probability:** Low  
**Impact:** Critical (user loses all data)  
**Mitigation:**
- Educate users: "Your data lives on GitHub. Deleting your GitHub account will delete your data."
- Prism NEVER deletes user repos (only user can delete via GitHub)
- Soft-delete Source records (mark as archived, don't delete)

**Fallback:**  
If user accidentally deletes repo, provide instructions for GitHub support to recover (GitHub can restore deleted repos within 90 days).

---

## Appendix A: Testing Strategy

### Unit Tests (Jest + React Testing Library)

**Coverage Target:** 85%

**Test Suites:**
- `TemplateService.test.ts`: File loading, caching, name generation
- `GitHubService.test.ts`: API calls, error handling, retry logic
- `source-service.test.ts`: Database operations, user scoping
- `TemplatePicker.test.tsx`: Component rendering, user interactions
- `WorkspaceCreationStep.test.tsx`: API integration, loading states, error handling

---

### Integration Tests (Playwright)

**E2E Scenarios:**
1. **Happy Path:** User signs up → connects GitHub → selects template → creates workspace → asks first question
2. **GitHub Already Connected:** User signs up (with existing GitHub account) → selects template → creates workspace
3. **Error Handling:** GitHub API error → user sees error message → retries successfully
4. **Template Selection:** User changes template selection → different template content loaded

---

### Performance Tests (k6 or Artillery)

**Scenarios:**
1. **Load Test:** 100 concurrent onboarding flows (measure P95 latency)
2. **Stress Test:** Gradually increase load until failure (find breaking point)
3. **Soak Test:** Sustained load for 1 hour (detect memory leaks)

---

## Appendix B: Monitoring & Observability

### Key Metrics

**Product Metrics:**
- Onboarding completion rate (%)
- Template adoption rate (%)
- Time to first answer (seconds, P50/P95/P99)
- Day-7 retention rate (%)

**Technical Metrics:**
- API latency (`/api/sources/create-from-template`, P50/P95/P99)
- GitHub API success rate (%)
- Database query time (milliseconds, P95)
- Error rate (errors per 1000 requests)

**Infrastructure Metrics:**
- Server CPU usage (%)
- Server memory usage (%)
- Database connection pool utilization (%)

---

### Alerting Rules

**Critical Alerts:**
- Onboarding completion rate drops below 40% (alert immediately)
- API error rate exceeds 5% (alert immediately)
- GitHub API success rate drops below 95% (alert immediately)

**Warning Alerts:**
- API P95 latency exceeds 5 seconds (alert after 5 minutes)
- Database query P95 exceeds 100ms (alert after 10 minutes)
- Server CPU usage exceeds 80% (alert after 5 minutes)

---

## Appendix C: Future Enhancements (Post-MVP)

### V2 Features

1. **Organization Support:**  
   Allow users to create repos in GitHub organizations (not just personal accounts)

2. **Team Onboarding:**  
   Invite team members during onboarding flow (multi-user setup)

3. **Start from Scratch:**  
   Option to create blank workspace (no template)

4. **Custom Templates:**  
   Allow users to save their own templates for reuse

5. **Content Migration:**  
   Import existing docs from Notion, Google Docs, Confluence

---

### Performance Optimizations

1. **Background Repo Creation:**  
   Move repo creation to async job queue (user gets instant confirmation, repo created in background)

2. **Template CDN:**  
   Serve static template files from CDN instead of local disk

3. **GraphQL Exploration:**  
   Re-evaluate GraphQL `createCommitOnBranch` mutation as it matures

---

### Security Enhancements

1. **OAuth Token Refresh:**  
   Implement refresh token flow for GitHub (tokens currently don't expire)

2. **Rate Limiting (Server-Side):**  
   Limit workspace creation to N per user per hour (prevent abuse)

3. **Audit Logging:**  
   Log all workspace creation events for compliance

---

## Conclusion

This architecture document provides a comprehensive technical blueprint for implementing Prism's Frictionless Onboarding feature. The design prioritizes **security**, **performance**, and **user experience**, while adhering to existing codebase standards and patterns.

**Key Takeaways:**

1. **Security First:** Epic 1 (Security Hardening) is a **P0 gate** that MUST be complete before shipping.
2. **User Data Ownership:** Repositories are created on user's GitHub account, private by default.
3. **Performance by Design:** Target <2 minutes from signup to first answer through optimized API patterns.
4. **Standards Adherence:** Follow existing Prism patterns (`requireUserAsync`, `getSourceForUser`, Zod validation, etc.).
5. **Fail Safely:** Graceful error handling with user-friendly messages, retry logic, and monitoring.

**Next Steps:**

1. Review this architecture document with the team
2. Create Linear issues for all implementation tasks
3. Begin Phase 1 (Security Hardening) immediately
4. Schedule weekly check-ins to track progress against roadmap

---

**Document Version:** 1.0  
**Last Updated:** 2026-02-20  
**Author:** BMAD Architect Agent  
**Status:** Ready for Review  
