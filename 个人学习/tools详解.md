# Claude Code Tools 详解

> 面向新手小白的工具系统完全指南

---

## 什么是 Tools？

**Tools（工具）** 是 Claude Code 的"手"——让 AI 能够执行具体操作，比如读写文件、执行命令、搜索内容等。

没有工具，AI 只能"空想"；有了工具，AI 才能真正"做事"。

---

## 工具一览表

Claude Code 内置了 **40+ 种工具**，大致分为以下几类：

### 📁 文件操作类

| 工具名称 | 作用 | 类比 |
|---------|------|------|
| `FileReadTool` | 读取文件内容 | 用眼睛看 |
| `FileEditTool` | 编辑文件（局部修改） | 用笔修改 |
| `FileWriteTool` | 写入文件（整体写入） | 重新写一份 |
| `GlobTool` | 按模式搜索文件 | 找文件 |
| `GrepTool` | 搜索文件内容 | 找文字 |
| `NotebookEditTool` | 编辑 Jupyter 笔记本 | 修改笔记本 |

### 💻 命令执行类

| 工具名称 | 作用 | 类比 |
|---------|------|------|
| `BashTool` | 执行 Shell 命令 | 用命令行操作 |
| `PowerShellTool` | 执行 PowerShell 命令 | Windows 命令行 |
| `REPLTool` | 进入交互式 REPL | 打开交互终端 |

### 🌐 网络类

| 工具名称 | 作用 | 类比 |
|---------|------|------|
| `WebFetchTool` | 获取网页内容 | 下载网页 |
| `WebSearchTool` | 搜索网络 | 百度/Google |

### 🤖 Agent/任务类

| 工具名称 | 作用 | 类比 |
|---------|------|------|
| `AgentTool` | 启动子 Agent | 派一个小弟 |
| `TaskCreateTool` | 创建任务 | 开新任务单 |
| `TaskListTool` | 列出任务 | 看任务列表 |
| `TaskGetTool` | 获取任务详情 | 查任务内容 |
| `TaskUpdateTool` | 更新任务状态 | 改任务状态 |
| `TaskStopTool` | 停止任务 | 中止任务 |
| `TaskOutputTool` | 获取任务输出 | 拿任务结果 |
| `TeamCreateTool` | 创建团队 | 组建团队 |
| `TeamDeleteTool` | 删除团队 | 解散团队 |

### ⚙️ 配置类

| 工具名称 | 作用 | 类比 |
|---------|------|------|
| `ConfigTool` | 读取/修改配置 | 设置菜单 |
| `SkillTool` | 使用技能 | 打开技能 |
| `ToolSearchTool` | 搜索可用工具 | 工具搜索 |

### 🔌 MCP 相关

| 工具名称 | 作用 | 类比 |
|---------|------|------|
| `MCPTool` | 使用 MCP 服务器工具 | 用插件 |
| `ListMcpResourcesTool` | 列出 MCP 资源 | 看插件清单 |
| `ReadMcpResourceTool` | 读取 MCP 资源 | 用插件资源 |

### ⏰ 计划/其他

| 工具名称 | 作用 | 类比 |
|---------|------|------|
| `EnterPlanModeTool` | 进入计划模式 | 制定计划 |
| `ExitPlanModeTool` | 退出计划模式 | 完成计划 |
| `BriefTool` | 生成摘要 | 写总结 |
| `AskUserQuestionTool` | 提问用户 | 问你问题 |
| `SleepTool` | 休眠一段时间 | 休息一下 |
| `SendMessageTool` | 发送消息 | 发短信 |
| `TodoWriteTool` | 写待办事项 | 列待办清单 |
| `LSPTool` | LSP 语言服务 | 代码智能 |
| `ExitWorktreeTool` | 退出 git worktree | 切换分支 |
| `EnterWorktreeTool` | 进入 git worktree | 切换分支 |

---

## 工具的目录结构

每个工具都是一个独立的文件夹，包含：

