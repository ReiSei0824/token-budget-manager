---
name: token-budget-manager
description: >
  Analyze Claude Code conversation transcripts (JSONL) to identify token waste
  patterns and provide optimization recommendations. Detect context inflation,
  cache thrashing, excessive read-to-write ratios, output verbosity, subagent
  cost dominance, and session fragmentation. Trigger on Chinese: "token管家",
  "token分析", "分析token使用", "token浪费", "对话token利用率", "预算管理",
  "token优化", "token检查", "token审计", "检查token消费", "快查token",
  "token快查". Also trigger on English: "token budget analysis",
  "analyze token usage", "check token waste", "optimize tokens", "token audit",
  "token health check", "quick token check", "token utilization".
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash(jq *)
  - Bash(ls *)
  - Bash(wc *)
  - Bash(head *)
  - Bash(tail *)
  - Bash(date *)
  - Bash(powershell *)
  - WebSearch
---

# Token管家 — Token Budget Manager

Analyses Claude Code JSONL transcripts to find token waste patterns and recommend optimizations. Complements (does not replace) the AI Code Reviewer — one checks code quality, this checks conversation efficiency.

**What this skill detects**: Context inflation (sessions running too long), cache thrashing (cache created but never read), excessive read-to-write ratios, output verbosity, subagent cost dominance, and session fragmentation.

**What this skill does NOT do**: Modify or delete transcripts, access Anthropic billing data (uses local estimates), read raw conversation content beyond what's needed for statistics.

---

## Input Handling

Determine which input mode the user wants from their query. Default to Mode A if ambiguous.

### Mode A: Current Session (default)

When the user asks about "当前对话" "这个会话" "this conversation" or gives no specific target:

1. Derive the sanitized project directory name. Take the current working directory, replace `:\` with `--` and remaining `\` with `-`. Example: `C:\Users\49337` → `C--Users-49337`.
2. Find the most recently modified `.jsonl` file: list files in `C:\Users\49337\.claude\projects\{sanitized-cwd}\*.jsonl` sorted by modification time.
3. Confirm with the user: "分析当前会话 [{sessionId前8位}]? (y/n)" — but if they said "快查" or "quick", skip confirmation.
4. Extract the session ID from the filename (GUID before `.jsonl`).

### Mode B: Specific Session by ID

When the user provides a session GUID or partial ID:

1. Search `C:\Users\49337\.claude\projects\*\*.jsonl` for the matching file.
2. If found in multiple projects, pick the most recently modified one.
3. If not found, report and list available sessions.

### Mode C: Recent Sessions (past N days)

When the user asks about "最近" "过去N天" "recent" "past N days":

1. Default N = 7. If the user specifies a number, use that.
2. List all `.jsonl` files under `C:\Users\49337\.claude\projects\`.
3. For each, check modification time. On Windows: `powershell (Get-Item '{path}').LastWriteTime`. On Unix: `date -r {path}`.
4. Filter to files modified within N days.
5. If > 10 sessions found, show a summary table and ask which to analyze. If ≤ 10, proceed with all.

### Mode D: Specific Project

When the user names a project or directory:

1. Map the project name to sanitized directory name under `C:\Users\49337\.claude\projects\`.
2. List all `.jsonl` files in that directory.
3. Proceed with all sessions found.

---

## Phase 1: Discover and Select Sessions

After determining the mode, execute discovery.

### Step 1.1: File discovery

Run Glob to find candidate `.jsonl` files. For Mode A, list only the current project directory. For Mode C/D, list across all project directories.

### Step 1.2: Quick validation

For each candidate file, read the first 3 lines to:
- Verify it's valid JSONL (each line starts with `{`)
- Extract `sessionId` from the first line that contains it
- Check for `ai-title` lines to get a human-readable session title

### Step 1.3: Session metadata

For each session, extract:
- **Session ID**: The GUID in the JSONL `sessionId` field
- **Title**: From `"type":"ai-title"` line, or first user prompt if no title
- **Turns**: Count of `"role":"user"` lines
- **File size**: From `ls` output
- **Last modified**: From file timestamp

Present the session list to the user. If only 1 session, auto-select it.

---

## Phase 2: Parse Usage Data

Read `<skill-dir>/references/cost-estimation.md` for pricing details.

### Step 2.1: Check jq availability

Run `jq --version`. If successful, use jq for all parsing (fast). If not, use fallback (Grep + targeted Read).

### Step 2.2: Extract usage records (jq path)

For each session JSONL file:

```
jq -c 'select(.message.usage != null) | {
  id: .message.id,
  input: .message.usage.input_tokens,
  output: .message.usage.output_tokens,
  cache_create: .message.usage.cache_creation_input_tokens,
  cache_read: .message.usage.cache_read_input_tokens,
  cache_5m: .message.usage.cache_creation.ephemeral_5m_input_tokens,
  cache_1h: .message.usage.cache_creation.ephemeral_1h_input_tokens,
  tier: .message.usage.service_tier,
  model: .message.model
}' {file}
```

Then deduplicate by `.id` — messages with the same `id` field (thinking + text splits) share the same usage object. Keep only the first occurrence of each ID.

### Step 2.3: Extract usage records (fallback — no jq)

```
1. Grep for '"input_tokens"' lines in the file. Count them with head_limit: 0.
2. For each matching line, extract the usage JSON object around it.
3. Parse: input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens.
4. Also extract message.id for deduplication.
5. Deduplicate by message.id.
```

### Step 2.4: Count turns

Search for `"type":"user"` lines to count user turns. Alternatively, count `turn_duration` system events.

### Step 2.5: Extract session title

Search for `"type":"ai-title"` lines. If found, extract the title. If not, use the first user message content as the title (truncated to 50 chars).

### Step 2.6: Aggregate statistics

For each session, compute:

| Metric | Formula |
|--------|---------|
| Total input | `sum(input_tokens)` after dedup |
| Total output | `sum(output_tokens)` after dedup |
| Total cache created | `sum(cache_create)` + `sum(cache_5m)` + `sum(cache_1h)` |
| Total cache read | `sum(cache_read)` |
| Cache hit rate | `cache_read / (cache_read + cache_created)` |
| Avg input/turn | `total_input / N_turns` |
| Avg output/response | `total_output / N_assistant_messages_with_output` |
| Estimated cost | Apply formula from cost-estimation.md |

### Step 2.7: Discover subagents

1. Check if directory `{sessionDir}/subagents/` exists.
2. For each `agent-*.jsonl` file: parse usage data using the same algorithm as Step 2.2/2.3.
3. Read `agent-*.meta.json` for agent type and description.
4. Aggregate subagent token counts separately, grouped by agent type if >1 subagent.

---

## Phase 3: Detect Waste Patterns

Read `<skill-dir>/references/waste-patterns.md` for the complete detection algorithms. Apply all 6 patterns.

### Pattern 1: Context Inflation (上下文膨胀)

Detect sessions with excessive turns or per-turn input growth.

- Extract turn count and input_tokens per turn
- Apply thresholds from waste-patterns.md
- Severity: 严重/中等/轻微

### Pattern 2: Cache Thrashing (缓存颠簸)

Detect when cache is created but never read.

- Compare `cache_read` vs `cache_creation` per message
- Count thrash events (creation > 0, read ≈ 0)
- Compute hit rate
- Severity based on hit rate: <20%=严重, <40%=中等, <60%=轻微

### Pattern 3: Read-to-Write Ratio (读写比失衡)

Detect when input vastly exceeds output.

- Compute `total_input / total_output`
- Severity based on ratio: >99:1=严重, >50:1=中等, >20:1=轻微

### Pattern 4: Output Verbosity (输出冗余)

Detect overly long assistant responses.

- Compute average and maximum output_tokens per response
- Severity: max>4000 or avg>1000=严重, avg>500=中等, avg>300=轻微

### Pattern 5: Subagent Cost Dominance (子代理成本霸权)

Detect when subagents consume disproportionate tokens.

- Compute `subagent_tokens / (main + subagent_tokens)`
- Severity: >80%=严重, >50%=中等, >30%=轻微
- Skip if no subagents found

### Pattern 6: Session Fragmentation (会话碎片化)

Detect overlapping file reads across sessions.

- Only applicable for Mode C/D (multi-session analysis)
- Extract file paths from user messages
- Compute overlap rate
- Severity: >50%=严重, >30%=中等, >10%=轻微
- Skip for single-session analysis

### Cross-pattern notes

- Patterns 1 and 2 often co-occur — long sessions naturally exceed the 5-min cache TTL
- Pattern 3 and 4 can be inversely correlated — some sessions are all-input (low output), others are all-output (verbose)
- Pattern 5 and 6 are structural — they indicate workflow issues, not just per-session behavior

---

## Phase 4: Generate Report

Read `<skill-dir>/references/output-template.md` for the exact report format.

### Health Score Calculation

```
Start at 100 points.
Deduct per finding:
  - 严重: -40 points
  - 中等: -20 points
  - 轻微: -10 points
