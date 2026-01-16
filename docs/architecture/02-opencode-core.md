# OpenCode 核心包架构

## 1. 概述

`packages/opencode` 是 OpenCode 的核心包，包含 CLI 应用程序、服务器逻辑、Agent 系统、Provider 集成等核心功能。

## 2. 目录结构

```
packages/opencode/src/
├── index.ts           # CLI 入口点 (Yargs)
├── acp/               # Agent Control Protocol
├── agent/             # Agent 代理系统
├── auth/              # 认证管理
├── bun/               # Bun 运行时工具
├── bus/               # 事件总线系统
├── cli/               # CLI 命令和 TUI
│   ├── cmd/           # CLI 命令实现
│   └── bootstrap.ts   # 启动引导
├── command/           # 命令执行框架
├── config/            # 配置管理
├── env/               # 环境变量
├── file/              # 文件系统操作
├── flag/              # 功能标志
├── format/            # 格式化工具
├── global/            # 全局状态
├── id/                # ID 生成
├── ide/               # IDE 集成
├── installation/      # 安装管理
├── lsp/               # 语言服务器协议
├── mcp/               # Model Context Protocol
├── patch/             # 代码补丁
├── permission/        # 权限系统
├── plugin/            # 插件系统
├── project/           # 项目管理
├── provider/          # AI Provider 系统
├── pty/               # PTY 终端
├── question/          # 问题交互
├── server/            # HTTP/WebSocket 服务器
├── session/           # 会话管理
├── share/             # 共享功能
├── shell/             # Shell 集成
├── skill/             # 技能系统
├── snapshot/          # 快照系统
├── storage/           # 存储抽象
├── tool/              # 工具系统
├── util/              # 工具函数
└── worktree/          # Git Worktree 管理
```

## 3. 核心模块架构图

```mermaid
graph TB
    subgraph "入口层 Entry Layer"
        INDEX[index.ts<br/>Yargs CLI]
        BOOTSTRAP[bootstrap.ts<br/>初始化]
    end

    subgraph "CLI 命令层"
        RUN[run.ts<br/>执行命令]
        SERVE[serve.ts<br/>启动服务]
        TUI[tui/<br/>终端界面]
        AUTH_CMD[auth.ts<br/>认证命令]
        MCP_CMD[mcp.ts<br/>MCP 命令]
    end

    subgraph "服务层 Service Layer"
        SERVER[server.ts<br/>HTTP/WS 服务器]
        SESSION[session/<br/>会话管理]
        AGENT[agent/<br/>代理系统]
    end

    subgraph "业务逻辑层"
        PROVIDER[provider/<br/>AI 提供商]
        TOOL[tool/<br/>工具系统]
        PERMISSION[permission/<br/>权限控制]
        COMMAND[command/<br/>命令执行]
    end

    subgraph "基础设施层"
        CONFIG[config/<br/>配置]
        STORAGE[storage/<br/>存储]
        BUS[bus/<br/>事件总线]
        FILE[file/<br/>文件系统]
        PTY[pty/<br/>终端]
    end

    INDEX --> BOOTSTRAP
    INDEX --> RUN
    INDEX --> SERVE
    INDEX --> TUI
    INDEX --> AUTH_CMD
    INDEX --> MCP_CMD

    RUN --> BOOTSTRAP
    SERVE --> SERVER
    TUI --> SERVER

    SERVER --> SESSION
    SESSION --> AGENT
    AGENT --> PROVIDER
    AGENT --> TOOL
    AGENT --> PERMISSION

    TOOL --> COMMAND
    TOOL --> FILE
    TOOL --> PTY

    SESSION --> CONFIG
    SESSION --> STORAGE
    SERVER --> BUS
```

## 4. CLI 命令执行流程

```mermaid
sequenceDiagram
    participant U as 用户
    participant Y as Yargs
    participant B as Bootstrap
    participant S as Server
    participant SE as Session
    participant A as Agent

    U->>Y: opencode run "message"
    Y->>Y: 解析参数
    Y->>B: 调用 RunCommand
    B->>B: 初始化环境
    B->>B: 加载配置
    B->>B: 初始化 Provider
    B->>S: 创建服务器
    S->>SE: 创建会话
    SE->>A: 选择 Agent
    A->>A: 执行对话循环
    A-->>U: 流式返回响应
```

## 5. 主要 CLI 命令

| 命令 | 文件 | 功能 |
|------|------|------|
| `run` | `cli/cmd/run.ts` | 执行单次对话 |
| `serve` | `cli/cmd/serve.ts` | 启动 HTTP 服务器 |
| `tui` | `cli/cmd/tui/` | 启动终端界面 |
| `auth` | `cli/cmd/auth.ts` | 认证管理 |
| `agent` | `cli/cmd/agent.ts` | Agent 管理 |
| `mcp` | `cli/cmd/mcp.ts` | MCP 服务器管理 |
| `pr` | `cli/cmd/pr.ts` | PR 处理 |
| `session` | `cli/cmd/session.ts` | 会话管理 |
| `models` | `cli/cmd/models.ts` | 模型列表 |
| `github` | `cli/cmd/github.ts` | GitHub 集成 |

## 6. 服务器架构

```mermaid
graph TB
    subgraph "Server (Hono)"
        CORS[CORS 中间件]
        AUTH[Basic Auth]
        LOG[日志中间件]

        subgraph "路由"
            HEALTH[/global/health]
            EVENT[/global/event<br/>SSE]
            SESSION_API[/session/*<br/>会话 API]
            TOOL_API[/tool/*<br/>工具 API]
            PROVIDER_API[/provider/*<br/>Provider API]
            PROJECT_API[/project/*<br/>项目 API]
            WS[WebSocket<br/>实时通信]
        end
    end

    CLIENT[客户端<br/>CLI/TUI/Web] --> CORS
    CORS --> AUTH
    AUTH --> LOG
    LOG --> HEALTH
    LOG --> EVENT
    LOG --> SESSION_API
    LOG --> TOOL_API
    LOG --> PROVIDER_API
    LOG --> PROJECT_API
    LOG --> WS
```

