# Architecture: Workspace File Structure & Code Generation

**Issues:** CRU-73, CRU-74  
**Date:** 2026-02-22

---

## 1. System Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Console UI                            │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │  FileTree     │  │ FileViewer   │  │ GeneratePanel │  │
│  │  Component    │  │ Component    │  │ Component     │  │
│  └──────┬───────┘  └──────┬───────┘  └───────┬───────┘  │
└─────────┼──────────────────┼──────────────────┼──────────┘
          │                  │                  │
    GET /files?path=   GET /files/content  POST /generate-from-file
          │                  │                  │
┌─────────┼──────────────────┼──────────────────┼──────────┐
│         ▼                  ▼                  ▼          │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐   │
│  │ files route  │  │ content route│  │ generate route│   │
│  └──────┬──────┘  └──────┬───────┘  └───────┬───────┘   │
│         │                │                   │           │
│         ▼                ▼                   ▼           │
│  ┌─────────────────────────────┐    ┌───────────────┐   │
│  │   file-service.ts           │    │ LLM Router    │   │
│  │   (new: read cloned repo)  │    │ (existing)    │   │
│  └──────────┬─────────────────┘    └───────────────┘   │
│             │                                           │
│             ▼                                           │
│  ┌─────────────────────────────┐                        │
│  │  source-service.ts          │                        │
│  │  (existing: get repo path)  │                        │
│  └─────────────────────────────┘                        │
│                    API Layer                             │
└─────────────────────────────────────────────────────────┘
```

## 2. New Files & Components

### Backend (API + Services)

| File | Purpose |
|------|---------|
| `src/lib/files/file-service.ts` | Core service: list directory, read file, validate paths |
| `src/lib/files/types.ts` | Types: `FileEntry`, `DirectoryListing`, `FileContent` |
| `src/lib/files/index.ts` | Barrel export |
| `app/api/sources/[sourceId]/files/route.ts` | `GET` — list directory entries |
| `app/api/sources/[sourceId]/files/content/route.ts` | `GET` — read file content |
| `app/api/sources/[sourceId]/generate-from-file/route.ts` | `POST` — AI code generation |

### Frontend (Components)

| File | Purpose |
|------|---------|
| `src/components/workspace/FileTree.tsx` | Recursive tree with expand/collapse, icons, selection |
| `src/components/workspace/FileTreeNode.tsx` | Single node (file or directory) |
| `src/components/workspace/FileViewer.tsx` | Raw file content display with syntax highlighting |
| `src/components/workspace/GeneratePanel.tsx` | Prompt input, preset chips, streaming output |
| `src/components/workspace/WorkspaceExplorer.tsx` | Layout container: tree + viewer/generate split |
| `src/components/workspace/index.ts` | Barrel export |
| `src/hooks/useFileTree.ts` | Hook: fetch/cache directory listings |
| `src/hooks/useFileContent.ts` | Hook: fetch file content |
| `src/hooks/useCodeGeneration.ts` | Hook: manage generation state, streaming |
| `app/console/workspace/page.tsx` | New console page for workspace view |

## 3. Data Flow

### 3.1 File Tree Loading (CRU-73)

```
User navigates to /console/workspace
  → WorkspaceExplorer mounts
  → useFileTree("") fetches GET /api/sources/{id}/files?path=
  → file-service.ts reads fs.readdir(repoClonePath)
  → Returns [{ name, path, type, extension, size }]
  → FileTree renders top-level entries

User clicks directory
  → useFileTree("docs/") fetches GET /api/sources/{id}/files?path=docs/
  → Children loaded and cached
  → FileTreeNode expands with animation
```

### 3.2 File Viewing

```
User clicks file in tree
  → useFileContent("docs/readme.md") fetches GET /api/sources/{id}/files/content?path=docs/readme.md
  → file-service reads fs.readFile (with 1MB limit)
  → FileViewer renders with syntax highlighting (by extension)
```

### 3.3 Code Generation (CRU-74)

```
User selects file → clicks "Generate"
  → GeneratePanel opens
  → User picks preset or types custom prompt
  → POST /api/sources/{id}/generate-from-file
    body: { filePath, prompt, generationType }
  → Route reads file content via file-service
  → Builds LLM prompt: system + file content + user instruction
  → Streams response via existing LLM streaming infra
  → GeneratePanel displays streaming output
  → User clicks "Copy" or "Save as..."
  → Save triggers POST /api/repo-ops/save-draft (existing)
