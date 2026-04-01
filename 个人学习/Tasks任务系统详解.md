# Claude Code Tasks（任务系统）详解

> 面向新手的任务系统完全指南

---

## 什么是 Tasks？

**Tasks（任务）** 是 Claude Code 中**后台工作**的抽象。比如：
- 运行一个耗时的 Bash 命令
- 启动一个子 Agent 处理复杂任务
- 远程执行任务

简单说：**Task = 后台要完成的工作**

---

## 任务类型一览

| 任务类型 | 类比 | 说明 |
|---------|------|------|
| `LocalShellTask` | 本地终端命令 | 在本地运行 Shell/Bash 命令 |
| `LocalAgentTask` | 本地子 Agent | 在本地启动一个 AI 子代理 |
| `RemoteAgentTask` | 远程子 Agent | 在远程启动一个 AI 子代理 |
| `InProcessTeammateTask` | 同进程队友 | 在同一个进程内的队友 Agent |
| `LocalWorkflowTask` | 本地工作流 | 执行本地工作流脚本 |
| `MonitorMcpTask` | MCP 监控任务 | 监控 MCP 服务器 |
| `DreamTask` | 梦境任务 | 特殊的后台思考任务 |

---

## 目录结构

```
src/tasks/
├── types.ts                    ← 所有任务状态的类型定义
├── LocalShellTask/             ← 本地 Shell 任务
│   ├── LocalShellTask.tsx      ← 主实现
│   ├── guards.ts               ← 类型守卫
│   └── killShellTasks.ts       ← 杀死任务
├── LocalAgentTask/             ← 本地 Agent 任务
│   └── LocalAgentTask.tsx      ← 主实现
├── RemoteAgentTask/            ← 远程 Agent 任务
├── InProcessTeammateTask/      ← 同进程队友任务
├── LocalWorkflowTask/          ← 工作流任务
├── DreamTask/                  ← 梦境任务
└── MonitorMTask/               ← 监控任务
```

---

## 核心类型定义

### 任务状态基类

```typescript
// Task.ts 中的基础接口
interface TaskStateBase {
  id: string              // 唯一标识
  status: 'pending' | 'running' | 'completed' | 'failed' | 'killed'
  notified: boolean       // 是否已通知
  createdAt: number       // 创建时间
}
```

### LocalShellTaskState（Shell 任务状态）

```typescript
type LocalShellTaskState = TaskStateBase & {
  type: 'local_shell'
  description: string     // 命令描述
  exitCode?: number       // 退出码
  outputPath: string      // 输出文件路径
  isBackgrounded: boolean  // 是否后台运行
  toolUseId?: string      // 关联的工具调用 ID
}
```

### LocalAgentTaskState（Agent 任务状态）

```typescript
type LocalAgentTaskState = TaskStateBase & {
  type: 'local_agent'
  agentId: string         // Agent ID
  prompt: string           // 给 Agent 的提示
  model?: string           // 使用的模型
  abortController?: AbortController
  progress?: AgentProgress // 进度信息
  retrieved: boolean       // 是否已获取结果
  messages?: Message[]    // Agent 的消息历史
  isBackgrounded: boolean  // 是否后台运行
}
```

---

## 任务生命周期

```
创建任务
    ↓
状态: pending（等待中）
    ↓
开始执行
    ↓
状态: running（运行中）
    ↓
┌─────────────────┐
│ 执行中...       │
│ - 可以后台挂起   │
│ - 可以取消      │
└─────────────────┘
    ↓
执行完成
    ↓
状态: completed / failed / killed
```

---

## Shell 任务详解

### 创建 Shell 任务

```typescript
// BashTool 内部调用
spawnShellTask({
  command: 'npm install',
  description: 'Install dependencies',
  cwd: '/project',
  timeout: 60000,
})
```

### 后台任务

Shell 任务可以**后台运行**：

```typescript
// 在命令后加 & 或者设置 isBackgrounded: true
spawnShellTask({
  command: 'npm run dev &',
  isBackgrounded: true,
})
```

### 任务通知

当后台任务完成时，会发送通知：

```
Background command "npm install" completed (exit code 0)
```

### 交互式提示检测

Claude Code 会检测命令是否在等待交互输入：

```typescript
// 如果检测到这些模式，会提醒用户
const PROMPT_PATTERNS = [
  /\(y\/n\)/i,
  /\[y\/n\]/i,
  /Press (any key|Enter)/i,
  /Continue\?/i,
]

function looksLikePrompt(tail: string): boolean {
  // 检测输出末尾是否像交互式提示
}
```

---

## Agent 任务详解

### 创建 Agent 任务

```typescript
// AgentTool 内部调用
const task = await createLocalAgentTask({
  prompt: '帮我重构这个文件',
  agentDefinition: selectedAgent,
  tools: availableTools,
  parentAgentId: currentAgentId,
})
```