```
tools/
├── BashTool/              ← 整个文件夹是一个工具
│   ├── BashTool.tsx       ← 主入口（最重要）
│   ├── UI.tsx             ← 界面渲染
│   ├── prompt.ts          ← 提示词
│   ├── utils.ts           ← 工具函数
│   ├── bashPermissions.ts ← 权限检查
│   ├── bashSecurity.ts    ← 安全检查
│   └── ...                ← 其他辅助文件
├── FileReadTool/
│   ├── FileReadTool.ts    ← 主入口
│   ├── UI.tsx             ← 界面渲染
│   ├── prompt.ts          ← 提示词
│   ├── limits.ts          ← 限制
│   └── ...
├── FileEditTool/
│   ├── FileEditTool.ts    ← 主入口
│   ├── UI.tsx             ← 界面渲染
│   ├── types.ts           ← 类型定义
│   ├── constants.ts       ← 常量
│   └── ...
└── ...
```

---

## 工具的通用结构

每个工具都由 `buildTool()` 函数构建，遵循统一模板：

```typescript
// 简化示例
export const FileReadTool = buildTool({
  name: 'Read',                    // 工具名称
  searchHint: 'read file contents',// 搜索提示
  maxResultSizeChars: 100_000,     // 最大结果大小
  strict: true,                    // 严格模式

  // 工具描述（函数形式，延迟加载）
  async description() {
    return 'A tool for reading files'
  },

  // 提示词（告诉 AI 何时使用）
  async prompt() {
    return getReadToolDescription()
  },

  // 用户-facing 名称
  userFacingName: 'Read',

  // 输入参数验证（Zod schema）
  inputSchema: z.object({
    file_path: z.string(),
    showLineNumbers: z.boolean().optional(),
  }),

  // 实际执行逻辑
  async execute(params, context) {
    // 1. 权限检查
    // 2. 读取文件
    // 3. 返回结果
    return { content: fileContents }
  },
})
```

---

## 核心文件：`src/Tool.ts`

这是工具系统的"工厂"，定义了工具的通用接口：

```typescript
// Tool.ts 核心类型
export type Tool = {
  name: string                    // 工具唯一名称
  searchHint?: string             // 搜索提示
  maxResultSizeChars?: number     // 最大结果字符数
  strict?: boolean                // 是否严格模式

  description(): string | Promise<string>      // 描述
  prompt?(): string | Promise<string>         // 使用提示
  userFacingName?: string                        // 显示名称
  inputSchema: z.ZodType                           // 输入验证

  // 核心：执行函数
  execute(
    params: unknown,             // 输入参数
    context: ToolUseContext,      // 执行上下文
    progress?: (progress: ToolProgressData) => void  // 进度回调
  ): Promise<ToolResultBlockParam>  // 执行结果
}
```

---

## 工具注册：`src/tools.ts`

`tools.ts` 是所有工具的"注册中心"：

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    BashTool,
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    GlobTool,
    GrepTool,
    // ... 更多工具
  ]
}
```

### 特性开关（Feature Flags）

有些工具是"条件编译"的，通过 `feature()` 函数控制：

```typescript
// 只有打开某个特性才会包含
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null

const WebBrowserTool = feature('WEB_BROWSER_TOOL')
  ? require('./tools/WebBrowserTool/WebBrowserTool.js').WebBrowserTool
  : null
```

这叫做 **Dead Code Elimination (DCE)** —— 不用的代码不打包。

---

## 工具执行流程

```
用户/AI 需要执行操作
        ↓
query.ts 调用工具系统
        ↓
找到对应的 Tool
        ↓
验证输入参数 (inputSchema)
        ↓
检查权限 (permissions)
        ↓
执行实际操作 (execute)
        ↓
渲染结果 UI (UI.tsx)
        ↓
返回结果给 AI
```

---

## 权限系统

工具和权限紧密集成，每个工具执行前都要检查权限：

```typescript
// 权限检查示例
const permission = await checkReadPermissionForTool(
  filePath,
  context.permissionContext
)