```

## 4. Component Design

### 4.1 FileTree Component

```tsx
interface FileTreeProps {
  sourceId: string;
  onFileSelect: (path: string) => void;
  selectedPath?: string;
  filter?: string;
}

// State per directory: 'collapsed' | 'loading' | 'expanded' | 'error'
// Cache: Map<dirPath, FileEntry[]> — survives re-renders via useRef
// Keyboard: ArrowUp/Down navigate, ArrowRight expand, ArrowLeft collapse, Enter select
// ARIA: role="tree", role="treeitem", aria-expanded
```

### 4.2 FileTreeNode Component

```tsx
interface FileTreeNodeProps {
  entry: FileEntry;
  depth: number;
  isSelected: boolean;
  onSelect: (path: string) => void;
  onToggle: (path: string) => void;
}

// Icons by extension:
// .md → DocumentIcon, .ts/.tsx → CodeIcon, .json → ConfigIcon
// .test.ts → TestIcon, directory → FolderIcon (open/closed)
// Indentation: depth * 16px padding-left
```

### 4.3 GeneratePanel Component

```tsx
interface GeneratePanelProps {
  sourceId: string;
  filePath: string;
  fileContent: string;
  onClose: () => void;
}

// Presets: [{label: "Write tests", prompt: "..."}, {label: "Generate docs", ...}]
// Output: streaming markdown with code blocks
// Actions: Copy to clipboard, Save as new file
```

### 4.4 WorkspaceExplorer Layout

```tsx
// Desktop: 300px sidebar (tree) + flex-1 main (viewer or generate panel)
// Mobile: full-width tree, tap file → full-width viewer, back button
// Tab bar: "Files" | "Generate" (when file selected)
```

## 5. API Specifications

### GET `/api/sources/[sourceId]/files`

**Query params:** `path` (string, default: `""` = root)

**Response 200:**
```json
{
  "entries": [
    { "name": "docs", "path": "docs", "type": "directory" },
    { "name": "README.md", "path": "README.md", "type": "file", "extension": ".md", "size": 1234 }
  ],
  "parentPath": null
}
```

**Security:**
- `requireUser()` — authenticated
- `sourceAccess()` — user owns source
- Path validation: resolve against repo root, reject `..` traversal
- Gitignore respect: skip `.git/`, `node_modules/`, configured ignores

### GET `/api/sources/[sourceId]/files/content`

**Query params:** `path` (string, required)

**Response 200:**
```json
{
  "content": "# README\n...",
  "path": "README.md",
  "size": 1234,
  "extension": ".md",
  "isBinary": false
}
```

**Limits:** 1MB max. Binary files: `isBinary: true`, no content.

### POST `/api/sources/[sourceId]/generate-from-file`

**Request body:**
```json
{
  "filePath": "src/lib/auth/rbac.ts",
  "prompt": "Write comprehensive unit tests for this module",
  "generationType": "tests"
}
```

**Response:** Streaming text (SSE or chunked), same pattern as `/api/assistant/chat`.

**Generation types & system prompts:**
| Type | System Prompt Summary |
|------|----------------------|
| `tests` | "Generate comprehensive unit tests for this code. Use the project's test patterns." |
| `docs` | "Generate documentation (JSDoc/README) for this code." |
| `similar` | "Generate a similar file following the same patterns for a new use case." |
| `refactor` | "Refactor this code for better readability, performance, or maintainability." |
| `custom` | User's prompt used directly. |

## 6. file-service.ts Design

```typescript
// src/lib/files/file-service.ts

import fs from 'fs/promises';
import path from 'path';

export interface FileEntry {
  name: string;
  path: string;        // relative to repo root
  type: 'file' | 'directory';
  extension?: string;   // e.g. ".ts"
  size?: number;        // bytes, files only
}

export interface DirectoryListing {
  entries: FileEntry[];
  parentPath: string | null;
}

export interface FileContent {
  content: string;
  path: string;
  size: number;
  extension: string;
  isBinary: boolean;
}

