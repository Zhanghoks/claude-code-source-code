# Claude Code Context（上下文管理）详解

> 面向新手的上下文系统完全指南

---

## 什么是 Context？

**Context（上下文）** = AI 思考时能看到的"记忆"和"背景信息"。

就像人对话时需要知道：
- 我们在聊什么项目？
- 当前是什么时间？
- 之前做了什么？

Claude Code 的 Context 就是给 AI 提供这些信息。

---

## Context 的组成部分

```
┌─────────────────────────────────────┐
│          完整 Context               │
├─────────────────────────────────────┤
│ 1. System Context（系统上下文）       │
│    - Git 状态                        │
│    - 当前日期                        │
│    - 缓存 breaker（调试用）           │
├─────────────────────────────────────┤
│ 2. User Context（用户上下文）         │
│    - MEMORY.md 内容                  │
│    - CLAUDE.md 文件                  │
├─────────────────────────────────────┤
│ 3. Messages（对话历史）              │
│    - 用户消息                        │
│    - AI 响应                         │
│    - 工具调用                        │
├─────────────────────────────────────┤
│ 4. Tools（可用工具）                  │
│    - 内置工具                        │
│    - MCP 工具                        │
├─────────────────────────────────────┤
│ 5. Agents（可用代理）                │
│    - Agent 定义                     │
└─────────────────────────────────────┘
```

---

## 核心文件一览

| 文件 | 作用 |
|------|------|
| `src/context.ts` | 上下文入口，获取 System/User Context |
| `src/memdir/memdir.ts` | 记忆目录管理（MEMORY.md） |
| `src/memdir/memoryTypes.ts` | 记忆文件类型定义 |
| `src/memdir/memoryAge.ts` | 记忆新鲜度计算 |
| `src/services/Compactor.ts` | 上下文压缩（Compact） |
| `src/utils/context.ts` | 上下文工具函数 |
| `src/utils/analyzeContext.ts` | 上下文分析 |

---

## System Context（系统上下文）

### Git 状态

```typescript
// src/context.ts
export const getGitStatus = memoize(async () => {
  const [branch, mainBranch, status, log, userName] = await Promise.all([
    getBranch(),
    getDefaultBranch(),
    execFileNoThrow(gitExe(), ['--no-optional-locks', 'status', '--short']),
    execFileNoThrow(gitExe(), ['--no-optional-locks', 'log', '--oneline', '-n', '5']),
    execFileNoThrow(gitExe(), ['config', 'user.name']),
  ])

  return [
    `Current branch: ${branch}`,
    `Main branch: ${mainBranch}`,
    `Git user: ${userName}`,
    `Status:\n${status || '(clean)'}`,
    `Recent commits:\n${log}`,
  ].join('\n\n')
})
```

### 输出示例

```
Current branch: main
Main branch (you will usually use this for PRs): main
Git user: Zhanghoks
Status:
M src/main.tsx
?? src/new-file.ts

Recent commits:
3da94d5 Update copyright attribution
1d3ebd7 Add copyright disclaimer
74f80e0 Make doc tree filenames clickable
```

---

## User Context（用户上下文）

### MEMORY.md 记忆系统

Claude Code 支持**记忆文件**，让你告诉 AI 重要的背景信息：

```markdown
# MEMORY.md

## 项目概述
这是一个 Claude Code 源码学习项目

## 用户偏好
- 使用中文交流
- 喜欢详细的文档

## 当前任务
学习 Claude Code 源码结构
```

### 记忆文件发现流程

```
1. 扫描当前目录
2. 查找 MEMORY.md
3. 查找 CLAUDE.md
4. 查找 .claude/ 目录下的文件
5. 加载并合并
```

### 记忆新鲜度

```typescript
// src/memdir/memoryAge.ts

// 计算记忆年龄（天数）
function memoryAgeDays(mtimeMs: number): number {
  return Math.max(0, Math.floor((Date.now() - mtimeMs) / 86_400_000))
}

// 转换为人类可读格式
function memoryAge(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d === 0) return 'today'
  if (d === 1) return 'yesterday'
  return `${d} days ago`
}
```

### 过期提醒

超过 1 天的记忆会收到警告：

```
> WARNING: This memory is 5 days old. Memories are point-in-time observations,
> not live state — claims about code behavior or file:line citations may be outdated.
```

---

## Context 管理

### 获取上下文

```typescript
// src/context.ts
export const getSystemContext = memoize(async () => {
  return {
    gitStatus: await getGitStatus(),
    cacheBreaker: getSystemPromptInjection(), // 调试用
  }
})

export const getUserContext = memoize(async () => {
  return {
    claudeMd: await getClaudeMds(),  // 记忆文件
    currentDate: `Today's date is ${getLocalISODate()}.`,
  }
})
```

### 上下文使用的地方

```
┌─────────────────────────────────────┐
│         query.ts（主循环）           │
│   - 每次 AI 思考前合并上下文          │
│   - 每次响应后可能触发压缩            │
└─────────────────────────────────────┘
```

---

## 上下文压缩（Compact）

当对话太长时，Claude Code 会**压缩上下文**，保留重要信息，丢弃细节。

### 压缩触发条件

```typescript
// src/services/Compactor.ts
export const MODEL_CONTEXT_WINDOW_DEFAULT = 200_000  // 20万 token

