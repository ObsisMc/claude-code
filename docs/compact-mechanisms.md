# Claude Code 上下文压缩机制解析

> 代码来源：`c:\home\codebase\claude-code-mine\src`（基于 claude-code 内部代码库）

---

## 总览

Claude Code 有多层上下文管理机制，从细粒度到粗粒度依次为：

```
每轮请求前
  └─ 0. snipCompactIfNeeded()           ← 模型主动调用，删除中间消息段，ant-only
  └─ 1. microcompactMessages()          ← 每轮都跑，清理 tool result 内容
  └─ 2. shouldAutoCompact()?
       └─ autoCompactIfNeeded()
            ├─ 3. trySessionMemoryCompaction()   ← 优先尝试，无 API 调用
            │       └─ 返回 null 时 fallback 到:
            └─ 4. compactConversation()          ← 调 API 生成 summary

[API 调用]

  遇到 413 prompt_too_long:
  └─ 5. reactiveCompact.tryReactiveCompact()    ← 兜底，ant-only

  Context Collapse 启用时（替代 autocompact）:
  └─ 6. contextCollapse（marble_origami）       ← 外科式区间压缩，ant-only
```

> Snip（0）与其他机制**不互斥**，同一轮里可以先 snip 再 microcompact 再 autocompact。

各层之间的 token 阈值关系（以 effectiveContextWindow 为基准）：

```
0%        90%           93%           95%         413
|---------|-------------|-------------|-----------|------>
          collapse      autocompact   collapse    reactive
          staging 开始  触发阈值      blocking    compact 兜底
                       （collapse 开启时被禁用）
```

---

## 0. Snip（HISTORY_SNIP，ant-only）

**文件**：`src/services/compact/snipCompact.ts`，`src/tools/SnipTool/`

与 compact 系列的根本区别：compact 替换对话**前缀**，Snip 是**从历史中间挖掉某段消息**，不生成 summary，直接删除。

### 模型如何引用消息

每条 user message 在发给 API 前会被注入一个短 ID 标签（`messages.ts:1614`）：

```
[id:a3f9k2] 用户消息内容...
```

ID 是从消息 UUID 派生的 6 位 base36 字符串（`deriveShortMessageId`，确定性）。模型调用 SnipTool 时通过这些 ID 指定要删除的消息范围。

### 执行流程

`snipCompactIfNeeded()` 在 `query.ts:403` 调用，返回 `{ messages, tokensFreed, boundaryMessage }`。

`snipTokensFreed` 必须单独传给 `shouldAutoCompact()`（`autoCompact.ts:167`），原因：被删消息虽已不在列表，但**存活 assistant 消息的 `usage` 字段仍记录 snip 前的上下文大小**，`tokenCountWithEstimation` 读到的是旧值，不修正会导致 autocompact 误判。

### Session 恢复

Boundary message 记录 `snipMetadata.removedUuids`。Resume 时 `applySnipRemovals()`（`sessionStorage.ts:1982`）过滤被删消息并重新链接 `parentUuid`——不这样做，parentUuid 链会把已删消息重新拼回来（真实 bug：397K tokens 显示 → 1.65M 实际，`adamr-20260320-165831`）。

---

## 1. MicroCompact

**文件**：`src/services/compact/microCompact.ts`

不改变对话结构，只清空旧 tool result 的内容（file read、shell、grep 等）。每轮请求前通过 `microcompactMessages()` 调用。

### 1.1 Cached Microcompact（ant-only）

**触发条件**：feature flag `CACHED_MICROCOMPACT` 开启，且模型支持 cache editing。

使用 cache editing API 在**服务端**删除 tool result，本地消息不变，不破坏已有缓存 prefix。需要 `tengu_cache_plum_violet` GrowthBook flag。

相关代码：`microCompact.ts:305` `cachedMicrocompactPath()`

```
注册 tool result → getToolResultsToDelete() → createCacheEditsBlock()
→ pendingCacheEdits（由 API 层消费）
```

### 1.2 Time-based Microcompact

**触发条件**：距上次 assistant 消息的时间间隔超过阈值（`gapThresholdMinutes`，来自 GrowthBook `getTimeBasedMCConfig()`）。

