# OpenCode 总体架构概览

## 1. 项目概述

OpenCode 是一个模块化的、基于 monorepo 的 AI 驱动开发工具，使用 Bun、TypeScript 和 SolidJS 构建。它采用分层架构，将 CLI、TUI、Web UI、后端服务和核心业务逻辑清晰分离。

## 2. 目录结构总览

```
opencode/
├── packages/               # Monorepo 工作区包
│   ├── opencode/           # 核心 CLI 应用 & 服务器逻辑
│   ├── console/            # 管理控制台 & 仪表板
│   ├── app/                # 共享 Web UI (SolidJS)
│   ├── desktop/            # Tauri 原生桌面应用
│   ├── web/                # Astro 文档站点
│   ├── ui/                 # UI 组件库 (SolidJS)
│   ├── sdk/                # 客户端/服务器 SDK
│   ├── plugin/             # 插件系统 & 接口
│   ├── util/               # 共享工具函数
│   └── ...
├── infra/                  # 基础设施即代码 (SST/Pulumi)
├── themes/                 # UI 主题
└── sdks/                   # 非 JS SDK
```

## 3. 核心架构图

```mermaid
graph TB
    subgraph "客户端层 Client Layer"
        CLI[CLI 命令行]
        TUI[TUI 终端界面]
        WEB[Web 应用]
        DESKTOP[桌面应用]
    end

    subgraph "服务层 Service Layer"
        SERVER[WebSocket/HTTP Server]
        AUTH[认证服务]
        SESSION[会话管理]
    end

    subgraph "业务逻辑层 Business Logic"
        AGENT[Agent 代理系统]
        PROVIDER[Provider 提供商系统]
        TOOL[Tool 工具系统]
        PERMISSION[Permission 权限系统]
    end

    subgraph "基础设施层 Infrastructure"
        STORAGE[存储系统]
        CONFIG[配置管理]
        MCP[MCP 服务器]
        PLUGIN[插件系统]
    end

    subgraph "外部服务 External Services"
        LLM[LLM 提供商<br/>Anthropic/OpenAI/Google...]
        DB[(数据库<br/>PlanetScale)]
        CF[Cloudflare<br/>Workers/Pages]
    end

    CLI --> SERVER
    TUI --> SERVER
    WEB --> SERVER
    DESKTOP --> WEB

    SERVER --> AUTH
    SERVER --> SESSION

    SESSION --> AGENT
    AGENT --> PROVIDER
    AGENT --> TOOL
    AGENT --> PERMISSION

    PROVIDER --> LLM
    TOOL --> STORAGE
    SESSION --> CONFIG
    TOOL --> MCP
    SESSION --> PLUGIN

    SERVER --> DB
    WEB --> CF
```

## 4. 请求处理流程

```mermaid
sequenceDiagram
    participant U as 用户
    participant C as CLI/TUI/Web
    participant S as Server
    participant A as Agent
    participant P as Provider
    participant T as Tool
    participant L as LLM

    U->>C: 输入命令/消息
    C->>S: WebSocket/HTTP 请求
    S->>S: 创建/恢复会话
    S->>A: 选择合适的 Agent
    A->>P: 获取 Provider 配置
    P->>L: 发送请求到 LLM
    L-->>A: 返回响应 (可能包含工具调用)

    loop 工具执行循环
        A->>T: 执行工具调用
        T-->>A: 返回工具结果
        A->>L: 发送工具结果
        L-->>A: 返回新响应
    end

    A-->>S: 返回最终响应
    S-->>C: 流式响应
    C-->>U: 显示结果
```

## 5. 多客户端统一架构

```mermaid
graph LR
    subgraph "前端客户端"
        A[CLI<br/>Yargs]
        B[TUI<br/>OpenTUI/SolidJS]
        C[Web App<br/>SolidJS]
        D[Desktop<br/>Tauri]
    end

    subgraph "共享层"
        E[packages/app<br/>共享 UI 组件]
        F[packages/ui<br/>基础组件库]
        G[packages/sdk<br/>API SDK]
    end

    subgraph "后端服务"
        H[Server<br/>Hono + WebSocket]
    end

    A --> H
    B --> E
    C --> E
    D --> C
    E --> F
    E --> G
    G --> H
```

## 6. 技术栈概览

| 层级 | 技术 |
|------|------|
| **运行时** | Bun 1.3.5 |
| **语言** | TypeScript 5.8 |
| **UI 框架** | SolidJS 1.9 |
| **样式** | Tailwind CSS v4 |
| **桌面** | Tauri v2 |
| **文档** | Astro + Starlight |
| **后端** | Hono (边缘函数) |
| **数据库** | PlanetScale (MySQL) + Drizzle ORM |
| **AI 集成** | Vercel AI SDK |
| **代码解析** | Tree-sitter |
| **终端** | PTY (bun-pty) |
| **构建** | Vite, esbuild, Turbo |
| **基础设施** | SST + Cloudflare |

## 7. 核心包职责

```mermaid
graph TB
    subgraph "packages/opencode"
        OC_CLI[cli/ - CLI 命令]
        OC_SERVER[server/ - 服务器]
        OC_AGENT[agent/ - 代理系统]
        OC_PROVIDER[provider/ - AI 提供商]
        OC_TOOL[tool/ - 工具系统]
        OC_SESSION[session/ - 会话管理]
    end

    subgraph "packages/app"
        APP_PAGES[pages/ - 页面组件]
        APP_COMP[components/ - 共享组件]
        APP_CTX[context/ - 全局状态]
    end

    subgraph "packages/ui"
        UI_COMP[components/ - 基础 UI]
        UI_THEME[theme/ - 主题系统]
        UI_PIERRE[pierre/ - Diff 显示]
    end

    subgraph "packages/console"
        CON_APP[app/ - Web 应用]
        CON_CORE[core/ - 后端逻辑]
        CON_FUNC[function/ - CF Workers]
    end

    subgraph "packages/sdk"
        SDK_CLIENT[client - 客户端 SDK]
        SDK_SERVER[server - 服务器 SDK]
    end

    OC_CLI --> OC_SERVER
    OC_SERVER --> OC_SESSION
    OC_SESSION --> OC_AGENT
    OC_AGENT --> OC_PROVIDER
    OC_AGENT --> OC_TOOL

    APP_PAGES --> APP_COMP
    APP_COMP --> UI_COMP
    APP_CTX --> SDK_CLIENT

    CON_APP --> CON_CORE
    CON_CORE --> SDK_SERVER
```

## 8. 数据流架构

```mermaid
flowchart TD
    subgraph "输入"
        I1[用户消息]
        I2[文件内容]
        I3[环境上下文]
    end

    subgraph "处理"
        P1[消息解析]
        P2[Agent 选择]
        P3[Prompt 构建]
        P4[LLM 调用]
        P5[工具执行]
    end

    subgraph "输出"
        O1[文本响应]
        O2[代码修改]
        O3[命令执行]
        O4[会话状态]
    end

    I1 --> P1
    I2 --> P3
    I3 --> P3
    P1 --> P2
    P2 --> P3
    P3 --> P4
    P4 --> P5
    P5 --> P4
    P4 --> O1
    P5 --> O2
    P5 --> O3
    P4 --> O4
```

## 9. 相关文档

- [OpenCode 核心包架构](./02-opencode-core.md)
- [会话与代理系统](./03-session-agent.md)
- [工具系统架构](./04-tool-system.md)
- [Provider 系统](./05-provider-system.md)
- [UI 与客户端架构](./06-ui-client.md)
- [Console 后台管理系统](./07-console-system.md)
