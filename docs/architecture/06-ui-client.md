# UI 与客户端架构

## 1. 概述

OpenCode 提供多种客户端形态：CLI、TUI (终端界面)、Web App 和 Desktop App。它们共享核心 UI 组件库和 SDK，通过统一的后端服务进行通信。

## 2. 客户端类型

```mermaid
graph TB
    subgraph "客户端形态"
        CLI[CLI<br/>packages/opencode]
        TUI[TUI<br/>packages/opencode/cli/cmd/tui]
        WEB[Web App<br/>packages/app]
        DESKTOP[Desktop<br/>packages/desktop]
    end

    subgraph "共享层"
        UI[UI 组件库<br/>packages/ui]
        SDK[SDK<br/>packages/sdk]
    end

    subgraph "后端"
        SERVER[Server<br/>HTTP + WebSocket]
    end

    CLI --> SERVER
    TUI --> SERVER
    WEB --> UI
    WEB --> SDK
    DESKTOP --> WEB
    SDK --> SERVER
```

## 3. packages/app 结构

```
packages/app/src/
├── app.tsx              # 主应用组件
├── entry.tsx            # 入口点
├── index.ts             # 导出
├── components/          # 页面组件
│   ├── dialog-*.tsx     # 对话框组件
│   ├── prompt-input.tsx # 输入组件
│   ├── session/         # 会话组件
│   ├── terminal.tsx     # 终端组件
│   └── titlebar.tsx     # 标题栏
├── context/             # 全局状态
│   ├── global-sync.tsx  # 全局同步
│   ├── sdk.tsx          # SDK Context
│   ├── layout.tsx       # 布局
│   ├── permission.tsx   # 权限
│   ├── prompt.tsx       # 提示输入
│   ├── terminal.tsx     # 终端
│   └── ...
├── pages/               # 页面
│   ├── home.tsx         # 主页
│   ├── session.tsx      # 会话页
│   └── layout.tsx       # 布局
├── hooks/               # 自定义 Hooks
└── utils/               # 工具函数
```

## 4. packages/ui 结构

```
packages/ui/src/
├── components/          # UI 组件
│   ├── button.tsx       # 按钮
│   ├── dialog.tsx       # 对话框
│   ├── markdown.tsx     # Markdown 渲染
│   ├── diff.tsx         # Diff 显示
│   ├── code.tsx         # 代码高亮
│   ├── message-part.tsx # 消息部分
│   ├── session-turn.tsx # 会话轮次
│   └── ...
├── context/             # UI Context
│   ├── dialog.tsx       # 对话框上下文
│   ├── marked.tsx       # Markdown 上下文
│   ├── diff.tsx         # Diff 上下文
│   ├── code.tsx         # 代码上下文
│   └── worker-pool.tsx  # Worker 池
├── theme/               # 主题系统
│   ├── context.tsx      # 主题上下文
│   ├── color.ts         # 颜色定义
│   ├── default-themes.ts# 默认主题
│   └── types.ts         # 类型定义
├── pierre/              # Diff 渲染库
├── hooks/               # UI Hooks
└── assets/              # 静态资源
```

## 5. Web App 架构

```mermaid
graph TB
    subgraph "App 层"
        ENTRY[entry.tsx]
        APP[app.tsx]
    end

    subgraph "Context 层"
        GLOBAL_SYNC[GlobalSync<br/>全局事件同步]
        SDK_CTX[SDK Context<br/>API 访问]
        LAYOUT[Layout<br/>布局管理]
        PERMISSION[Permission<br/>权限管理]
        PROMPT[Prompt<br/>输入管理]
        TERMINAL[Terminal<br/>终端管理]
        FILE[File<br/>文件管理]
        COMMAND[Command<br/>命令管理]
    end

    subgraph "页面层"
        HOME[Home<br/>主页]
        SESSION[Session<br/>会话页]
    end

    subgraph "组件层"
        DIALOGS[Dialogs<br/>对话框]
        PROMPT_INPUT[PromptInput<br/>输入框]
        TERMINAL_COMP[Terminal<br/>终端]
        MESSAGES[Messages<br/>消息显示]
    end

    ENTRY --> APP
    APP --> GLOBAL_SYNC
    APP --> SDK_CTX
    APP --> LAYOUT
    APP --> PERMISSION
    APP --> PROMPT
    APP --> TERMINAL
    APP --> FILE
    APP --> COMMAND

    GLOBAL_SYNC --> HOME
    GLOBAL_SYNC --> SESSION

    SESSION --> DIALOGS
    SESSION --> PROMPT_INPUT
    SESSION --> TERMINAL_COMP
    SESSION --> MESSAGES
```