**缓存已过期**，直接 **mutate 本地消息内容**清空旧 tool result，保留最近 N 条（`keepRecent`）。清完后重置 cachedMCState，避免下次 cached MC 尝试删除服务端不存在的条目。

相关代码：`microCompact.ts:422` `evaluateTimeBasedTrigger()` / `maybeTimeBasedMicrocompact()`

---

## 2. SessionMemory Compact

**文件**：`src/services/compact/sessionMemoryCompact.ts`

**核心优势：不需要 API 调用**。依赖后台 Session Memory 模块持续维护的会话记忆文件作为 summary。

**Gate**：同时需要 `tengu_session_memory` + `tengu_sm_compact` 两个 feature flag，或 `ENABLE_CLAUDE_CODE_SM_COMPACT` 环境变量。

### 工作流程

```
trySessionMemoryCompaction()
  └─ shouldUseSessionMemoryCompaction()  → 检查 flag
  └─ waitForSessionMemoryExtraction()    → 等待后台提取完成
  └─ getSessionMemoryContent()           → 读取记忆文件
  └─ calculateMessagesToKeepIndex()      → 确定保留哪些消息
  └─ createCompactionResultFromSessionMemory()  → 构建结果（无 API 调用）
```

### 消息保留逻辑（`calculateMessagesToKeepIndex`，`sessionMemoryCompact.ts:324`）

从 `lastSummarizedMessageId` 之后开始，向前扩展直到满足：
- 至少 `minTokens`（默认 10,000）token
- 至少 `minTextBlockMessages`（默认 5）条有文本内容的消息
- 不超过 `maxTokens`（默认 40,000）
- 不打断 tool_use / tool_result 配对（`adjustIndexToPreserveAPIInvariants`）

阈值配置来自 GrowthBook `tengu_sm_compact_config`，可远程调整。

### Fallback 条件

以下情况返回 `null`，交由 `compactConversation()` 处理：
- Session memory 文件不存在
- 文件内容是空模板（尚未提取到实际内容）
- `lastSummarizedMessageId` 在当前消息列表中找不到
- 压缩后 token 数仍超过阈值（`postCompactTokenCount >= autoCompactThreshold`）

---

## 3. AutoCompact 协调层

**文件**：`src/services/compact/autoCompact.ts`

`autoCompactIfNeeded()` 是协调入口，判断何时压缩以及选哪条路径。

### 触发阈值（`getAutoCompactThreshold`，`autoCompact.ts:72`）

```
autoCompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS (13,000)

effectiveContextWindow = contextWindow - min(maxOutputTokens, 20,000)
```

可通过 `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` 环境变量覆盖（百分比，用于测试）。

### 三重互斥 Guard（`shouldAutoCompact`，`autoCompact.ts:160`）

| 条件 | 原因 |
|---|---|
| `querySource === 'session_memory' \| 'compact'` | 防递归死锁（forked agent 共享进程） |
| `REACTIVE_COMPACT` + `tengu_cobalt_raccoon` flag 开启 | Reactive-only 模式，让 reactive 兜底 |
| `CONTEXT_COLLAPSE` 启用时 | Collapse 自己管理 headroom，autocompact 在 93% 触发会与 collapse 的 90%~95% 窗口竞争 |

注意：gating 刻意放在 `shouldAutoCompact` 而非 `isAutoCompactEnabled`，保留 reactiveCompact 作为 413 兜底（它直接查 `isAutoCompactEnabled`）。

### 执行流程（`autoCompactIfNeeded`，`autoCompact.ts:241`）

```
1. trySessionMemoryCompaction()  → 成功则返回，无需 API 调用
2. compactConversation()         → 完整压缩，需要 API 调用
```

### 熔断器（Circuit Breaker）

连续失败 3 次（`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES`）后停止重试，避免上下文已不可恢复时继续浪费 API 调用（BQ 数据：曾有 session 连续失败 3,272 次，浪费约 250K API calls/day）。

---

## 4. compactConversation（底层实现）

**文件**：`src/services/compact/compact.ts`

实际的压缩逻辑：调 Claude API，用压缩 prompt 生成整段历史的 summary，然后把对话替换为：

```
[boundaryMarker] + [summaryMessages] + [messagesToKeep] + [attachments]
```

