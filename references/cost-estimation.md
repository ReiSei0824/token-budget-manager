# Cost Estimation Reference

Token pricing used for cost estimation. All figures are approximate — actual billing depends on contract terms, service tier, and promotions.

## Anthropic API Pricing (Standard Tier, May 2026)

| Token Type | Price per Million Tokens | Notes |
|-----------|-------------------------|-------|
| Input | $3.00 | Regular prompt input |
| Output | $15.00 | Generated completion |
| Cache Write | $3.75 | 25% premium over input |
| Cache Read | $0.30 | 90% discount from input |

## Cost Calculation Per Message

```
cost = (input_tokens × 3.00 + output_tokens × 15.00
        + cache_creation_tokens × 3.75 + cache_read_tokens × 0.30) ÷ 1,000,000
```

## Service Tier Handling

- `"standard"` → Use pricing above
- `"plus"` → Same pricing (no public differentiation yet)
- `null` / missing → Fall back to standard pricing

## Rounding

Display costs rounded to:
- **≥ $1.00**: 2 decimal places ($12.34)
- **$0.01–$0.99**: 3 decimal places ($0.456)
- **<$0.01**: Show as "< $0.01"

## Savings Estimates Per Pattern

| Pattern | Savings Formula | Rationale |
|---------|----------------|-----------|
| Context Inflation | `(turns - 50) × avg_input_per_turn × $3.00 / 1,000,000` | Trimming above turn 50 |
| Cache Thrashing | `cache_creation_tokens × 0.90 × $3.75 / 1,000,000` | Cache writes wasted if never read |
| Read:Write Ratio | `total_input × 0.30 × $3.00 / 1,000,000` | 30% input reduction achievable |
| Output Verbosity | `(avg_output - 300) × n_responses × $15.00 / 1,000,000` | Trim to healthy 300-token avg |
| Subagent Dominance | `subagent_tokens × 0.20 × avg_cost / 1,000,000` | ~20% overhead is orchestration |
| Session Fragmentation | `duplicate_file_tokens × $3.00 / 1,000,000` | Avoid re-reading same files |

## Important

Label ALL cost figures as "估算" (estimate) in reports. Pricing changes and the opacity of `service_tier` means these are directional, not exact.
