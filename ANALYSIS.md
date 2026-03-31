# Claude Code 源码综合分析报告

> **说明**: 本文件是对 `claude-code-leaked` 仓库的完整技术分析，涵盖内容概述、运行逻辑、模块架构及关键实现细节。

---

## 目录

1. [项目概述](#1-项目概述)
2. [目录结构](#2-目录结构)
3. [整体架构](#3-整体架构)
4. [启动流程](#4-启动流程)
5. [核心模块详解](#5-核心模块详解)
   - 5.1 [命令系统 (commands/)](#51-命令系统-commands)
   - 5.2 [工具系统 (tools/)](#52-工具系统-tools)
   - 5.3 [查询引擎 (QueryEngine)](#53-查询引擎-queryengine)
   - 5.4 [权限系统](#54-权限系统)
   - 5.5 [状态管理 (state/)](#55-状态管理-state)
   - 5.6 [服务层 (services/)](#56-服务层-services)
   - 5.7 [UI 层 (components/)](#57-ui-层-components)
   - 5.8 [Bridge 系统 (bridge/)](#58-bridge-系统-bridge)
   - 5.9 [MCP 集成](#59-mcp-集成)
   - 5.10 [记忆系统 (memdir/)](#510-记忆系统-memdir)
6. [主要设计模式](#6-主要设计模式)
7. [认证与安全](#7-认证与安全)
8. [性能优化策略](#8-性能优化策略)
9. [Feature Flags 与编译优化](#9-feature-flags-与编译优化)
10. [技术栈总览](#10-技术栈总览)
11. [总结](#11-总结)

---

## 1. 项目概述

**Claude Code** 是 Anthropic 官方出品的 AI 软件工程 CLI 工具，其源码通过 npm 注册表的 source map 文件泄露（2026 年 3 月 31 日）。

### 基本数据

| 项目 | 数值 |
|------|------|
| 编程语言 | TypeScript（严格模式） |
| 运行时 | Bun（二进制可执行文件） |
| 总代码量 | ~380,000 行 |
| 总文件数 | ~1,884 个 TypeScript/TSX 文件 |
| 终端 UI 框架 | React 18 + Ink |
| API 集成 | Anthropic Claude API |
| Slash 命令数 | ~80 个 |
| 工具数量 | ~40 个 |
| UI 组件数 | ~140 个 |

### 功能概览

Claude Code 是一个在终端中运行的 AI 编程助手，支持：

- **Shell 执行**：运行 Bash 命令
- **文件操作**：读、写、编辑文件，支持图片/PDF/Jupyter Notebook
- **代码搜索**：基于 ripgrep 的 Glob/Grep 工具
- **Web 访问**：网页内容抓取、网络搜索
- **AI 子代理**：生成和管理子 Agent
- **Git 集成**：commit、PR review 等
- **IDE 集成**：VS Code、JetBrains 桥接模式
- **MCP 协议**：接入任意 MCP 服务器
- **持久化记忆**：跨会话保存用户偏好和项目上下文
- **任务系统**：创建、分配、追踪 AI 任务

---

## 2. 目录结构

```
src/
├── main.tsx                         # CLI 入口（Commander.js 解析参数）
├── commands.ts                      # 命令注册表
├── tools.ts                         # 工具注册表与工厂函数
├── Tool.ts                          # 工具基类类型定义
├── QueryEngine.ts                   # LLM 推理引擎
├── query.ts                         # 查询管线与流式处理
├── context.ts                       # 系统/用户上下文收集
├── cost-tracker.ts                  # Token 用量与费用追踪
├── setup.ts                         # 会话初始化
├── history.ts                       # 对话历史管理
│
├── commands/                        # ~80 个 slash 命令
│   ├── commit.ts                    # Git commit 生成
│   ├── review.ts                    # 代码审查
│   ├── compact/                     # 上下文压缩
│   ├── config/                      # 配置管理
│   ├── doctor/                      # 环境诊断
│   ├── bridge/                      # IDE 桥接模式
│   ├── agents/                      # 子代理管理
│   ├── mcp/                         # MCP 服务器管理
│   ├── login/ / logout/             # OAuth 认证
│   ├── memory/                      # 持久化记忆
│   ├── skills/                      # Skill 管理
│   ├── tasks/                       # 任务管理
│   ├── vim/                         # Vim 模式切换
│   └── [其余 60+ 命令]
│
├── tools/                           # ~40 个工具实现
│   ├── BashTool/                    # Shell 命令执行
│   ├── FileReadTool/                # 文件读取（含图片/PDF/Notebook）
│   ├── FileWriteTool/               # 文件创建
│   ├── FileEditTool/                # 文件搜索/替换修改
│   ├── GlobTool/                    # 文件 Glob 模式匹配
│   ├── GrepTool/                    # ripgrep 代码搜索
│   ├── WebFetchTool/                # URL 内容抓取
│   ├── WebSearchTool/               # 网络搜索
│   ├── AgentTool/                   # 子代理生成
│   ├── SkillTool/                   # Skill 执行
│   ├── MCPTool/                     # MCP 工具调用
│   ├── LSPTool/                     # 语言服务协议
│   ├── NotebookEditTool/            # Jupyter Notebook 编辑
│   ├── TaskCreateTool/              # 任务创建
│   ├── EnterPlanModeTool/           # 进入 Plan 模式
│   ├── EnterWorktreeTool/           # Git Worktree 隔离
│   ├── ToolSearchTool/              # 延迟工具发现
│   ├── AskUserQuestionTool/         # 交互式提问
│   ├── SendMessageTool/             # 代理间通信
│   ├── RemoteTriggerTool/           # 远程触发器
│   ├── ScheduleCronTool/            # 定时任务
│   ├── TodoWriteTool/               # Todo 管理
│   ├── SleepTool/                   # 主动模式等待
│   ├── PowerShellTool/              # Windows PowerShell
│   └── [10+ 更多工具]
│
├── components/                      # ~140 个 React/Ink 组件
│   ├── App.tsx                      # 根组件
│   ├── REPL.tsx                     # 交互会话主界面
│   ├── PromptInput.tsx              # 用户输入处理
│   ├── messages/                    # 消息渲染
│   ├── agents/                      # 代理 UI
│   ├── tasks/                       # 任务 UI
│   ├── permissions/                 # 权限确认对话框
│   ├── diff/                        # Diff 可视化
│   ├── design-system/               # 主题与设计标记
│   └── [更多组件]
│
├── services/                        # 核心服务层
│   ├── api/claude.ts                # Anthropic API 客户端
│   ├── mcp/                         # MCP 协议实现
│   ├── analytics/                   # 分析与 A/B 测试
│   ├── compact/                     # 上下文压缩算法
│   ├── oauth/                       # OAuth 2.0 流程
│   ├── plugins/                     # 插件系统
│   ├── lsp/                         # 语言服务协议
│   ├── policyLimits/                # 组织策略执行
│   └── extractMemories/             # 记忆提取
│
├── bridge/                          # IDE 集成桥接
│   ├── bridgeMain.ts                # 桥接主循环
│   ├── bridgeMessaging.ts           # 协议定义
│   ├── sessionRunner.ts             # 会话执行
│   └── jwtUtils.ts                  # JWT 认证
│
├── state/                           # 状态管理
│   ├── AppStateStore.ts             # 状态类型定义
│   ├── AppState.tsx                 # React Context Provider
│   └── store.ts                     # 自定义 Store
│
├── skills/                          # Skill 系统
├── plugins/                         # 插件系统
├── tasks/                           # 任务类型（DreamTask/LocalAgentTask 等）
├── coordinator/                     # 多代理协调
├── memdir/                          # 持久化记忆文件
├── vim/                             # Vim 键位实现
├── voice/                           # 语音输入（实验性）
├── remote/                          # 远程会话
├── cli/transports/                  # WebSocket/SSE/Hybrid 传输层
├── ink/                             # Ink 渲染器（含 Yoga 布局引擎）
├── entrypoints/                     # 多入口（CLI/MCP/SDK）
├── utils/                           # 工具函数（git/auth/permissions 等）
├── types/                           # TypeScript 类型定义
├── constants/                       # 常量（系统提示词/Beta Header 等）
├── schemas/                         # Zod 验证模式
├── migrations/                      # 配置迁移脚本
├── native-ts/                       # 原生绑定（Yoga 布局/文件索引）
├── buddy/                           # 伴侣精灵彩蛋 (CompanionSprite)
└── upstreamproxy/                   # 上游代理配置
```

---

## 3. 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户 / IDE                              │
└──────────────────────┬──────────────────────┬───────────────────┘
                       │ 终端输入              │ Bridge 协议
                       ▼                       ▼
          ┌────────────────────┐    ┌─────────────────────┐
          │  React + Ink UI    │    │  IDE Bridge          │
          │  (REPL.tsx, 等)    │    │  (VS Code/JetBrains) │
          └────────┬───────────┘    └──────────┬───────────┘
                   │                            │
                   └────────────┬───────────────┘
                                ▼
                   ┌────────────────────────┐
                   │     AppState (Context) │  ← React 状态中心
                   └────────────┬───────────┘
                                │
              ┌─────────────────┼──────────────────┐
              ▼                 ▼                   ▼
   ┌──────────────┐  ┌─────────────────┐  ┌──────────────────┐
   │ CommandSystem│  │  QueryEngine    │  │   Tool System    │
   │ (~80 cmds)   │  │  (LLM Loop)     │  │   (~40 tools)    │
   └──────────────┘  └────────┬────────┘  └──────┬───────────┘
                              │                   │
                              ▼                   ▼
                   ┌─────────────────┐  ┌──────────────────┐
                   │  Anthropic API  │  │ Permission System │
                   │  (claude.ts)    │  │ (ask/allow/deny)  │
                   └─────────────────┘  └──────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
 ┌──────────────┐  ┌──────────────────┐  ┌──────────────────┐
 │  MCP Service │  │  Analytics &     │  │  Compact Service │
 │  (MCP Proto) │  │  Feature Flags   │  │  (上下文压缩)     │
 └──────────────┘  └──────────────────┘  └──────────────────┘
```

---

## 4. 启动流程

### 4.1 入口文件 `main.tsx`

CLI 启动遵循性能优化的并行化策略：

```
阶段 1 — 模块加载前并行预取（减少冷启动延迟）
   ├── startMdmRawRead()         → 读取 MDM（设备管理）配置
   └── startKeychainPrefetch()   → 从 macOS Keychain 预取 OAuth Token / API Key

阶段 2 — 模块加载
   ├── Commander.js              → CLI 参数解析
   ├── Anthropic SDK             → API 客户端
   ├── React / Ink               → 终端 UI
   └── 延迟加载 OpenTelemetry / gRPC（~1.1MB，避免阻塞）

阶段 3 — 初始化 init()
   ├── 校验配置文件
   ├── 设置 OpenTelemetry 遥测
   ├── 初始化 GrowthBook 特性标志
   ├── 执行配置迁移（migrateOpusToOpus1m 等）
   └── 并行连接 Anthropic API

阶段 4 — 执行
   ├── 交互模式：渲染 REPL UI（React + Ink）
   └── 非交互模式：直接执行命令后退出
```

### 4.2 会话初始化 `setup.ts`

```
1. 加载 ~/.claude/settings.json
2. 加载项目级 .claude/config.json
3. 连接 MCP 服务器
4. 加载 Plugins / Skills
5. 加载记忆文件（memdir）
6. 构建系统提示词（context.ts）
7. 启动会话主循环
```

---

## 5. 核心模块详解

### 5.1 命令系统 (commands/)

**注册模式**：`commands.ts` 集中注册所有命令，每个命令是独立模块。

```typescript
// 每个命令的基本结构
export default {
  name: 'commit',
  description: '创建 Git commit',
  inputSchema: z.object({ message: z.string().optional() }),
  execute: async (input, context) => { ... }
}
```

**部分命令根据 Feature Flags 条件加载**（如 KAIROS、BRIDGE_MODE 模式专用命令）。

**常用命令示例**：

| 命令 | 功能 |
|------|------|
| `/commit` | AI 生成 commit 信息并提交 |
| `/review` | 代码或 PR 审查 |
| `/compact` | 压缩对话上下文 |
| `/config` | 编辑配置 |
| `/doctor` | 环境诊断 |
| `/memory` | 管理持久化记忆 |
| `/mcp` | 管理 MCP 服务器 |
| `/diff` | 查看文件变更 |
| `/cost` | 显示会话费用 |
| `/bridge` | IDE 桥接模式 |

---

### 5.2 工具系统 (tools/)

#### 基础类型 `Tool.ts`

```typescript
interface Tool {
  name: string
  description: string                // 提供给 Claude 理解工具用途
  inputSchema: ZodSchema             // 输入验证（Zod v4）
  permission: PermissionMode         // 'always' | 'ask' | 'never'
  execute(input, context): Promise<ToolResult>
  // 可选：流式进度状态
}
```

#### 工具工厂 `tools.ts`

根据 Feature Flags 和当前模式，`getTools()` 动态返回可用工具列表。

#### 关键工具实现

**BashTool**
- 在 CWD 执行 Shell 命令
- 捕获 stdout/stderr，支持超时
- 通过权限系统验证危险命令
- 支持流式输出

**FileReadTool**
- 支持纯文本（带行号偏移）
- 支持图片（PNG/JPEG/WebP/GIF，自动压缩）
- 支持 PDF（分页读取）
- 支持 Jupyter Notebook（提取 cells）
- 阻止读取 `/dev/zero`、`/proc` 等设备文件

**AgentTool**
- 生成子 Agent（本地或远程）
- 将当前工具集传递给子 Agent
- 处理 Agent 间消息转发和失败恢复

**MCPTool**
- 调用 MCP 服务器注册的工具
- 处理 OAuth 认证（elicitation）
- 大输出结果持久化存储

---

### 5.3 查询引擎 (QueryEngine)

这是整个项目的**核心推理循环**，驱动与 Claude API 的所有交互。

```typescript
// 配置结构
type QueryEngineConfig = {
  cwd: string
  tools: Tool[]
  commands: Command[]
  mcpClients: MCPServerConnection[]
  canUseTool: CanUseToolFn        // 权限检查函数
  getAppState: () => AppState
  setAppState: (fn) => void
  customSystemPrompt?: string
  maxTurns?: number
  maxBudgetUsd?: number
  thinkingConfig?: ThinkingConfig
  // ...
}
```

**主循环流程**：

```
1. 构建系统提示词
   ├── 收集 git 状态（当前分支、最近 commit）
   ├── 加载记忆文件（~/.claude/memory/）
   ├── 注入自定义系统提示词
   └── 加入 Anthropic 归因头

2. 准备用户消息
   ├── 处理附件（图片、文件引用）
   └── 注入相关记忆

3. 流式调用 Claude API
   ├── 发送 messages[]
   ├── 接收 streaming response
   └── 解析 tool_use blocks

4. 并行执行工具（StreamingToolExecutor）
   ├── 检查权限 → 执行工具
   └── 收集 tool_results

5. 将结果追加到 messages[]

6. 循环直到 stop_reason !== 'tool_use' 或达到 maxTurns
```

---

### 5.4 权限系统

位于 `utils/permissions/`，控制所有工具的执行授权。

**权限模式**：

| 模式 | 行为 |
|------|------|
| `default` | 首次使用时弹出确认框 |
| `auto` | ML 分类器自动判断（需 TRANSCRIPT_CLASSIFIER flag） |
| `plan` | 计划模式，只允许只读操作 |
| `bypassPermissions` | 跳过所有权限检查（危险） |

**权限检查流程**：

```
工具调用
    │
    ▼
检查规则匹配（always-allow / always-deny / always-ask）
    │
    ├── 匹配 always-allow → 直接执行
    ├── 匹配 always-deny  → 拒绝
    └── 未匹配 → 检查模式
                 ├── auto 模式 → 运行分类器
                 └── 其他模式 → 弹出确认对话框
                                 ├── 用户批准 → 执行
                                 └── 用户拒绝 → 返回错误
```

**文件系统权限**：
- 路径白名单/黑名单
- 只读 Scratchpad 模式
- 阻止写入系统目录

---

### 5.5 状态管理 (state/)

使用 React Context + 自定义 Store（非 Redux）管理全局状态。

```typescript
// AppState 主要字段（定义于 AppStateStore.ts）
type AppState = {
  messages: Message[]                    // 对话历史
  agents: AgentId[]                      // 活跃代理 ID
  agentStates: Map<AgentId, AgentState>  // 代理状态
  tasks: TaskState[]                     // 任务列表
  mcpClients: MCPServerConnection[]      // MCP 连接
  tools: Tool[]                          // 当前可用工具
  commands: Command[]                    // 当前可用命令
  currentPromptInput: string             // 当前输入框内容
  isGenerating: boolean                  // 是否正在生成
  toolPermissionContext: ...             // 权限上下文
  notifications: Notification[]          // 通知列表
  settings: SettingsJson                 // 用户配置
  theme: ThemeName                       // 主题
  fileStateCache: FileStateCache         // 文件快照缓存
  sessionId: SessionId                   // 会话 ID
  model: ModelSetting                    // 当前模型
  effort: EffortValue                    // 思考力度
  speculation: SpeculationState          // 推测模式状态
  // 50+ 更多字段
}
```

**状态机（Speculation）**：

```typescript
type SpeculationState =
  | { status: 'idle' }
  | { status: 'active'; id: string; messagesRef: ... }
```

在用户空闲时在后台推测下一步操作，用户可接受或拒绝。

---

### 5.6 服务层 (services/)

#### API 服务 `services/api/claude.ts`
- Anthropic SDK 封装
- 流式响应处理
- Token 计数（调用 `/tokenize` API）
- 提示词缓存（cache_creation / cache_read tokens）
- 错误处理与重试逻辑
- 多模型支持（Opus 4.6 / Sonnet 4.6 / Haiku 等）

#### Compact 服务 `services/compact/`
- **自动触发条件**：Token 接近上限警告、消息数超阈值
- **压缩策略**：
  1. 保留最近 N 条消息
  2. 用 LLM 总结早期消息（包含 tool_use 摘要）
  3. 插入 `CompactBoundaryMessage` 标记边界

#### Analytics 服务 `services/analytics/`
- **GrowthBook**：特性标志与 A/B 测试
- **Datadog**：性能指标
- **OpenTelemetry**：第一方事件日志（gRPC 导出）
- **隐私保护**：`logForDiagnosticsNoPII()` 去除 PII 信息

#### MCP 服务 `services/mcp/client.ts`
- 管理多个 MCP 服务器连接（stdio/SSE/WebSocket/HTTP）
- 工具发现与动态注册
- 资源读取（ListMcpResources / ReadMcpResource）
- 提示词建议
- OAuth 认证代理

---

### 5.7 UI 层 (components/)

使用 **React + Ink**（React for CLI）构建终端 UI。

**核心组件树**：

```
App.tsx (AppStateProvider + 主题)
└── REPL.tsx (主会话界面)
    ├── PromptInput.tsx (用户输入处理，含 Vim 模式)
    ├── messages/ (消息渲染)
    │   ├── UserMessage.tsx
    │   ├── AssistantMessage.tsx (含 Markdown 渲染)
    │   ├── ToolUse.tsx (工具调用展示)
    │   └── ToolResult.tsx
    ├── agents/AgentProgressLine.tsx (代理状态)
    ├── tasks/TaskList.tsx (任务列表)
    ├── permissions/PermissionDialog.tsx (权限确认框)
    ├── diff/StructuredDiff.tsx (代码 diff 展示)
    └── Spinner.tsx / HighlightedCode.tsx / ...
```

**布局引擎**：使用 Yoga（Facebook 的 Flexbox 实现），在终端中实现精确的 Flexbox 布局。

---

### 5.8 Bridge 系统 (bridge/)

允许 IDE（VS Code、JetBrains）嵌入 Claude Code。

```
┌─────────────────┐         WebSocket / HTTP
│    IDE 插件      │ ◄──────────────────────── ┌──────────────────┐
│  (VS Code 等)    │                            │  bridgeMain.ts   │
└─────────────────┘                            │  (Bridge 主循环)  │
                                               └──────┬───────────┘
                                                      │
                                               ┌──────▼───────────┐
                                               │  sessionRunner   │
                                               │  (会话执行器)     │
                                               └──────┬───────────┘
                                                      │
                                               ┌──────▼───────────┐
                                               │  QueryEngine     │
                                               │  (复用主引擎)     │
                                               └──────────────────┘
```

- **协议**：`bridgeMessaging.ts` 定义双向消息格式
- **认证**：JWT（`jwtUtils.ts`），时间限制 Token
- **权限代理**：`bridgePermissionCallbacks.ts` 将权限请求转发给 IDE

---

### 5.9 MCP 集成

完整支持 **Model Context Protocol (MCP)**，允许接入任意 MCP 服务器。

```
Claude Code ◄──► MCP Client ◄──► MCP Server 1 (文件系统)
                              ◄──► MCP Server 2 (数据库)
                              ◄──► MCP Server 3 (浏览器)
                              ◄──► ...
```

**配置路径**：
- 全局：`~/.claude/mcp.json`
- 项目：`.claude/mcp.json`

**连接方式**：stdio（子进程）、SSE、WebSocket、HTTP

**延迟工具加载（ToolSearch）**：为避免工具列表过长（超过 API 限制），使用 `ToolSearchTool` 按需发现工具：

```typescript
// Claude 调用 ToolSearch 发现工具
const results = await toolSearch({ query: "file system operations" })
// 返回匹配工具的完整 Schema，之后才能调用
```

---

### 5.10 记忆系统 (memdir/)

**持久化记忆**跨会话保存，注入系统提示词。

- **存储位置**：`~/.claude/memory/`
- **文件格式**：Markdown（带 YAML frontmatter）
- **自动记忆**（`.auto.md` 文件）：LLM 从对话中提取关键信息自动保存
- **注入时机**：每次会话初始化时读取，加入系统提示词

---

## 6. 主要设计模式

### 6.1 并行预取启动优化

```typescript
// 在任何模块导入前启动 I/O
const mdmPromise = startMdmRawRead()
const keychainPromise = startKeychainPrefetch()

// 后续按需 await
const mdmConfig = await mdmPromise
```

### 6.2 延迟加载重模块

```typescript
// OpenTelemetry（~400KB）、gRPC（~700KB）延迟到需要时才加载
const initializeTelemetry = async () => {
  const { setupTelemetry } = await import('./telemetry.js')
  return setupTelemetry()
}
```

### 6.3 Zod Schema 验证

所有工具输入和配置文件均使用 Zod v4 进行运行时类型验证，防止无效输入传递到工具执行层。

### 6.4 Memoize 缓存

```typescript
// 系统上下文（git 状态、环境变量等）只收集一次
export const getSystemContext = memoize(async () => {
  const gitStatus = await getGitStatus()
  return { gitStatus, cacheBreaker, ... }
})
```

### 6.5 消息类型系统

```typescript
type Message =
  | UserMessage
  | AssistantMessage
  | SystemMessage
  | ToolUseMessage
  | ToolResultMessage
  | LocalCommandMessage
  | AttachmentMessage
  | ProgressMessage
  | TombstoneMessage
  | CompactBoundaryMessage
```

每种消息类型有精确的 TypeScript 类型定义，确保类型安全。

### 6.6 成本计算

```
总费用（USD）=
  input_tokens × input_price
  + output_tokens × output_price
  + cache_creation_tokens × cache_write_price
  + cache_read_tokens × cache_read_price
  × session_price_multiplier
```

---

## 7. 认证与安全

### 7.1 OAuth 2.0 流程

```
1. CLI 打开浏览器 → anthropic.com OAuth 页面
2. 用户授权
3. 重定向携带 auth code 回 CLI
4. CLI 用 code 换取 access/refresh token
5. Token 存储在 macOS Keychain（Linux: pass 或明文文件）
```

### 7.2 API Key 存储

| 平台 | 存储位置 |
|------|---------|
| macOS | Keychain（通过 `keytar`） |
| Linux | `pass` 密码管理器或 `~/.claude/` 明文 |
| Windows | Credential Manager |

### 7.3 Session JWT

- Bridge 模式使用时间限制 JWT
- 防止 `.claude/` 目录被盗后 token 被滥用
- 每次会话刷新

### 7.4 代码注入防护

- 所有工具输入经 Zod 验证
- BashTool 参数经过 Shell 转义
- 无 `eval()` 调用
- 路径遍历检测（阻止 `../../etc/passwd` 类型路径）

---

## 8. 性能优化策略

| 优化点 | 实现方式 |
|--------|---------|
| 冷启动加速 | MDM + Keychain 并行预取（在 import 前） |
| 重模块懒加载 | OpenTelemetry/gRPC 按需 `import()` |
| 系统上下文缓存 | `lodash.memoize` 缓存 git 状态等 |
| 并行工具执行 | `StreamingToolExecutor` 并行运行工具 |
| HTTP 响应缓存 | WebFetchTool 15 分钟 TTL 缓存 |
| 推测预执行 | 用户空闲时在后台推测下一步 |
| 死代码消除 | Bun build-time feature flags 剔除未用代码 |
| FPS 监控 | 追踪终端渲染帧率确保响应性 |
| 提示词缓存 | 利用 Anthropic 提示词缓存减少 Token 费用 |

---

## 9. Feature Flags 与编译优化

使用 Bun 的 `bun:bundle` API 在**编译时**按条件剔除死代码：

```typescript
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null  // 编译时剔除，减小二进制体积
```

**已知 Feature Flags**：

| Flag | 功能 |
|------|------|
| `PROACTIVE` | 主动 AI 模式 |
| `KAIROS` | Assistant/Brief 模式 |
| `BRIDGE_MODE` | IDE 集成 |
| `DAEMON` | 守护进程/服务器模式 |
| `VOICE_MODE` | 语音输入（实验性） |
| `AGENT_TRIGGERS` | 定时触发器 |
| `COORDINATOR_MODE` | 多代理协调 |
| `TRANSCRIPT_CLASSIFIER` | 自动权限分类 |
| `REACTIVE_COMPACT` | 响应式上下文压缩 |
| `CONTEXT_COLLAPSE` | 上下文折叠 |
| `EXPERIMENTAL_SKILL_SEARCH` | 技能搜索 |
| `UDS_INBOX` | Unix Domain Socket 消息传递 |

---

## 10. 技术栈总览

| 类别 | 技术 |
|------|------|
| **运行时** | Bun（TypeScript JIT，二进制打包） |
| **语言** | TypeScript 5.x（strict 模式） |
| **终端 UI** | React 18 + Ink |
| **布局引擎** | Yoga（Facebook Flexbox，原生绑定） |
| **CLI 解析** | Commander.js |
| **输入验证** | Zod v4 |
| **代码搜索** | ripgrep（子进程） |
| **API 客户端** | @anthropic-ai/sdk |
| **MCP 协议** | MCP SDK |
| **LSP 协议** | 语言服务协议 |
| **遥测** | OpenTelemetry + gRPC |
| **特性标志** | GrowthBook |
| **监控** | Datadog |
| **图片处理** | 自定义有损压缩 |
| **认证** | OAuth 2.0 + macOS Keychain + JWT |

---

## 11. 总结

Claude Code 是一个高度工程化的、生产级的 CLI AI 编程助手。其核心特点：

1. **规模庞大**：~1,900 个 TypeScript 文件，~380,000 行代码，体现了 Anthropic 在工具开发上的重大投入。

2. **架构清晰**：Query Engine（推理循环）→ Tool System（工具执行）→ Permission System（权限控制）→ Service Layer（外部服务）的分层架构设计合理。

3. **扩展性强**：通过 Plugin、Skill、MCP 三套机制支持第三方扩展，满足不同用户需求。

4. **工程严谨**：全面的 TypeScript 严格类型、Zod 运行时验证、完善的错误处理，以及从 Keychain 存储到 JWT 会话的安全体系。

5. **性能导向**：从并行预取启动到 Bun 编译时死代码消除，每个细节都体现了对性能的极致追求。

6. **多模态支持**：不仅处理文本，还能读取图片、PDF、Jupyter Notebook，体现了对实际工程场景的深度理解。

7. **隐藏功能**：代码中包含多个实验性/内测功能（语音输入、多代理协调、推测执行、伴侣精灵彩蛋等），展示了 Anthropic 未来的产品方向。

---

*本分析基于 2026-03-31 泄露的源码，源码通过 npm 注册表的 source map 文件获取。*