if (permission.result === 'deny') {
  return {
    content: [{
      type: 'tool_use_blocked',
      reason: 'Permission denied',
    }]
  }
}
```

---

## 最常用的工具详解

### BashTool（命令执行）

```typescript
// 能力：执行任意 shell 命令
// 输入：command (string), timeout? (number)
// 输出：stdout, stderr, exit code

// 特点：
// - 支持后台任务
// - 支持沙箱模式
// - 支持超时控制
// - 命令分类（读/写/搜索）
```

**命令分类系统**（决定 UI 如何显示）：

```typescript
// 搜索命令 → 显示为可折叠
BASH_SEARCH_COMMANDS = new Set(['find', 'grep', 'rg', 'ag', ...])

// 读取命令 → 显示为可折叠
BASH_READ_COMMANDS = new Set(['cat', 'head', 'tail', 'wc', ...])

// 列表命令 → 显示为目录列表
BASH_LIST_COMMANDS = new Set(['ls', 'tree', 'du'])
```

---

### FileReadTool（文件读取）

```typescript
// 能力：读取文件内容
// 输入：file_path (string), offset?, limit?, showLineNumbers?
// 输出：文件内容 + 元数据

// 特点：
// - 支持大文件分块读取
// - 支持行号显示
// - 支持图片处理（截图等）
// - 阻塞设备文件检查（/dev/* 安全）
```

---

### FileEditTool（文件编辑）

```typescript
// 能力：编辑文件（保持原格式）
// 输入：file_path, old_string, new_string
// 输出：编辑结果

// 特点：
// - 保持引号风格
// - 支持 sed 风格编辑
// - 检测文件意外修改
// - 支持 Notebook 编辑
```

---

## 新手学习建议

### 1. 从简单的开始
- `TodoWriteTool` — 代码少，容易理解
- `SleepTool` — 最简单的工具
- `GlobTool` — 文件搜索，相对独立

### 2. 然后学中等复杂度
- `FileReadTool` — 有 UI、有权限、有处理逻辑
- `GrepTool` — 搜索功能，逻辑清晰

### 3. 最后挑战复杂工具
- `BashTool` — 最大最复杂的工具（各种检查、安全、进度）
- `FileEditTool` — 编辑逻辑复杂
- `AgentTool` — 涉及子 Agent 启动

### 阅读顺序推荐

```
Tool.ts (理解接口)
    ↓
tools.ts (了解注册)
    ↓
TodoWriteTool/ (最简单的完整示例)
    ↓
FileReadTool/ (有 UI、权限、处理)
    ↓
BashTool/ (最复杂的工具)
```

---

## 常见问题

**Q: 为什么 BashTool 这么大？**
A: 因为它要处理：命令解析、安全检查、权限管理、进度显示、后台任务、图片处理等。

**Q: 什么是 `buildTool()`？**
A: 这是一个工厂函数，统一创建工具实例，封装通用逻辑（输入验证、错误处理等）。

**Q: 工具的 `inputSchema` 是什么？**
A: Zod 验证模式，用于验证用户输入的参数是否符合要求。

**Q: 什么是 `ToolUseContext`？**
A: 工具执行的上下文，包含：权限信息、AppState、AbortController 等。

**Q: 工具如何显示进度？**
A: 通过 `progress` 回调函数，实时更新 UI。

---

## 总结

| 概念 | 说明 |
|------|------|
| `Tool.ts` | 工具的"规格说明书" |
| `tools.ts` | 所有工具的"注册表" |
| `buildTool()` | 创建工具的"工厂函数" |
| `xxxTool.ts` | 每个工具的"实现" |
| `UI.tsx` | 工具的"外观" |
| `inputSchema` | 工具的"参数校验" |

记住：**工具 = 规格 + 实现 + 外观 + 参数校验**

---

*祝学习愉快！ 🚀*
