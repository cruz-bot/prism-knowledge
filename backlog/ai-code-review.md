---
title: "AI-Powered Code Review"
type: canonical
category: backlog
strategic_role: Differentiating
priority: high
release: R3
journey_phases:
  - 3
---

# AI-Powered Code Review

## Summary

AI-powered code review brings automated, context-aware review feedback to every pull request in Nexus. Unlike simple linting or static analysis, this system understands team conventions, codebase patterns, and historical bug data to provide reviews that feel like they come from a senior teammate.

## Problem Statement

Code review is a bottleneck in most development workflows:

- **Waiting time:** Average 24-hour delay from PR creation to first review
- **Inconsistency:** Review quality varies by reviewer availability and expertise
- **Toil:** 60% of review comments are about style/convention, not logic
- **Scaling pain:** As teams grow, review load doesn't distribute evenly

AI review doesn't replace human reviewers — it handles the repetitive checks instantly, freeing human reviewers to focus on architecture, design, and business logic.

## Requirements

### Must Have (R3)

- Automated review comments on every PR within 60 seconds
- Convention detection learned from team's historical commits
- Security vulnerability scanning (OWASP Top 10)
- Confidence scores on every suggestion (0.0–1.0)
- Human-readable explanations for every flagged issue
- One-click accept/dismiss for AI suggestions
- Opt-out at team and individual level

### Should Have (R3)

- Custom rule definitions in natural language ("We never use `any` type in TypeScript")
- Review summary for PR authors (what changed, risk areas, suggested reviewers)
- Learning from accepted/dismissed suggestions (feedback loop)
- Integration with existing CI checks

### Could Have (Future)

- Auto-fix for high-confidence suggestions
- Cross-repository convention analysis
- Review assignment optimization based on expertise matching

## Technical Design

### RAG Pipeline

The AI review system uses Retrieval-Augmented Generation to provide context-aware reviews:

1. **Diff extraction** — Parse PR diff into semantic chunks (functions, classes, blocks)
2. **Context retrieval** — Query pgvector for similar code patterns in the codebase
3. **Convention loading** — Retrieve team-specific conventions from the convention store
4. **Review generation** — LLM generates review comments with citations
5. **Post-processing** — Filter by confidence threshold, deduplicate, format for display

### Model Selection

We're evaluating multiple approaches:

| Approach | Latency | Cost | Quality |
|----------|---------|------|---------|
| Claude (Anthropic API) | ~10s | $0.02/review | High |
| GPT-4o (OpenAI API) | ~8s | $0.03/review | High |
| Fine-tuned open-source | ~3s | $0.005/review | Medium-High |
| Hybrid (open-source + Claude fallback) | ~5s | $0.01/review | High |

Initial launch will use the hybrid approach: open-source model for convention and style checks, Claude for complex logic and security analysis.

### Data Isolation

Strict tenant isolation is non-negotiable:

- Each organization's codebase embeddings stored in separate pgvector schemas
- No cross-tenant data in LLM context windows
- Option for enterprise customers to use their own LLM API keys
- All code data encrypted at rest with per-tenant keys

## Success Metrics

- **Speed:** First AI review within 60 seconds of PR creation
- **Acceptance rate:** > 40% of AI suggestions accepted by developers
- **Time saved:** Reduce human review time by 30%
- **Accuracy:** < 5% false positive rate for security findings

## Related Documents

- [R3 Intelligence Release](../roadmap/r3-intelligence.md)
- [Product Vision](../narratives/product-vision.md)
- [Architecture Overview](../narratives/architecture-overview.md)
