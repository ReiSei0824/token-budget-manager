# Waste Pattern Detection Algorithms

Detailed detection logic for all 6 token waste patterns. Each pattern includes: algorithm (pseudocode), thresholds, edge cases, and false-positive avoidance.

---

## Pattern 1: 上下文膨胀 (Context Inflation)

### Detection Algorithm

```
1. Count turns: N_turns = count of user messages in session
2. Extract input_tokens for each user+assistant turn
3. Compute:
   - avg_input_top5 = mean of the 5 largest input_tokens values
   - max_input = maximum single-turn input_tokens
   - trend = check if input_tokens is growing over the last 10 turns
            (linear regression slope of input_tokens ~ turn_index for last 10 turns)
4. Classify:
   - If N_turns > 80 AND avg_input_top5 > 50000 → 严重
   - If N_turns > 50 → 中等
   - If N_turns > 30 → 轻微
   - If trend shows >20% growth over last 10 turns AND N_turns > 30 → 上调一级
```

### Edge Cases

- **Very short sessions (≤5 turns)**: Skip detection entirely
- **First turn has large input**: That's the system prompt loading — expected, exclude turn 1 from avg_input_top5
- **Intentional long session**: Debugging may need 80+ turns. Note session type before flagging.

### Savings Estimate

```
wasted_turns = max(0, N_turns - 50)
savings = wasted_turns × avg_input_per_turn × input_cost_per_token
```

---

## Pattern 2: 缓存颠簸 (Cache Thrashing)

### Detection Algorithm

```
1. For each assistant message with usage data:
   - cache_write = cache_creation_input_tokens (legacy) OR cache_creation.ephemeral_5m_input_tokens (new)
   - cache_read = cache_read_input_tokens
2. Count thrash events:
   - thrash = (cache_read == 0 AND cache_write > 0)
3. Compute:
   - total_cache_read = sum(cache_read)
   - total_cache_write = sum(cache_write)
   - cache_hit_rate = total_cache_read / (total_cache_read + total_cache_write) [if both > 0]
   - If both are 0: cache_hit_rate = "N/A (no caching used)"
4. Classify:
   - If cache_hit_rate < 0.20 OR N_thrash > 10 → 严重
   - If cache_hit_rate < 0.40 OR N_thrash > 5 → 中等
   - If cache_hit_rate < 0.60 → 轻微
```

### Edge Cases

- **Both cache fields zero**: "N/A" — the model/setup might not support caching. Not a defect.
- **Short session (< 5 turns)**: Skip — not enough turns to expect cache reuse.
- **5-minute TTL**: If session duration > 1 hour and inter-turn gaps > 5 min, low hit rate is expected. Note this in the finding rather than flagging as 严重.
- **Legacy vs new cache format**: Check both formats. Prefer the new format (`cache_creation.ephemeral_*`) if non-zero; fall back to legacy.

### Savings Estimate

```
savings = total_cache_write × 0.90 × cache_write_cost_per_token
```
(90% because cache write = 1.25× input cost; if never read, the premium is wasted)

---

## Pattern 3: 读写比失衡 (Excessive Read-to-Write)

### Detection Algorithm

```
1. total_input = sum of all input_tokens (deduplicated by message.id)
2. total_output = sum of all output_tokens (deduplicated by message.id)
3. If total_output == 0: ratio = INF, input_pct = 100%
4. Else:
   - ratio = total_input / total_output
   - input_pct = total_input / (total_input + total_output) × 100
5. Classify:
   - If input_pct > 99.5% OR ratio > 99:1 → 严重
   - If ratio > 50:1 → 中等
   - If ratio > 20:1 → 轻微
```

### Edge Cases

- **Zero output**: Possible in sessions that are all tool calls or errors. Note this separately ("no assistant output detected").
- **Code implementation sessions**: Naturally higher input (reading files). Adjust: healthy is < 60:1 for coding, < 20:1 for chat.
- **Single-turn sessions**: Read:write ratio is meaningless for 1-turn sessions.

