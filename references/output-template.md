# Report Output Template

Use this exact structure when generating reports in Phase 4. Replace `{placeholders}` with actual values. Omit sections marked `[conditional]` when not applicable.

## Language Rule

- User query in Chinese → Report in Chinese (use Chinese labels below)
- User query in English → Report in English (use English labels below)
- Mixed → Default to Chinese

---

# Token 使用分析报告

## 执行摘要

| 指标 | 数值 |
|------|------|
| 分析会话数 | {N} |
| 总输入 Token | {N} |
| 总输出 Token | {N} |
| 总 Token 量 | {N} |
| 估算成本 | ${N.NN} (近似值) |
| 缓存命中率 | {N}% ({healthy/warning/critical}) |
| 平均轮次/会话 | {N} |

**综合健康评分**: {A/B/C/D/F} ({优秀/良好/一般/较差/很差})

---

## 浪费模式明细

| 模式 | 严重度 | 估算浪费 | 节约潜力 |
|------|--------|---------|---------|
| 上下文膨胀 | {严重/中等/轻微/无} | {N} tokens | ~{N} tokens |
| 缓存颠簸 | {严重/中等/轻微/无} | {N} tokens | ~{N} tokens |
| 读写比失衡 | {严重/中等/轻微/无} | {N} tokens | ~{N} tokens |
| 输出冗余 | {严重/中等/轻微/无} | {N} tokens | ~{N} tokens |
| 子代理成本 | {严重/中等/轻微/无} | {N} tokens | ~{N} tokens |
| 会话碎片化 | {严重/中等/轻微/无} | {N} tokens | ~{N} tokens |

---

## 浪费模式详情

### {模式名称} — 🔴严重 / 🟡中等 / 🟢轻微

**检测方式**: {一句话说明测量了什么}

**数据**:
- {具体数据点1}
- {具体数据点2}

**为什么重要**: {用一句话解释影响}

**建议**: {可操作的具体建议}

[重复以上 block，仅输出有检测到的模式。跳过的模式（无发现）用一行 "无此模式" 标注]

---

## 会话排名

| 排名 | 会话 ID | 标题 | Token 总量 | 估算成本 | 健康 | 主要问题 |
|------|---------|------|-----------|---------|------|---------|
| 1 | {guid} | {title} | {N} | ${N} | {grade} | {pattern} |

[每会话一行，按成本降序排列]

---

## 优化建议（按影响排序）

### 1. {最有影响力的建议}
**预计节约**: ~{N} tokens / ${N}

{具体的、可操作的步骤}

### 2. {第二建议}
...

[最多列出5条。跳过没有实质节约潜力的项]

---

## 健康基线对照

| 指标 | 你的数值 | 健康基线 | 状态 |
|------|---------|---------|------|
| 缓存命中率 | {N}% | > 60% | {✅/⚠️/❌} |
| 读写比 | {N}:1 | < 50:1 | {✅/⚠️/❌} |
| 平均轮次/会话 | {N} | < 30 | {✅/⚠️/❌} |
| 平均输出/回复 | {N} | < 500 | {✅/⚠️/❌} |
| 子代理成本占比 | {N}% | < 30% | {✅/⚠️/❌} |
| 会话重复率 | {N}% | < 10% | {✅/⚠️/❌} |

---

## 注意事项

- 所有成本为近似估算，实际费用以 Anthropic 账单为准
- 分析基于本地 JSONL 文件，不包含 API 调用的服务端计量差异
- 会话类型会影响正常值范围（调试 vs 聊天 vs 代码审查各有不同）
- 缓存 TTL 自 2026 年 3 月起为 5 分钟，间歇超过 5 分钟的对话自然缓存命中率低

---

## Quick Check 模板（Phase 5 使用）

```
🔍 Token 速查 — 当前会话 [{sessionId前8位}]

📊 轮次: {N} ({healthy/warning})
📦 缓存命中率: {N}% ({healthy/warning})
📖 读写比: {N}:1 ({healthy/warning})

{✅ 未发现明显浪费 / ⚠️ 发现 {N} 个需关注的问题: {简要列表}}
```
