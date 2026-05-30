# Token管家 · Token Budget Manager

[English](#english) | [中文](#chinese)

---

<h2 id="english">English</h2>

Analyze Claude Code conversation transcripts (JSONL) to identify **token waste patterns** and provide optimization recommendations. Complements the AI Code Reviewer — one checks code quality, this checks conversation efficiency.

### What It Detects (6 Waste Patterns)

| # | Pattern | What It Catches |
|---|---------|-----------------|
| 1 | **Context Inflation** | Sessions running too long (30+/50+/80+ turns), input tokens growing with each turn |
| 2 | **Cache Thrashing** | Prompt cache created but never read — cache writes wasted due to session gaps >5 min |
| 3 | **Read:Write Ratio** | Input massively outweighs output (>20:1 / >50:1 / >99:1), signaling excessive context stuffing |
| 4 | **Output Verbosity** | Overly long assistant responses (avg >300/500/1000, max >4000 tokens per message) |
| 5 | **Subagent Dominance** | Subagents consuming disproportionate tokens (>30% / >50% / >80% of session total) |
| 6 | **Session Fragmentation** | Same files re-read across multiple sessions, indicating workflow siloing |

### Installation

```bash
# Clone into your Claude Code skills directory
git clone https://github.com/ReiSei0824/token-budget-manager.git ~/.claude/skills/token-budget-manager
```

Or add it as a submodule:

```bash
cd ~/.claude/skills
git submodule add https://github.com/ReiSei0824/token-budget-manager.git token-budget-manager
```

### Usage

Trigger the skill by mentioning any of these phrases in Claude Code:

**Chinese**: `token管家`, `token分析`, `分析token使用`, `token浪费`, `对话token利用率`, `预算管理`, `token优化`, `token检查`, `token审计`, `检查token消费`, `快查token`, `token快查`

**English**: `token budget analysis`, `analyze token usage`, `check token waste`, `optimize tokens`, `token audit`, `token health check`, `quick token check`, `token utilization`

Five analysis modes are supported:

| Mode | Trigger | Description |
|------|---------|-------------|
| **Current Session** | `分析当前对话` / `analyze this conversation` | Analyze the currently active session |
| **Specific Session** | Provide a session GUID | Analyze a single session by ID |
| **Recent Sessions** | `最近7天` / `past N days` | Batch-analyze sessions from the past N days |
| **Specific Project** | Name a project | Analyze all sessions under a project directory |
| **Quick Check** | `快查` / `quick token check` | Lightweight 3-metric check in ~5 seconds |

### Workflow

```
Discover Sessions → Parse Usage Data → 6 Detection Passes → Health Score → Report
```

1. **Discover** — finds `.jsonl` transcript files by session ID, date range, or project
2. **Parse** — extracts token usage with deduplication, counts turns, discovers subagents
3. **Detect** — runs all 6 waste pattern checks with severity classification (严重/中等/轻微)
4. **Score** — computes health score (A–F) and generates ranked optimization tips
5. **Report** — structured output: executive summary, waste breakdown, per-session ranking, baseline comparison

### What It Does NOT Do

- Modify or delete transcripts (read-only)
- Access Anthropic billing data (uses local estimates)
- Read raw conversation content beyond what's needed for statistics

### Requirements

- **jq** (recommended) — for fast JSONL parsing; falls back to Grep + Read if unavailable
- Claude Code session transcripts (`.jsonl` files)

---

<h2 id="chinese">中文</h2>

分析 Claude Code 对话记录（JSONL），识别 **Token 浪费模式** 并提供优化建议。与 AI Code Reviewer 互补 — 一个检查代码质量，这个检查对话效率。

### 六大检测模式

| # | 模式 | 检测内容 |
|---|------|---------|
| 1 | **上下文膨胀** | 会话轮次过长（30/50/80+ 轮），每轮输入 token 持续增长 |
| 2 | **缓存颠簸** | 缓存写入后从未被读取 — 因会话间隔超过 5 分钟导致缓存浪费 |
| 3 | **读写比失衡** | 输入远超输出（>20:1 / >50:1 / >99:1），存在大量冗余上下文 |
| 4 | **输出冗余** | 回复过长（平均 >300/500/1000 token，单次峰值 >4000 token） |
| 5 | **子代理成本霸权** | 子代理消耗占比过高（>30% / >50% / >80%） |
| 6 | **会话碎片化** | 多个会话重复读取相同文件，工作流孤岛化 |

### 安装

```bash
# 克隆到 Claude Code 技能目录
git clone https://github.com/ReiSei0824/token-budget-manager.git ~/.claude/skills/token-budget-manager
```

如果你已经在用 git 管理 skills 目录：

```bash
cd ~/.claude/skills
git submodule add https://github.com/ReiSei0824/token-budget-manager.git token-budget-manager
```

### 使用方式

在 Claude Code 中说以下任一关键词即可触发：

**中文**: `token管家`, `token分析`, `分析token使用`, `token浪费`, `对话token利用率`, `预算管理`, `token优化`, `token检查`, `token审计`, `检查token消费`, `快查token`, `token快查`

**英文**: `token budget analysis`, `analyze token usage`, `check token waste`, `optimize tokens`, `token audit`, `token health check`, `quick token check`, `token utilization`

支持五种分析模式：

| 模式 | 触发方式 | 说明 |
|------|---------|------|
| **当前会话** | `分析当前对话` | 分析正在进行的会话 |
| **指定会话** | 提供会话 GUID | 按 ID 分析单个会话 |
| **近期会话** | `最近7天` | 批量分析过去 N 天的所有会话 |
| **指定项目** | 指定项目名 | 分析某个项目目录下的所有会话 |
| **快速检查** | `快查` / `quick` | 仅需 3 项指标，约 5 秒完成 |

### 工作流

```
发现会话 → 解析用量 → 6 项浪费检测 → 健康评分 → 输出报告
```

1. **发现会话** — 按会话 ID、日期范围或项目查找 `.jsonl` 记录文件
2. **解析用量** — 提取 token 用量（自动去重）、统计轮次、发现子代理
3. **检测浪费** — 运行全部 6 项检测，输出严重度（严重/中等/轻微）
4. **健康评分** — 计算综合评分（A–F），生成按影响排序的优化建议
5. **输出报告** — 结构化输出：执行摘要、浪费明细、会话排名、健康基线对照

### 不做什么

- 不修改或删除对话记录（只读分析）
- 不访问 Anthropic 账单数据（使用本地估算）
- 不读取对话原始内容（仅提取统计数据）

### 依赖

- **jq**（推荐）— 快速 JSONL 解析；不可用时自动回退到 Grep + Read
- Claude Code 会话记录文件（`.jsonl`）

---

## File Structure

```
token-budget-manager/
├── SKILL.md                              # Main skill definition (307 lines)
├── README.md                             # This file
└── references/
    ├── cost-estimation.md                # Token pricing & savings formulas
    ├── waste-patterns.md                 # 6 detection algorithms with thresholds
    ├── output-template.md                # Report structure & quick check card
    └── healthy-baselines.md              # Baseline metrics for comparison
```

## License

MIT
