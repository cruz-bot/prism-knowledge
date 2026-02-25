# PRD: Workspace File Structure & Code Generation

**Epic:** Integration & Aha Moment  
**Issues:** CRU-73 (File Structure Display), CRU-74 (Generate Code from File)  
**Priority:** P2 | **Status:** Draft  
**Date:** 2026-02-22

---

## 1. Problem Statement

Prism users connect knowledge repositories but have no way to browse the raw file structure of their workspace. They see documents only through Prism's parsed graph (categories, types, views) — never the actual directory tree. This creates a disconnect between the source repo and the Prism experience.

Additionally, once a user can see their files, there's no way to use Prism's AI capabilities to generate new code/content based on a selected file's context. This is the "aha moment" — where users see Prism not just as a reader but as a creative tool.

## 2. Goals

1. **CRU-73:** Let users browse the actual file/folder structure of their connected source repository in a tree view
2. **CRU-74:** Let users select a file from that tree and generate new code/content using AI, with the file's content as context

## 3. User Stories

### CRU-73: File Structure Display

**US-73-1: View file tree**  
> As a Prism user, I want to see my workspace's file and folder structure in a tree view so I can understand how my repository is organized.

**Acceptance Criteria:**
- [ ] Tree view shows directories and files from the connected source
- [ ] Directories are expandable/collapsible
- [ ] Files show appropriate icons based on extension (`.md`, `.ts`, `.json`, etc.)
- [ ] Tree loads lazily (top-level first, children on expand) for large repos
- [ ] Empty directories are shown but visually distinguished
- [ ] Loading states shown while fetching directory contents
- [ ] Error states for inaccessible directories

**US-73-2: Navigate to file**  
> As a user, I want to click a file in the tree to view its contents so I can quickly access any document.

**Acceptance Criteria:**
- [ ] Clicking a file opens it in the existing document viewer (for known types)
- [ ] Raw file content shown for non-document files (code, config)
- [ ] File path displayed as breadcrumb above content
- [ ] Selected file highlighted in tree

**US-73-3: Search/filter tree**  
> As a user, I want to filter the file tree by name so I can quickly find files in large repos.

**Acceptance Criteria:**
- [ ] Filter input at top of tree panel
- [ ] Filters in real-time as user types
- [ ] Matching files shown with parent directories auto-expanded
- [ ] Clear filter button

### CRU-74: Generate Code from Selected File

**US-74-1: Generate from file context**  
> As a user, I want to select a file and ask Prism to generate new code or content based on that file so I can accelerate my work.

**Acceptance Criteria:**
- [ ] "Generate" action available when a file is selected in the tree
- [ ] User provides a prompt describing what to generate
- [ ] AI receives the selected file's content as context
- [ ] Generated output displayed in a preview panel with syntax highlighting
- [ ] User can copy generated content to clipboard
- [ ] Loading/streaming state during generation

**US-74-2: Choose generation type**  
> As a user, I want to choose what kind of content to generate (tests, documentation, similar file, refactor) so the output matches my intent.

**Acceptance Criteria:**
- [ ] Preset generation types: "Write tests", "Generate docs", "Create similar", "Refactor", "Custom prompt"
- [ ] Each preset pre-fills an appropriate system prompt
- [ ] Custom prompt allows freeform instructions
- [ ] Generation type affects output formatting

**US-74-3: Save generated content**  
> As a user, I want to save generated content as a new file in my workspace so I can commit it.

**Acceptance Criteria:**
- [ ] "Save as" action with file path input
- [ ] Path defaults to same directory as source file
- [ ] Validates path (no overwrite without confirmation)
- [ ] Saved via existing repo-ops commit flow
- [ ] Success toast with link to new file

## 4. Functional Requirements

### FR-1: File Tree API
- `GET /api/sources/{sourceId}/files?path=` — Returns directory listing for a given path
- Returns: `{ entries: [{ name, path, type: 'file'|'directory', size?, extension? }] }`
- Server reads from the cloned repo on disk (already available via git-operations)
- Path traversal protection (no `..` escapes)
- Respects source's configured `directories` filter

### FR-2: File Content API
- `GET /api/sources/{sourceId}/files/content?path=` — Returns raw file content
- Returns: `{ content: string, path: string, size: number, extension: string }`
- Size limit: 1MB max file content
- Binary files return metadata only (size, type) with `isBinary: true`

### FR-3: Code Generation API
- `POST /api/sources/{sourceId}/generate-from-file` — Generates content from file context
- Request: `{ filePath: string, prompt: string, generationType: string }`
- Uses existing LLM router (`src/lib/llm/router.ts`)
- Streams response for real-time display
- Traces generation via existing trace service

### FR-4: File Tree Component
- Recursive tree component with expand/collapse
- Virtual scrolling for repos with 1000+ files
- Keyboard navigation (arrow keys, enter to expand/select)
- Context menu on right-click (Generate, View, Copy path)

### FR-5: Generation Panel
- Split view: file tree (left) + generation panel (right)
- Prompt input with preset chips
- Streaming output with syntax highlighting
- Copy/Save actions on generated content

## 5. Non-Functional Requirements

- **Performance:** Tree should render < 200ms for directories with < 500 items
- **Security:** All file access scoped to source's cloned repo path; no filesystem escape
- **Accessibility:** Tree navigable via keyboard; ARIA tree role attributes
- **Mobile:** Tree collapses to full-width; generation panel stacks below

## 6. Out of Scope (v1)

- Multi-file selection for generation context
- Drag-and-drop file organization
- File editing in-browser (beyond generated content)
- Git diff view of file changes
- Real-time file watching / auto-refresh

## 7. Dependencies

- CRU-74 depends on CRU-73 (needs file tree to select files)
- Existing: `src/lib/git-operations.ts` (repo clone paths)
- Existing: `src/lib/source-service.ts` (source metadata, repo paths)
- Existing: `src/lib/llm/router.ts` (AI generation)
- Existing: `src/lib/repo-ops/service.ts` (commit/save flow)

## 8. Success Metrics

- **Adoption:** 60%+ of active users browse file tree within first week
- **Aha moment:** 30%+ of users who browse files try code generation
- **Retention:** Users who generate code return 2x more often