### 进度追踪

Agent 任务会追踪进度：

```typescript
type AgentProgress = {
  toolUseCount: number      // 已使用的工具数量
  tokenCount: number        // 已消耗的 token 数
  lastActivity?: ToolActivity  // 最近一次活动
  recentActivities?: ToolActivity[]  // 最近 5 次活动
  summary?: string          // 摘要
}

type ToolActivity = {
  toolName: string
  input: Record<string, unknown>
  activityDescription?: string  // 如 "Reading src/foo.ts"
  isSearch?: boolean        // 是否是搜索操作
  isRead?: boolean          // 是否是读取操作
}
```

### 消息传递

Agent 可以通过 `SendMessage` 互相通信：

```typescript
// 发送消息给另一个 Agent
SendMessageTool({
  content: '任务完成了',
  recipientId: 'other-agent-id',
})
```

---

## 任务管理工具

Claude Code 提供了一组任务管理工具：

| 工具 | 作用 |
|------|------|
| `TaskCreateTool` | 创建新任务 |
| `TaskListTool` | 列出所有任务 |
| `TaskGetTool` | 获取任务详情 |
| `TaskUpdateTool` | 更新任务状态 |
| `TaskStopTool` | 停止任务 |
| `TaskOutputTool` | 获取任务输出 |

### TaskListTool 输出示例

```json
{
  "tasks": [
    {
      "id": "task_123",
      "type": "local_shell",
      "status": "completed",
      "description": "npm install"
    },
    {
      "id": "task_456",
      "type": "local_agent",
      "status": "running",
      "description": "重构代码"
    }
  ]
}
```

---

## 任务框架

### 注册任务

```typescript
import { registerTask } from 'utils/task/framework.js'

registerTask({
  id: taskId,
  type: 'local_shell',
  state: initialState,
})
```

### 更新状态

```typescript
import { updateTaskState } from 'utils/task/framework.js'

updateTaskState(taskId, setAppState, (task) => ({
  ...task,
  status: 'completed',
  notified: true,
}))
```

### 磁盘输出

任务输出可以保存到磁盘：

```typescript
import { getTaskOutputPath, evictTaskOutput } from 'utils/task/diskOutput.js'

// 获取输出文件路径
const outputPath = getTaskOutputPath(taskId)

// 驱逐（清理）旧任务的输出
evictTaskOutput(taskId)
```

---

## 任务与权限

任务系统和权限紧密集成：

```typescript
// 每个任务都有权限上下文
type TaskPermissionContext = {
  mode: PermissionMode
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
}
```

---

## 前台 vs 后台任务

### 前台任务（Foreground）

- 在主会话中运行
- 用户会等待结果
- `isBackgrounded = false`

### 后台任务（Background）

- 独立运行，不阻塞用户
- 完成后通过通知告知
- `isBackgrounded = true`

```typescript
// 判断是否是后台任务
function isBackgroundTask(task: TaskState): boolean {
  if (task.status !== 'running' && task.status !== 'pending') {
    return false
  }
  if ('isBackgrounded' in task && task.isBackgrounded === false) {
    return false
  }
  return true
}
```

---

## 任务工具类比

| 任务类型 | 现实类比 | 特点 |
|---------|---------|------|
| LocalShellTask | 厨房里的烤箱 | 后台烤东西，不耽误做其他事 |
| LocalAgentTask | 派一个小弟去买菜 | 小弟去买菜，你可以继续做饭 |
| RemoteAgentTask | 叫外卖 | 外面有人帮你做 |
| InProcessTeammate | 同组同事 | 同一个办公室里协作 |

---

## 新手学习建议

### 推荐阅读顺序

1. **`tasks/types.ts`** — 理解任务状态类型
2. **`tasks/LocalShellTask/LocalShellTask.tsx`** — 最简单的任务类型
3. **`tasks/LocalAgentTask/LocalAgentTask.tsx`** — 复杂的 Agent 任务
4. **`utils/task/framework.ts`** — 任务注册和管理框架

### 理解重点

1. **状态机**：pending → running → completed/failed/killed
2. **进度追踪**：toolUseCount、tokenCount、recentActivities
3. **后台运行**：isBackgrounded 标志
4. **输出管理**：磁盘存储 + 驱逐机制

---

## 总结

| 概念 | 说明 |
|------|------|
| Task | 后台工作的抽象 |
| TaskState | 任务状态（pending/running/completed/failed/killed） |
| LocalShellTask | 本地 Shell 命令 |
| LocalAgentTask | 本地 AI 子代理 |
| ProgressTracker | 进度追踪器 |
| isBackgrounded | 是否后台运行 |

**核心思想**：把一切"需要等待的工作"都抽象成 Task，统一管理生命周期和状态。

---

*祝学习愉快！ 🚀*
