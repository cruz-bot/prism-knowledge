# CRU-75: Fetch File Content and Add to Generation Context

## Problem
When generating code (tests, docs, refactors) from a selected file, the LLM only sees the target file's content. It has no visibility into imported types, interfaces, or dependencies — leading to inaccurate or incomplete generated code.

## Solution
Resolve local imports from the selected file and include their content as additional context in the LLM prompt. This gives the model a richer understanding of types, interfaces, and dependencies.

## Scope
- Parse TypeScript/JavaScript import statements from the selected file
- Resolve local relative imports to actual file paths
- Read resolved files (skip node_modules, skip binary, cap total context size)
- Include resolved file contents in the generation prompt as "Related Files"
- Cap at ~5 related files or 50KB total to avoid token bloat

## Out of Scope
- Transitive dependency resolution (imports of imports)
- Package/node_modules content
- Dynamic imports

## Success Criteria
- Generation prompts include content of locally-imported files
- Total context is bounded to prevent token overflow
- Existing generation behavior unchanged when no imports found
- Tests cover import parsing and context assembly
