---
title: "Board Update — February 2025"
type: canonical
category: exec_briefing
persona: executive
briefing_date: "2025-02-15"
created_by: human
---

# Board Update — February 2025

## Purpose

Monthly board update covering product progress, go-to-market developments, and strategic decisions requiring board input.

## Product Update

### R1 Post-Launch Health

R1 has been in GA for two months. The platform is stable with 99.94% uptime in January. Key post-launch activities:

- **R1.1 patch** (shipped Jan 20): Fixed SAML edge cases with Okta and Azure AD, added offline mode MVP, improved editor startup time by 300ms
- **Security audit** (completed Feb 5): Third-party penetration test by NCC Group. 2 medium findings (both patched), 0 critical. Full report available in the data room.
- **SOC 2 Type II** (in progress): Audit period started January 1. Expected completion April 2025. Required for enterprise pipeline.

### R2 Collaboration — Beta Results

The real-time sync engine entered closed beta on February 1 with 20 teams (85 developers). Early results are extremely encouraging:

**Engagement metrics:**
- 78% of beta users tried multiplayer editing in their first week
- Average sync session duration: 47 minutes
- 23% of all editing sessions are now multiplayer (up from 0%)

**Qualitative feedback themes:**

1. **"This is magic"** — The real-time cursor presence and conflict-free editing consistently delights
2. **"Needs voice"** — 6 teams requested integrated voice chat for pair programming (added to R2 backlog)
3. **"Slow on large files"** — CRDT performance degrades on files > 5,000 lines. Optimization sprint planned for March.

**Beta NPS: 72** (vs. editor-only NPS of 56). Collaboration is our growth lever.

## Go-to-Market

### Pipeline

| Stage | Count | Total ACV |
|-------|-------|-----------|
| Prospect (qualified) | 12 | $180K |
| Demo completed | 6 | $96K |
| Evaluation/trial | 3 | $48K |
| Negotiation | 1 | $18K |

The evaluation-stage companies (a FinTech with 200 devs, a healthcare SaaS with 80 devs, and a gaming studio with 150 devs) are all waiting for R2 GA before committing. Collaboration features are the primary purchase driver.

### Competitive Intelligence

- **Cursor** raised $60M Series B (announced Feb 3). Focused on AI-first desktop experience. No collaboration features announced — confirms our strategic differentiation.
- **GitHub Codespaces** launched improved container build times but still no multiplayer editing.
- **Replit** continues to focus on education and hobbyist markets. No enterprise SSO or compliance.

Our positioning remains strong: **enterprise-grade collaboration** is a whitespace that no competitor is addressing directly.

## Financial Summary

| Metric | January | February (projected) |
|--------|---------|---------------------|
| MRR | $22.5K | $25.2K |
| New teams | 28 | 35 (projected) |
| Churn | 2 teams | 1 team |
| Net revenue retention | 112% | 115% |
| Burn rate | $195K | $200K |

MRR growth is accelerating. Net revenue retention above 110% indicates existing teams are expanding usage — a strong signal for product-market fit.

## Strategic Discussion: Series A Timing

We're approaching the decision point on Series A timing. Two options for board discussion:

**Option A: Raise in Q3 2025 (July–August)**
- Pros: R2 will be GA with 3-4 months of traction data. Stronger metrics for fundraise.
- Cons: Tighter runway (12 months remaining at raise time). Less negotiating leverage.
- Target: $8–10M at $40–50M valuation

**Option B: Raise in Q4 2025 (October–November)**
- Pros: R3 prototype ready, potentially 2-3 enterprise contracts closed. Stronger story.
- Cons: Only 8-9 months runway at raise time. Higher execution risk.
- Target: $10–12M at $60–80M valuation

**Recommendation:** Option A. The market is receptive to collaboration tools now, and waiting introduces unnecessary runway risk. Enterprise contracts in pipeline de-risk even if not yet closed.

## Action Items

- [ ] Board to vote on Series A timing (Option A vs B) — deadline March 1
- [ ] CFO to prepare financial model for both scenarios
- [ ] CEO to begin warm introductions with target VCs
- [ ] CTO to finalize R2 GA timeline with engineering team