Clamp to [0, 100].

Grade:
  A (优秀): >= 90
  B (良好): >= 70
  C (一般): >= 50
  D (较差): >= 30
  F (很差): < 30
```

### Report structure

1. **Executive Summary**: Total tokens, cost, cache hit rate, avg turns, health score
2. **Waste Breakdown**: Table of 6 patterns with severity, waste estimate, savings potential
3. **Waste Pattern Details**: Only for patterns with findings — data, why it matters, recommendation
4. **Per-Session Ranking**: Sort by cost descending. Show session ID, title, tokens, cost, health, top issue
5. **Optimization Tips**: Ranked by savings impact. Max 5 tips. Skip trivial ones.
6. **Healthy Baselines Comparison**: Side-by-side table with ✅/⚠️/❌ status

If ALL patterns return "无" (no issues found), output: "✅ 未发现 Token 浪费模式。你的使用习惯很好！"

---

## Phase 5: Quick Check (轻量速查)

Triggered when user says "快查" "quick" "利用率" "简单看看" or asks about a quick overview without full analysis.

### Quick Check Algorithm

1. Run Phase 1 Mode A to find the current session.
2. Extract only:
   - Turn count from `"type":"user"` line count
   - First and last 3 `message.usage` records for cache hit rate
   - Total input + output for read:write ratio
3. Compute 3 metrics:
   - **Turns**: count + healthy/warning label
   - **Cache hit rate**: % + healthy/warning label
   - **Read:Write ratio**: X:1 + healthy/warning label
4. Output the Quick Check card from output-template.md.

Do NOT run full Phase 3 (all 6 patterns) in Quick Check mode. This keeps it under ~5 seconds.

---

## Guardrails

1. **Do not read full JSONL content**: Use targeted extraction (jq or Grep). A 750KB file should result in <50 lines of parsed output, not the raw content.
2. **Non-destructive analysis only**: Never modify or delete JSONL files. This skill is entirely read-only.
3. **Privacy protection**: Do not output raw conversation text. Only aggregate statistics and token counts.
4. **Cost estimates are approximate**: Always label cost figures as "估算" or "approximate". Pricing changes and service_tier opacity make these directional.
5. **Deduplicate by message.id**: Every assistant message appears at least twice in JSONL (thinking + text). Summing without dedup doubles the token counts.
6. **Evidence for every finding**: Cite specific metrics (e.g., "53 turns, avg 62,000 input/turn") — never vague statements like "this session seems long."
7. **Session type awareness**: Check the session title and first user prompt to understand purpose. Adjust expectations accordingly (debugging vs code review vs chat).
8. **Subagent cost ≠ waste**: Parallel test execution or research exploration via subagents is legitimate. Flag as pattern to investigate, not a definite issue.
9. **5-minute cache TTL note**: Since March 2026, cache TTL is 5 minutes. Sessions with inter-turn gaps >5 min naturally have low hit rates — note this context, don't always flag as 严重.
10. **jq fallback**: If jq is unavailable, use Grep + targeted Read. Note "jq 不可用，使用回退解析" in the report footer.
11. **Short session skip**: Sessions with <5 turns should skip most patterns. Note "会话过短，跳过深度分析" and only show basic stats.
12. **Respect file permissions**: Do not attempt to read `.jsonl` files the user can't access. Skip them with a note.