const IGNORED_DIRS = ['.git', 'node_modules', '.next', 'dist', '.cache'];
const MAX_FILE_SIZE = 1_048_576; // 1MB
const BINARY_EXTENSIONS = ['.png', '.jpg', '.gif', '.ico', '.woff', '.woff2', '.ttf', '.eot', '.zip', '.tar', '.gz'];

/**
 * List directory entries in the cloned repo.
 */
export async function listDirectory(repoPath: string, relativePath: string): Promise<DirectoryListing> {
  const safePath = validatePath(relativePath);
  const fullPath = path.join(repoPath, safePath);
  
  const dirents = await fs.readdir(fullPath, { withFileTypes: true });
  const entries: FileEntry[] = [];
  
  for (const dirent of dirents) {
    if (IGNORED_DIRS.includes(dirent.name)) continue;
    if (dirent.name.startsWith('.')) continue; // skip hidden files
    
    const entryPath = safePath ? `${safePath}/${dirent.name}` : dirent.name;
    
    if (dirent.isDirectory()) {
      entries.push({ name: dirent.name, path: entryPath, type: 'directory' });
    } else {
      const ext = path.extname(dirent.name);
      const stats = await fs.stat(path.join(fullPath, dirent.name));
      entries.push({ name: dirent.name, path: entryPath, type: 'file', extension: ext, size: stats.size });
    }
  }
  
  // Sort: directories first, then alphabetical
  entries.sort((a, b) => {
    if (a.type !== b.type) return a.type === 'directory' ? -1 : 1;
    return a.name.localeCompare(b.name);
  });
  
  return { entries, parentPath: safePath ? path.dirname(safePath) || null : null };
}

/**
 * Read file content from the cloned repo.
 */
export async function readFileContent(repoPath: string, relativePath: string): Promise<FileContent> {
  const safePath = validatePath(relativePath);
  const fullPath = path.join(repoPath, safePath);
  const ext = path.extname(safePath);
  const stats = await fs.stat(fullPath);
  
  if (stats.size > MAX_FILE_SIZE) {
    throw new Error(`File too large: ${stats.size} bytes (max ${MAX_FILE_SIZE})`);
  }
  
  const isBinary = BINARY_EXTENSIONS.includes(ext.toLowerCase());
  const content = isBinary ? '' : await fs.readFile(fullPath, 'utf-8');
  
  return { content, path: safePath, size: stats.size, extension: ext, isBinary };
}

/**
 * Validate and normalize a relative path. Prevents directory traversal.
 */
function validatePath(relativePath: string): string {
  const normalized = path.normalize(relativePath).replace(/\\/g, '/');
  if (normalized.startsWith('..') || normalized.includes('/../') || path.isAbsolute(normalized)) {
    throw new Error('Invalid path: directory traversal not allowed');
  }
  return normalized === '.' ? '' : normalized;
}
```

## 7. Console Page Route

New page at `app/console/workspace/page.tsx`:
- Added to console sidebar navigation in `ConsoleLayoutClient.tsx`
- Icon: `FolderTree` (lucide-react)
- Label: "Workspace"
- Position: after "Explorer", before "Dashboard"

## 8. Integration with Existing Systems

| System | Integration Point |
|--------|-------------------|
| **Source Service** | `getSourceById()` → get `repoClonePath` for file reads |
| **Auth** | `requireUser()` + source ownership check on every API call |
| **LLM Router** | `routeRequest()` for code generation, same as assistant/authoring |
| **Trace Service** | Log generation calls for audit trail |
| **Repo Ops** | "Save as" uses existing `save-draft` → `commit` flow |
| **Motion** | Tree expand/collapse animations via existing `src/lib/motion` |

## 9. Testing Strategy

| Layer | Approach |
|-------|----------|
| `file-service.ts` | Unit tests with temp directories; test path traversal rejection |
| API routes | Integration tests with mock source + temp repo |
| FileTree component | React Testing Library; test expand, select, filter, keyboard |
| GeneratePanel | Test preset selection, prompt submission, streaming display |
| E2E | Playwright: connect source → browse files → generate → save |

## 10. Migration / Rollout

- **Feature flag:** `workspace.fileTree` (default off in v1, on for dogfooding)
- **No database changes** — reads directly from cloned repos on disk
- **No breaking changes** — purely additive (new route, new components)
