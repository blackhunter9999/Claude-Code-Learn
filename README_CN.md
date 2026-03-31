# Claude Code 补档

> **2026 年 3 月 31 日，Anthropic 的 Claude Code CLI 完整源码通过其 npm registry 中暴露的 `.map` 文件被泄露。**

---

## 泄露过程

[Chaofan Shou (@Fried_rice)](https://x.com/Fried_rice) 发现了此次泄露并公开发帖：

> **“Claude code source code has been leaked via a map file in their npm registry!”**
>
> —— [@Fried_rice，2026-03-31](https://x.com/Fried_rice/status/2038894956459290963)

已发布 npm 包中的 source map 文件包含了对完整、未混淆 TypeScript 源码的引用，而这些源码可以从 Anthropic 的 R2 存储桶以 zip 压缩包形式下载。

---

## 概览

Claude Code 是 Anthropic 的官方 CLI 工具，可让你直接在终端与 Claude 交互，执行软件工程任务——编辑文件、运行命令、搜索代码库、管理 git 工作流等。

本仓库包含泄露的 `src/` 目录。

- **泄露日期**：2026-03-31
- **语言**：TypeScript
- **运行时**：Bun
- **终端 UI**：React + [Ink](https://github.com/vadimdemedes/ink)（CLI 场景下的 React）
- **规模**：约 1,900 个文件，512,000+ 行代码

---

## 目录结构

```
src/
├── main.tsx                 # 入口（基于 Commander.js 的 CLI 解析器）
├── commands.ts              # 命令注册表
├── tools.ts                 # 工具注册表
├── Tool.ts                  # 工具类型定义
├── QueryEngine.ts           # LLM 查询引擎（核心 Anthropic API 调用器）
├── context.ts               # 系统/用户上下文收集
├── cost-tracker.ts          # Token 成本跟踪
│
├── commands/                # 斜杠命令实现（约 50 个）
├── tools/                   # Agent 工具实现（约 40 个）
├── components/              # Ink UI 组件（约 140 个）
├── hooks/                   # React Hooks
├── services/                # 外部服务集成
├── screens/                 # 全屏 UI（Doctor、REPL、Resume）
├── types/                   # TypeScript 类型定义
├── utils/                   # 工具函数
│
├── bridge/                  # IDE 集成桥接层（VS Code、JetBrains）
├── coordinator/             # 多 Agent 协调器
├── plugins/                 # 插件系统
├── skills/                  # 技能系统
├── keybindings/             # 快捷键配置
├── vim/                     # Vim 模式
├── voice/                   # 语音输入
├── remote/                  # 远程会话
├── server/                  # 服务端模式
├── memdir/                  # 记忆目录（持久化记忆）
├── tasks/                   # 任务管理
├── state/                   # 状态管理
├── migrations/              # 配置迁移
├── schemas/                 # 配置 Schema（Zod）
├── entrypoints/             # 初始化逻辑
├── ink/                     # Ink 渲染器封装
├── buddy/                   # Companion 精灵（彩蛋）
├── native-ts/               # 原生 TypeScript 工具
├── outputStyles/            # 输出样式
├── query/                   # 查询管线
└── upstreamproxy/           # 代理配置
```

---

## 核心架构

### 1. 工具体系（`src/tools/`）

Claude Code 可调用的每个工具都以独立模块实现。每个工具都会定义输入 Schema、权限模型与执行逻辑。

| 工具 | 说明 |
|---|---|
| `BashTool` | Shell 命令执行 |
| `FileReadTool` | 文件读取（图片、PDF、notebook） |
| `FileWriteTool` | 文件创建/覆盖 |
| `FileEditTool` | 文件局部修改（字符串替换） |
| `GlobTool` | 文件模式匹配搜索 |
| `GrepTool` | 基于 ripgrep 的内容检索 |
| `WebFetchTool` | 获取 URL 内容 |
| `WebSearchTool` | Web 搜索 |
| `AgentTool` | 启动子 Agent |
| `SkillTool` | 执行技能 |
| `MCPTool` | 调用 MCP 服务器工具 |
| `LSPTool` | Language Server Protocol 集成 |
| `NotebookEditTool` | Jupyter Notebook 编辑 |
| `TaskCreateTool` / `TaskUpdateTool` | 任务创建与管理 |
| `SendMessageTool` | Agent 间消息通信 |
| `TeamCreateTool` / `TeamDeleteTool` | 团队 Agent 管理 |
| `EnterPlanModeTool` / `ExitPlanModeTool` | Plan 模式切换 |
| `EnterWorktreeTool` / `ExitWorktreeTool` | Git worktree 隔离 |
| `ToolSearchTool` | 延迟工具发现 |
| `CronCreateTool` | 定时触发创建 |
| `RemoteTriggerTool` | 远程触发 |
| `SleepTool` | 主动模式等待 |
| `SyntheticOutputTool` | 结构化输出生成 |

### 2. 命令系统（`src/commands/`）

面向用户的斜杠命令，以 `/` 前缀调用。

| 命令 | 说明 |
|---|---|
| `/commit` | 创建 git 提交 |
| `/review` | 代码审查 |
| `/compact` | 上下文压缩 |
| `/mcp` | MCP 服务器管理 |
| `/config` | 设置管理 |
| `/doctor` | 环境诊断 |
| `/login` / `/logout` | 身份认证 |
| `/memory` | 持久化记忆管理 |
| `/skills` | 技能管理 |
| `/tasks` | 任务管理 |
| `/vim` | Vim 模式切换 |
| `/diff` | 查看变更 |
| `/cost` | 查看使用成本 |
| `/theme` | 切换主题 |
| `/context` | 上下文可视化 |
| `/pr_comments` | 查看 PR 评论 |
| `/resume` | 恢复上次会话 |
| `/share` | 分享会话 |
| `/desktop` | 切换到桌面应用 |
| `/mobile` | 切换到移动应用 |

### 3. 服务层（`src/services/`）

| 服务 | 说明 |
|---|---|
| `api/` | Anthropic API 客户端、文件 API、bootstrap |
| `mcp/` | Model Context Protocol 服务器连接与管理 |
| `oauth/` | OAuth 2.0 认证流程 |
| `lsp/` | Language Server Protocol 管理器 |
| `analytics/` | 基于 GrowthBook 的特性开关与分析 |
| `plugins/` | 插件加载器 |
| `compact/` | 对话上下文压缩 |
| `policyLimits/` | 组织策略限制 |
| `remoteManagedSettings/` | 远程托管设置 |
| `extractMemories/` | 自动提取记忆 |
| `tokenEstimation.ts` | Token 数量估算 |
| `teamMemorySync/` | 团队记忆同步 |

### 4. Bridge 系统（`src/bridge/`）

一个双向通信层，用于连接 IDE 扩展（VS Code、JetBrains）与 Claude Code CLI。

- `bridgeMain.ts` —— Bridge 主循环
- `bridgeMessaging.ts` —— 消息协议
- `bridgePermissionCallbacks.ts` —— 权限回调
- `replBridge.ts` —— REPL 会话桥接
- `jwtUtils.ts` —— 基于 JWT 的认证
- `sessionRunner.ts` —— 会话执行管理

### 5. 权限系统（`src/hooks/toolPermission/`）

在每次工具调用时进行权限检查。会提示用户批准/拒绝，或根据配置的权限模式自动处理（`default`、`plan`、`bypassPermissions`、`auto` 等）。

### 6. Feature Flags

通过 Bun 的 `bun:bundle` feature flags 实现死代码消除：

```typescript
import { feature } from 'bun:bundle'

// 不活跃代码会在构建阶段被完全剔除
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

显著标记：`PROACTIVE`、`KAIROS`、`BRIDGE_MODE`、`DAEMON`、`VOICE_MODE`、`AGENT_TRIGGERS`、`MONITOR_TOOL`

---

## 关键文件详解

### `QueryEngine.ts`（约 46K 行）

LLM API 调用的核心引擎。处理流式响应、工具调用循环、thinking 模式、重试逻辑与 token 计数。

### `Tool.ts`（约 29K 行）

定义所有工具的基础类型与接口——输入 Schema、权限模型与进度状态类型。

### `commands.ts`（约 25K 行）

管理全部斜杠命令的注册和执行。使用条件导入以在不同环境加载不同命令集。

### `main.tsx`

基于 Commander.js 的 CLI 解析器 + React/Ink 渲染器初始化。启动时会并行处理 MDM 设置、keychain 预取和 GrowthBook 初始化，以提升冷启动速度。

---

## 技术栈

| 类别 | 技术 |
|---|---|
| Runtime | [Bun](https://bun.sh) |
| Language | TypeScript（strict） |
| Terminal UI | [React](https://react.dev) + [Ink](https://github.com/vadimdemedes/ink) |
| CLI Parsing | [Commander.js](https://github.com/tj/commander.js)（extra-typings） |
| Schema Validation | [Zod v4](https://zod.dev) |
| Code Search | [ripgrep](https://github.com/BurntSushi/ripgrep)（通过 GrepTool） |
| Protocols | [MCP SDK](https://modelcontextprotocol.io)、LSP |
| API | [Anthropic SDK](https://docs.anthropic.com) |
| Telemetry | OpenTelemetry + gRPC |
| Feature Flags | GrowthBook |
| Auth | OAuth 2.0、JWT、macOS Keychain |

---

## 值得注意的设计模式

### 并行预取（Parallel Prefetch）

通过并行预取 MDM 设置、Keychain 读取和 API 预连接来优化启动时间——在重量级模块求值前就开始。

```typescript
// main.tsx —— 作为副作用在其他 import 前触发
startMdmRawRead()
startKeychainPrefetch()
```

### 延迟加载（Lazy Loading）

重型模块（OpenTelemetry ~400KB，gRPC ~700KB）通过动态 `import()` 延迟到真正需要时再加载。

### Agent Swarms

子 Agent 通过 `AgentTool` 启动，`coordinator/` 负责多 Agent 编排。`TeamCreateTool` 支持团队级并行协作。

### Skill System

可复用工作流定义在 `skills/` 中，并通过 `SkillTool` 执行。用户可添加自定义技能。

### Plugin Architecture

内置与第三方插件都通过 `plugins/` 子系统加载。

---

## 免责声明

本仓库归档了于 **2026-03-31** 从 Anthropic npm registry 泄露的源码。所有原始源码版权归 [Anthropic](https://www.anthropic.com) 所有。

仅限于学习使用。

最后编辑：
Beamus Wayne
2026年3月31日

# Claude-Code-Learn