## 7. 事件总线系统

```mermaid
graph LR
    subgraph "事件发布者"
        SESSION_PUB[Session]
        AGENT_PUB[Agent]
        TOOL_PUB[Tool]
    end

    BUS[Bus<br/>事件总线]

    subgraph "事件订阅者"
        SERVER_SUB[Server<br/>SSE 推送]
        TUI_SUB[TUI<br/>界面更新]
        LOG_SUB[Logger<br/>日志记录]
    end

    SESSION_PUB --> BUS
    AGENT_PUB --> BUS
    TOOL_PUB --> BUS

    BUS --> SERVER_SUB
    BUS --> TUI_SUB
    BUS --> LOG_SUB
```

## 8. 项目管理模块

```mermaid
graph TB
    subgraph "Project Module"
        PROJECT[project.ts<br/>项目定义]
        INSTANCE[instance.ts<br/>项目实例]
        BOOTSTRAP_P[bootstrap.ts<br/>项目初始化]
        VCS[vcs.ts<br/>版本控制]
    end

    subgraph "项目资源"
        CONFIG_FILE[.opencode/config.json]
        AGENT_FILE[.opencode/agent.ts]
        SESSIONS[.opencode/sessions/]
        SNAPSHOTS[.opencode/snapshots/]
    end

    PROJECT --> INSTANCE
    BOOTSTRAP_P --> INSTANCE
    INSTANCE --> VCS

    INSTANCE --> CONFIG_FILE
    INSTANCE --> AGENT_FILE
    INSTANCE --> SESSIONS
    INSTANCE --> SNAPSHOTS
```

## 9. 存储系统

```mermaid
graph TB
    subgraph "Storage Abstraction"
        STORAGE_IF[Storage Interface]
    end

    subgraph "存储实现"
        FILE_STORAGE[文件存储<br/>.opencode/]
        MEMORY[内存存储]
    end

    subgraph "存储内容"
        SESSIONS_S[会话数据]
        SNAPSHOTS_S[快照数据]
        CONFIG_S[配置数据]
        AUTH_S[认证数据]
    end

    STORAGE_IF --> FILE_STORAGE
    STORAGE_IF --> MEMORY

    FILE_STORAGE --> SESSIONS_S
    FILE_STORAGE --> SNAPSHOTS_S
    FILE_STORAGE --> CONFIG_S
    FILE_STORAGE --> AUTH_S
```

## 10. TUI 架构

```mermaid
graph TB
    subgraph "TUI (OpenTUI + SolidJS)"
        APP[app.tsx<br/>主应用]

        subgraph "Context 层"
            ARGS_CTX[ArgsContext]
            SDK_CTX[SDKContext]
            SYNC_CTX[SyncContext]
            THEME_CTX[ThemeContext]
            KEYBIND_CTX[KeybindContext]
        end

        subgraph "组件层"
            PROMPT[Prompt<br/>输入组件]
            DIALOG[Dialog<br/>对话框]
            BORDER[Border<br/>边框]
            TODO_ITEM[TodoItem<br/>任务项]
        end

        subgraph "路由"
            HOME[Home<br/>主页]
            SESSION_PAGE[Session<br/>会话页]
        end
    end

    APP --> ARGS_CTX
    APP --> SDK_CTX
    APP --> SYNC_CTX
    APP --> THEME_CTX
    APP --> KEYBIND_CTX

    ARGS_CTX --> HOME
    SDK_CTX --> SESSION_PAGE

    SESSION_PAGE --> PROMPT
    SESSION_PAGE --> DIALOG
    HOME --> BORDER
    SESSION_PAGE --> TODO_ITEM
```

## 11. 初始化流程

```mermaid
flowchart TD
    START[启动 OpenCode]

    subgraph "初始化阶段"
        PARSE[解析 CLI 参数]
        LOG_INIT[初始化日志系统]
        ENV_LOAD[加载环境变量]
        CONFIG_LOAD[加载配置文件]
        AUTH_INIT[初始化认证]
        PROVIDER_INIT[初始化 Provider]
        PLUGIN_LOAD[加载插件]
    end

    subgraph "运行阶段"
        CMD_DISPATCH[命令分发]
        SERVER_START[启动服务器]
        SESSION_CREATE[创建会话]
    end

    START --> PARSE
    PARSE --> LOG_INIT
    LOG_INIT --> ENV_LOAD
    ENV_LOAD --> CONFIG_LOAD
    CONFIG_LOAD --> AUTH_INIT
    AUTH_INIT --> PROVIDER_INIT
    PROVIDER_INIT --> PLUGIN_LOAD
    PLUGIN_LOAD --> CMD_DISPATCH
    CMD_DISPATCH --> SERVER_START
    SERVER_START --> SESSION_CREATE
```

## 12. 关键依赖

| 依赖 | 用途 |
|------|------|
| `yargs` | CLI 参数解析 |
| `hono` | HTTP 服务器框架 |
| `zod` | 数据验证 |
| `ai` (Vercel AI SDK) | AI 模型集成 |
| `tree-sitter` | 代码解析 |
| `bun-pty` | PTY 终端模拟 |
| `remeda` | 函数式工具库 |

## 13. 相关文档

- [会话与代理系统](./03-session-agent.md)
- [工具系统架构](./04-tool-system.md)
- [Provider 系统](./05-provider-system.md)
