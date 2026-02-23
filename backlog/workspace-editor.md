---
title: "Workspace Editor"
type: canonical
category: backlog
strategic_role: Core
priority: high
release: R1
journey_phases:
  - 2
  - 3
---

# Workspace Editor

## Summary

The workspace editor is Nexus's primary interface — a browser-based code editor built on Monaco that provides a VS Code-comparable editing experience. It is the foundation upon which all other features (collaboration, AI review, analytics) are built.

## Problem Statement

Developers currently rely on local IDE installations that are:

- **Environment-dependent** — "Works on my machine" remains a daily frustration
- **Isolated** — No built-in way to share editing sessions
- **Setup-heavy** — New team members spend hours configuring their environment

A browser-based editor eliminates environment issues, enables seamless collaboration, and reduces onboarding friction from days to minutes.

## Requirements

### Must Have (R1)

- Monaco-based editor with syntax highlighting for 40+ languages
- Multi-tab editing with split-pane support
- Integrated file explorer with drag-and-drop
- Built-in terminal (WebSocket-backed PTY)
- Basic Git operations (commit, push, pull, branch)
- VS Code keybinding import
- Theme support (dark, light, high-contrast)
- File search (Cmd+P / Ctrl+P)

### Should Have (R1.1)

- Extension API for syntax extensions
- Vim and Emacs keybinding modes
- Multi-cursor editing
- Snippet support with team-shared snippet libraries
- File-level permissions integration

### Could Have (Future)

- Notebook interface (Jupyter-style) for data science workflows
- Visual diff viewer for merge conflicts
- Integrated debugging with breakpoints

## Technical Design

### Architecture

The editor runs as a React application embedding the Monaco Editor component. File operations go through the File Service API, while terminal sessions use direct WebSocket connections.

```typescript
// Editor initialization
const editor = monaco.editor.create(container, {
  language: detectLanguage(filePath),
  theme: userPreferences.theme,
  minimap: { enabled: true },
  fontSize: userPreferences.fontSize,
  wordWrap: 'on',
  formatOnSave: true,
});

// Terminal connection
const terminal = new Terminal({
  cols: 120,
  rows: 40,
  rendererType: 'webgl',
});
const ws = new WebSocket(`wss://${host}/terminal/${sessionId}`);
terminal.attach(ws);
```

### Performance Targets

| Metric | Target | Current |
|--------|--------|---------|
| Time to interactive | < 2s | 1.8s |
| Keystroke-to-render latency | < 50ms | 35ms |
| File open (< 1MB) | < 200ms | 150ms |
| File open (1-10MB) | < 1s | 800ms |
| Memory usage (10 tabs) | < 500MB | 420MB |

### Offline Support

The editor supports offline mode via a service worker that caches:

- Editor bundle and assets
- Recently accessed files (last 50)
- Git staging state

Changes made offline are queued and synced on reconnect. Conflicts are resolved using the same CRDT mechanism as real-time collaboration.

## Success Metrics

- **Adoption:** 80% of invited developers use the editor at least 3x/week
- **Satisfaction:** Editor NPS > 50
- **Performance:** P95 keystroke latency < 50ms sustained
- **Retention:** 70% monthly active retention after first month

## Dependencies

- Auth Service for session management and workspace access
- File Service for storage and Git operations
- CDN for editor bundle delivery (Monaco is ~5MB)

## Related Documents

- [R1 Foundation Release](../roadmap/r1-foundation.md)
- [User Journey — Stages 2 & 3](../narratives/user-journey.md)
- [Product Vision](../narratives/product-vision.md)