## 6. Context 系统详解

```mermaid
classDiagram
    class GlobalSyncContext {
        +sessions: Session[]
        +currentSession: Session
        +events: EventStream
        +subscribe(event) void
    }

    class SDKContext {
        +client: OpencodeClient
        +session: SessionAPI
        +provider: ProviderAPI
        +tool: ToolAPI
    }

    class LayoutContext {
        +sidebarOpen: boolean
        +panelLayout: PanelLayout
        +toggleSidebar() void
        +setPanel(config) void
    }

    class PermissionContext {
        +pending: PermissionRequest[]
        +approve(id) void
        +deny(id) void
    }

    class PromptContext {
        +value: string
        +files: File[]
        +submit() void
        +attach(file) void
    }

    class TerminalContext {
        +terminals: Terminal[]
        +active: Terminal
        +create() Terminal
        +close(id) void
    }

    GlobalSyncContext --> SDKContext
    LayoutContext --> PermissionContext
    PromptContext --> TerminalContext
```

## 7. TUI 架构

```mermaid
graph TB
    subgraph "TUI (OpenTUI + SolidJS)"
        APP_TUI[app.tsx]

        subgraph "TUI Context"
            ARGS[ArgsContext<br/>命令行参数]
            SDK_TUI[SDKContext<br/>API 客户端]
            SYNC_TUI[SyncContext<br/>状态同步]
            THEME_TUI[ThemeContext<br/>主题]
            KEYBIND[KeybindContext<br/>快捷键]
            LOCAL[LocalContext<br/>本地状态]
        end

        subgraph "TUI 组件"
            PROMPT_TUI[Prompt<br/>输入]
            DIALOG_TUI[Dialogs<br/>对话框]
            BORDER[Border<br/>边框]
            LOGO[Logo<br/>标志]
            TIPS[Tips<br/>提示]
            TODO[TodoItem<br/>任务]
        end

        subgraph "TUI 路由"
            ROUTE[RouteContext]
            HOME_TUI[Home<br/>主页]
            SESSION_TUI[Session<br/>会话]
        end
    end

    APP_TUI --> ARGS
    APP_TUI --> SDK_TUI
    APP_TUI --> SYNC_TUI
    APP_TUI --> THEME_TUI
    APP_TUI --> KEYBIND
    APP_TUI --> LOCAL

    ARGS --> ROUTE
    ROUTE --> HOME_TUI
    ROUTE --> SESSION_TUI

    SESSION_TUI --> PROMPT_TUI
    SESSION_TUI --> DIALOG_TUI
    HOME_TUI --> BORDER
    HOME_TUI --> LOGO
```

## 8. Desktop App 架构

```mermaid
graph TB
    subgraph "Tauri"
        RUST[Rust Backend]
        WEBVIEW[WebView]
    end

    subgraph "功能"
        NATIVE_FS[原生文件系统]
        NATIVE_DIALOG[原生对话框]
        NATIVE_NOTIFY[原生通知]
        AUTO_UPDATE[自动更新]
        WINDOW[窗口管理]
        SHELL[Shell 集成]
    end

    subgraph "Web App"
        APP_EMBEDDED[packages/app]
    end

    RUST --> NATIVE_FS
    RUST --> NATIVE_DIALOG
    RUST --> NATIVE_NOTIFY
    RUST --> AUTO_UPDATE
    RUST --> WINDOW
    RUST --> SHELL

    WEBVIEW --> APP_EMBEDDED
    RUST --> WEBVIEW
```

## 9. 组件库架构

