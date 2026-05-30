# Healthy Baselines Reference

Thresholds for interpreting token usage metrics. Derived from published research (Harness 2026 State of DevOps, LinearB 8.1M PR analysis, community studies).

## Baseline Table

| Metric | Healthy | Warning | Critical | Source |
|--------|---------|---------|----------|--------|
| Cache Hit Rate | > 60% | 30–60% | < 30% | Community benchmarks; 5-min TTL change |
| Read:Write Ratio | < 50:1 | 50:1–99:1 | > 99:1 | Typical coding sessions ~30:1 |
| Avg Turns/Session | < 30 | 30–50 | > 50 | Long sessions = 74% of all tokens |
| Avg Output/Response | < 300 | 300–500 | > 500 | Healthy: concise answers <300 tokens |
| Subagent Cost Share | < 30% | 30–50% | > 50% | Subagents as supplementary, not primary |
| Session Duplication Rate | < 10% | 10–30% | > 30% | Avoid re-reading same files across sessions |

## Cache TTL Context

Since March 2026, Anthropic reduced default cache TTL from 1 hour to **5 minutes**. Implications:

- Turns spaced > 5 min apart: cache is useless, cache_creation tokens are wasted
- Fast consecutive turns (< 5 min): cache should hit > 80%
- Sessions with long pauses between turns will naturally have lower hit rates — this is expected, not a defect

## Session Type Adjustments

Not all sessions have the same profile. Adjust expectations based on session purpose:

| Session Type | Expected Turns | Expected Read:Write | Notes |
|-------------|---------------|--------------------|-------|
| Code debugging | 20–40 | 30:1–60:1 | Lots of file reading expected |
| Code review (PR) | 10–20 | 20:1–40:1 | Focused on a diff |
| Research/exploration | 15–30 | 10:1–30:1 | More output (analysis) |
| Implementation | 30–60 | 15:1–40:1 | Alternating read and write |
| Chat/Q&A | 5–15 | 1:1–10:1 | Mostly conversation |

## When to Skip Baselines

- **Session < 5 turns**: Too short to assess patterns meaningfully
- **Session with no output tokens**: Tool-only or error session
- **Session < 2 minutes total**: Possible quick check, skip full scoring