### Savings Estimate

```
savings = total_input × 0.30 × input_cost_per_token
```
(Assumes 30% of input context could be trimmed without quality loss)

---

## Pattern 4: 输出冗余 (Output Verbosity)

### Detection Algorithm

```
1. For each assistant message with output_tokens > 0, record the value
2. Compute:
   - avg_output = mean(output_tokens values)
   - max_output = max(output_tokens values)
   - p90_output = 90th percentile
3. Count "verbose" responses: messages where output_tokens > 1000
4. Classify:
   - If max_output > 4000 → 严重 (single response > 4K output)
   - If avg_output > 1000 → 严重
   - If avg_output > 500 → 中等
   - If avg_output > 300 → 轻微
```

### Edgge Cases

- **Code generation sessions**: 2000+ token outputs may be normal (generating a full file). Use context: check if session involves Write/Edit tool calls.
- **Conversational sessions**: >300 tokens per response is unusual for back-and-forth Q&A.
- **System prompt echo**: Ignore the very first assistant message (may contain long system-level output).

### Savings Estimate

```
excess_per_response = max(0, avg_output - 300)
savings = excess_per_response × n_assistant_messages × output_cost_per_token
```

---

## Pattern 5: 子代理成本霸权 (Subagent Cost Dominance)

### Detection Algorithm

```
1. Discover subagent files:
   - Check directory: {sessionDir}/subagents/
   - For each agent-*.jsonl: sum input_tokens + output_tokens
   - Read agent-*.meta.json for agent type/description
2. Compute:
   - main_tokens = main_session_input + main_session_output
   - subagent_tokens = sum of all subagent input + output
   - subagent_ratio = subagent_tokens / (main_tokens + subagent_tokens)
   - n_subagents = count of subagent processes
3. Classify:
   - If subagent_ratio > 0.80 → 严重
   - If subagent_ratio > 0.50 → 中等
   - If subagent_ratio > 0.30 → 轻微
   - If n_subagents > 5 → 上调一级 (over-fragmentation signal)
```

### Edge Cases

- **No subagents directory**: Skip this pattern entirely. Note "无子代理使用".
- **Heavy parallel workloads**: Running tests in parallel via subagents is efficient, not waste. Check agent descriptions from .meta.json.
- **Explore agents**: These are typically research/reading operations — dominant subagent cost is expected and not necessarily wasteful.

### Savings Estimate

```
overhead_pct = 0.20  # ~20% of subagent cost is orchestration overhead
savings = subagent_tokens × overhead_pct × avg_token_cost
```

---

## Pattern 6: 会话碎片化 (Session Fragmentation)

### Detection Algorithm

```
[Only applicable to Mode C (recent sessions) and Mode D (project)]

1. For each session being analyzed, extract the first 3 user messages
2. Identify file paths mentioned in those messages (regex: /[\w./-]+\.[\w]{1,6}/)
3. Build a file→sessions mapping: {file_path: [session_ids]}
4. Compute:
   - duplicate_files = files appearing in 2+ sessions
   - fragmentation_score = len(duplicate_files) / len(all_files) [if all_files > 0]
   - session_overlap_rate = sessions_with_duplicates / total_sessions
5. Classify:
   - If session_overlap_rate > 0.50 → 严重
   - If session_overlap_rate > 0.30 → 中等
   - If session_overlap_rate > 0.10 → 轻微
```

### Edge Cases

- **Only 1 session analyzed**: Skip this pattern (requires multi-session analysis).
- **Same project files across sessions**: Reading package.json is cheap and expected. Weight by file size: large files (>5KB) re-read across sessions count more.
- **Temporal proximity**: Sessions within the same day reading the same files is expected (continuous work). Sessions days apart reading the same files is fragmentation.

### Savings Estimate

```
# Estimate cost of re-reading duplicate files across sessions
duplicate_cost = sum(file_size_in_tokens × n_duplicate_sessions × input_cost_per_token
                     for each duplicate file)
```