被以下三方共用：
- `autoCompactIfNeeded()` 的 fallback 路径
- 手动 `/compact` 命令（`src/commands/compact/compact.ts`）
- `reactiveCompact.reactiveCompactOnPromptTooLong()`

压缩后调用 `runPostCompactCleanup()`：重置 microcompact state，若是 main thread 还重置 contextCollapse state。

---

## 5. ReactiveCompact（ant-only）

**文件**：`src/services/compact/reactiveCompact.ts`（不在此 codebase，通过 `feature('REACTIVE_COMPACT')` 动态加载）

**触发时机**：API 返回真实的 413 `prompt_too_long` 错误之后，被动触发。

在 `query.ts:1119` 收到 withheld 413 后调用：

```typescript
const compacted = await reactiveCompact.tryReactiveCompact({
  hasAttempted: hasAttemptedReactiveCompact,
  querySource,
  ...
})
```

即使 autocompact 被 context-collapse 关闭，reactive 也保持活跃作为最后兜底（`autoCompact.ts:207` 的注释）。

---

## 6. Context Collapse（ant-only）

**文件**：`src/services/contextCollapse/index.ts`（不在此 codebase，通过 `feature('CONTEXT_COLLAPSE')` 动态加载）

内部代号：**marble_origami**（ctx-agent 的 query source）。

与 compact 系列不同，Context Collapse 不是整体替换对话，而是**外科式压缩**——只把低价值的消息区间替换为占位符，其余保持原样。

### 占位符格式

```
<collapsed id="abc123">这段消息的 summary...</collapsed>
```

持久化到 transcript（`src/types/logs.ts:255`）：

```typescript
type ContextCollapseCommitEntry = {
  type: 'marble-origami-commit'
  collapseId: string
  summaryUuid: string
  summaryContent: string  // <collapsed id="...">...</collapsed>
}
```

### 两阶段流程

**Staging 阶段（约 90% 时）**：ctx-agent 分析上下文，标记可折叠的区间（含 summary、risk 评分），写入 `marble-origami-snapshot` 条目。此时消息还在，只是标记候选。

**Commit / Drain 阶段（约 95% 或真实 413 时）**：把 staged 区间真正提交，占位符替换原消息。

413 时的恢复路径（`query.ts:1094`）：

```
recoverFromOverflow()
  → drained.committed > 0 → 重试请求（transition: 'collapse_drain_retry'）
  → committed === 0       → fallthrough 到 reactiveCompact
```

### 在 query loop 中的位置（`query.ts:440`）

```typescript
const collapseResult = await contextCollapse.applyCollapsesIfNeeded(
  messagesForQuery,
  toolUseContext,
  querySource,
)
messagesForQuery = collapseResult.messages
```

在每轮 API 调用**之前**，把已 committed 的区间投影成占位符（`projectView`），从 ctx-agent 视角看到的是折叠后的对话。

### 与 AutoCompact 的互斥

Context Collapse 启用时，`shouldAutoCompact()` 直接返回 `false`（`autoCompact.ts:215`）。两者的 token 阈值窗口有重叠（autocompact ~93%，collapse 90%~95%），若同时运行会竞争，autocompact 通常会先赢，破坏 collapse 精心保存的粒度。

---

## 各机制对比

| 机制 | 触发方 | 触发时机 | 是否需要 API | 粒度 | 可用范围 |
|---|---|---|---|---|---|
| **Snip** | 模型主动调 Tool | 任意轮次 | 否 | 任意中间段（直接删除） | ant-only |
| Cached Microcompact | 系统自动 | 每轮（工具数超阈值） | 否（cache editing） | tool result 级 | ant-only |
| Time-based Microcompact | 系统自动 | 每轮（时间间隔超阈值） | 否 | tool result 级 | 全员 |
| SessionMemory Compact | 系统自动 | token > 阈值 | 否 | 消息级 | ant-only（flag） |
| AutoCompact | 系统自动 | token > 阈值（~93%） | 是 | 整段前缀 | 全员 |
| ReactiveCompact | 系统自动 | 413 后 | 是 | 整段前缀 | ant-only |
| Context Collapse | ctx-agent | 90% staging / 95% commit | 是 | 区间级（占位符替换） | ant-only |