// 当上下文接近上限时触发压缩
```

### 压缩策略

| 策略 | 说明 |
|------|------|
| `autoCompact` | 自动压缩 |
| `snipCompact` | 掐头（删除最早的对话） |
| `contextCollapse` | 折叠（把旧对话浓缩） |

### 微压缩（Micro-Compact）

```typescript
// src/services/compact/microCompact.ts

// 轻量级压缩，只压缩消息格式
function microcompactMessages(messages: Message[]): Message[] {
  // 移除冗余元数据
  // 简化格式
}
```

---

## 上下文分析

### 分析工具

```typescript
// src/utils/analyzeContext.ts

type ContextData = {
  totalTokens: number
  maxTokens: number
  percentage: number
  model: string
  categories: {
    name: string
    tokens: number
    count: number
  }[]
  memoryFiles?: MemoryFile[]
  mcpTools?: MCPTool[]
  agents?: AgentDefinition[]
}
```

### 查看上下文使用

```
/context
```

这个命令会输出当前上下文使用情况的表格：

```
| Category | Tokens | Count |
|----------|--------|-------|
| Messages | 45,231 | 128 |
| Tools | 12,456 | 42 |
| System | 3,200 | 5 |
| Memory | 1,500 | 3 |
|----------|--------|-------|
| Total | 62,387 | 178 |
| Model: claude-sonnet-4-5
| Usage: 31% of 200,000 tokens
```

---

## Context 的限制

### 模型上下文窗口

```typescript
// src/utils/context.ts
export const MODEL_CONTEXT_WINDOW_DEFAULT = 200_000  // 20万 token

// 支持 1M 上下文的模型
export function modelSupports1M(model: string): boolean {
  return canonical.includes('claude-sonnet-4') ||
         canonical.includes('opus-4-6')
}
```

### 1M 上下文

某些模型支持 100 万 token 的上下文：

```typescript
// 模型名称带 [1m] 后缀表示启用
has1mContext(model)  // 检查是否启用
```

### 1M 禁用

企业用户可能需要禁用 1M 上下文（HIPAA 合规）：

```typescript
export function is1mContextDisabled(): boolean {
  return isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_1M_CONTEXT)
}
```

---

## 消息与 Context

### 消息结构

```typescript
type Message =
  | UserMessage      // 用户消息
  | AssistantMessage // AI 响应
  | SystemMessage    // 系统消息
  | ProgressMessage  // 进度消息
```

### 压缩边界

对话中会有一个**压缩边界标记**：

```typescript
// 压缩边界之前的消息可以被压缩/删除
// 压缩边界之后的消息必须保留

function getMessagesAfterCompactBoundary(messages: Message[]): Message[] {
  // 返回压缩边界后的消息
}
```

---

## 记忆文件最佳实践

### MEMORY.md 结构

```markdown
# MEMORY.md

## 关于我
- 名字：张三
- 偏好：中文交流

## 当前项目
- 项目名：我的网站
- 技术栈：React + Node.js
- 主要任务：添加登录功能

## 重要约定
- 所有 API 调用使用 /api 前缀
- 样式使用 Tailwind CSS
```

### 不要存储的内容

```typescript
// 这些不应该放进 MEMORY.md
const WHAT_NOT_TO_SAVE_SECTION = [
  '临时变量',
  '代码片段（会过时）',
  '文件路径（会变化）',
  '机密信息',
]
```

---

## Context 管理命令

| 命令 | 作用 |
|------|------|
| `/context` | 显示当前上下文使用情况 |
| `/compact` | 手动触发压缩 |
| `/clear` | 清空对话历史 |

---

## 新手学习建议

### 推荐阅读顺序

1. **`src/context.ts`** — 理解上下文入口
2. **`src/memdir/memdir.ts`** — 记忆目录管理
3. **`src/memdir/memoryAge.ts`** — 记忆新鲜度
4. **`src/services/Compactor.ts`** — 压缩机制
5. **`src/utils/analyzeContext.ts`** — 上下文分析

### 理解重点

1. **两层缓存**：System Context + User Context
2. **记忆文件**：MEMORY.md 的发现和加载
3. **压缩机制**：当上下文太大时如何处理
4. **Token 限制**：模型不是无限制的

---

## 流程图

```
用户启动 Claude Code
        ↓
加载 System Context
  - Git 状态
  - 环境变量
        ↓
加载 User Context
  - MEMORY.md
  - CLAUDE.md
  - 当前日期
        ↓
合并所有上下文
        ↓
发送给 AI 模型
        ↓
AI 响应 + 工具调用
        ↓
如果上下文太大 → 触发压缩
        ↓
返回结果给用户
```

---

## 总结

| 概念 | 说明 |
|------|------|
| System Context | 系统级信息（Git 状态等） |
| User Context | 用户记忆（MEMORY.md） |
| Messages | 对话历史 |
| Compact | 上下文压缩机制 |
| Memory Age | 记忆新鲜度计算 |

**核心思想**：把 AI 需要知道的一切都组织成 Context，在有限的 Token 限制内最大化利用。

---

*祝学习愉快！ 🚀*