```mermaid
graph TB
    subgraph "基础组件"
        BUTTON[Button]
        INPUT[TextField]
        CHECKBOX[Checkbox]
        RADIO[RadioGroup]
        SELECT[Select]
        SWITCH[Switch]
    end

    subgraph "布局组件"
        DIALOG_COMP[Dialog]
        POPOVER[Popover]
        TOOLTIP[Tooltip]
        DROPDOWN[DropdownMenu]
        TABS[Tabs]
        ACCORDION[Accordion]
    end

    subgraph "业务组件"
        MARKDOWN[Markdown]
        CODE[Code]
        DIFF_COMP[Diff]
        MESSAGE[MessagePart]
        TURN[SessionTurn]
        REVIEW[SessionReview]
    end

    subgraph "图标组件"
        ICON[Icon]
        FILE_ICON[FileIcon]
        PROVIDER_ICON[ProviderIcon]
        LOGO_COMP[Logo]
    end
```

## 10. 主题系统

```mermaid
flowchart TD
    subgraph "主题来源"
        DEFAULT[默认主题]
        BUILTIN[内置主题]
        CUSTOM[自定义主题]
    end

    subgraph "主题加载"
        LOADER[Theme Loader]
        RESOLVE[Theme Resolver]
    end

    subgraph "主题应用"
        CSS_VARS[CSS 变量]
        CONTEXT[Theme Context]
        COMPONENTS[组件样式]
    end

    DEFAULT --> LOADER
    BUILTIN --> LOADER
    CUSTOM --> LOADER

    LOADER --> RESOLVE
    RESOLVE --> CSS_VARS
    RESOLVE --> CONTEXT
    CSS_VARS --> COMPONENTS
```

### 10.1 内置主题

- Catppuccin (默认)
- Catppuccin Frappe
- Catppuccin Macchiato
- Nord
- Dracula
- Gruvbox
- One Dark
- GitHub
- Monokai
- Material
- 更多...

## 11. 数据流

```mermaid
sequenceDiagram
    participant U as 用户
    participant UI as UI 组件
    participant CTX as Context
    participant SDK as SDK Client
    participant WS as WebSocket
    participant S as Server

    U->>UI: 用户操作
    UI->>CTX: 更新状态
    CTX->>SDK: API 调用
    SDK->>S: HTTP 请求
    S-->>SDK: 响应

    S->>WS: 推送事件
    WS->>CTX: 更新状态
    CTX->>UI: 重新渲染
    UI-->>U: 显示更新
```

## 12. 消息渲染流程

```mermaid
flowchart TD
    MSG[Message 数据]
    PARSE[解析 Parts]

    subgraph "Part 类型"
        TEXT[Text<br/>Markdown 渲染]
        CODE[Code<br/>语法高亮]
        TOOL[ToolCall<br/>工具调用]
        RESULT[ToolResult<br/>工具结果]
        IMAGE[Image<br/>图片显示]
        FILE[File<br/>文件附件]
    end

    RENDER[组合渲染]
    OUTPUT[最终输出]

    MSG --> PARSE
    PARSE --> TEXT
    PARSE --> CODE
    PARSE --> TOOL
    PARSE --> RESULT
    PARSE --> IMAGE
    PARSE --> FILE

    TEXT --> RENDER
    CODE --> RENDER
    TOOL --> RENDER
    RESULT --> RENDER
    IMAGE --> RENDER
    FILE --> RENDER

    RENDER --> OUTPUT
```

## 13. 关键文件

### packages/app

| 文件 | 功能 |
|------|------|
| `app.tsx` | 主应用组件 |
| `context/global-sync.tsx` | 全局状态同步 |
| `context/sdk.tsx` | SDK Context |
| `pages/session.tsx` | 会话页面 |
| `components/prompt-input.tsx` | 输入组件 |

### packages/ui

| 文件 | 功能 |
|------|------|
| `components/markdown.tsx` | Markdown 渲染 |
| `components/diff.tsx` | Diff 显示 |
| `components/code.tsx` | 代码高亮 |
| `components/session-turn.tsx` | 会话轮次 |
| `theme/context.tsx` | 主题上下文 |

## 14. 相关文档

- [OpenCode 核心包架构](./02-opencode-core.md)
- [Console 后台管理系统](./07-console-system.md)
