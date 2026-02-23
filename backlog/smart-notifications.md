---
title: "Smart Notifications"
type: canonical
category: backlog
strategic_role: Bet
priority: medium
release: R2
journey_phases:
  - 4
---

# Smart Notifications

## Summary

Smart notifications provide context-aware alerts that cut through the noise of modern development workflows. Instead of notifying developers about everything, the system scores each event's relevance to the specific developer and delivers only what matters, when it matters.

## Problem Statement

Notification fatigue is a real and measurable productivity killer:

- Developers receive an average of **73 notifications per day** across tools
- **62% are irrelevant** to their current work
- Each interruption costs **23 minutes** to regain deep focus (UC Irvine study)
- Teams that over-notify see **40% higher notification disable rates**, defeating the purpose entirely

The goal isn't fewer notifications — it's smarter ones.

## Requirements

### Must Have (R2)

- Relevance scoring engine that considers: recency of file interaction, code ownership, team role, current focus state
- Focus mode: batch and hold notifications during deep work sessions
- Delivery channel preferences: in-app, email digest, Slack, webhook
- @mention override: direct mentions always break through filters
- Notification center with read/unread state and grouping

### Should Have (R2)

- Smart digest: daily summary of activity organized by importance
- Quiet hours configuration per user
- Team-level notification policies (e.g., "all security findings are urgent")
- Snooze functionality with re-surface

### Could Have (Future)

- ML-based optimal delivery timing
- Cross-tool notification aggregation (Slack + GitHub + Nexus unified inbox)
- Notification analytics for team leads

## Relevance Scoring Model

Each notification event receives a relevance score (0.0–1.0) based on weighted factors:

```
score = w1 * file_recency       // How recently you edited the affected file
      + w2 * code_ownership     // CODEOWNERS or Git blame attribution  
      + w3 * team_proximity     // Same team, shared project, or org-wide
      + w4 * event_severity     // Security finding > comment > file change
      + w5 * sender_importance  // Tech lead's comment > bot notification
```

Default weights (tunable per user):

| Factor | Weight | Rationale |
|--------|--------|-----------|
| File recency | 0.30 | Most predictive of relevance |
| Code ownership | 0.25 | Strong signal for who cares |
| Event severity | 0.20 | Critical events always surface |
| Team proximity | 0.15 | Team context matters |
| Sender importance | 0.10 | Social signal |

**Threshold:** Score > 0.5 → immediate notification. Score 0.3–0.5 → batched. Score < 0.3 → digest only.

## Strategic Rationale

Smart notifications are classified as a **Bet** because:

- The relevance scoring model is unproven at scale
- Getting it wrong (too aggressive filtering) could cause developers to miss critical updates
- Getting it right would be a significant differentiator — no competitor does this well
- The feedback loop (users adjusting thresholds) needs careful UX design

## Success Metrics

- **Notification reduction:** 50% fewer interruptions vs. naive "notify everything" approach
- **Relevance perception:** > 70% of delivered notifications rated "relevant" by users
- **Focus time increase:** 30+ minutes of additional focus time per developer per day
- **Disable rate:** < 10% of users disable notifications entirely

## Related Documents

- [R2 Collaboration Release](../roadmap/r2-collaboration.md)
- [User Journey — Stage 4](../narratives/user-journey.md)
