# 会话与代理系统架构

## 1. 概述

会话 (Session) 和代理 (Agent) 系统是 OpenCode 的核心，负责管理用户与 AI 之间的对话交互、任务执行和状态管理。

## 2. Agent 代理系统

### 2.1 Agent 类型

| Agent | 模式 | 描述 | 权限 |
|-------|------|------|------|
| `build` | primary | 默认代理，完整功能访问 | 完全访问 + 提问 |
| `plan` | primary | 只读规划代理 | 只读 + 计划退出 |
| `general` | subagent | 通用子代理，执行多步任务 | 无 Todo 访问 |
| `explore` | subagent | 代码探索专用代理 | 只读 + 搜索 |
| `compaction` | primary (hidden) | 上下文压缩代理 | 最小权限 |
| `title` | primary (hidden) | 标题生成代理 | 最小权限 |
| `summary` | primary (hidden) | 摘要生成代理 | 最小权限 |

### 2.2 Agent 架构图

```mermaid
graph TB
    subgraph "Agent 系统"
        AGENT_MGR[Agent Manager]

        subgraph "Primary Agents"
            BUILD[build<br/>主代理]
            PLAN[plan<br/>规划代理]
        end

        subgraph "Sub Agents"
            GENERAL[general<br/>通用子代理]
            EXPLORE[explore<br/>探索代理]
        end

        subgraph "Hidden Agents"
            COMPACTION[compaction<br/>压缩代理]
            TITLE[title<br/>标题代理]
            SUMMARY[summary<br/>摘要代理]
        end
    end

    AGENT_MGR --> BUILD
    AGENT_MGR --> PLAN
    AGENT_MGR --> GENERAL
    AGENT_MGR --> EXPLORE
    AGENT_MGR --> COMPACTION
    AGENT_MGR --> TITLE
    AGENT_MGR --> SUMMARY

    BUILD --> GENERAL
    BUILD --> EXPLORE
```

### 2.3 Agent 配置结构

```mermaid
classDiagram
    class AgentInfo {
        +string name
        +string description
        +enum mode: subagent|primary|all
        +boolean native
        +boolean hidden
        +number topP
        +number temperature
        +string color
        +Ruleset permission
        +Model model
        +string prompt
        +object options
        +number steps
    }

    class Model {
        +string modelID
        +string providerID
    }

    class Ruleset {
        +Rule[] rules
    }

    AgentInfo --> Model
    AgentInfo --> Ruleset
```

### 2.4 Agent 权限系统

```mermaid
flowchart TD
    subgraph "权限配置来源"
        DEFAULT[默认权限]
        USER[用户配置]
        AGENT_CFG[Agent 特定配置]
    end

    subgraph "权限合并"
        MERGE[PermissionNext.merge]
    end

    subgraph "最终权限"
        FINAL[合并后的权限集]
    end

    DEFAULT --> MERGE
    USER --> MERGE
    AGENT_CFG --> MERGE
    MERGE --> FINAL

    subgraph "权限类型"
        READ[read - 读取]
        EDIT[edit - 编辑]
        BASH[bash - 命令执行]
        EXT_DIR[external_directory - 外部目录]
        QUESTION[question - 提问]
        PLAN_ENTER[plan_enter - 进入规划]
        PLAN_EXIT[plan_exit - 退出规划]
    end

    FINAL --> READ
    FINAL --> EDIT
    FINAL --> BASH
    FINAL --> EXT_DIR
    FINAL --> QUESTION
    FINAL --> PLAN_ENTER
    FINAL --> PLAN_EXIT
```

## 3. Session 会话系统

### 3.1 Session 数据结构

```mermaid
classDiagram
    class SessionInfo {
        +string id
        +string slug
        +string projectID
        +string directory
        +string parentID
        +Summary summary
        +Share share
        +string title
        +string version
        +Time time
        +Ruleset permission
        +Revert revert
    }

    class Summary {
        +number additions
        +number deletions
        +number files
        +FileDiff[] diffs
    }

    class Share {
        +string url
    }

    class Time {
        +number created
        +number updated
        +number compacting
        +number archived
    }

    class Revert {
        +string messageID
        +string partID
        +string snapshot
        +string diff
    }

    SessionInfo --> Summary
    SessionInfo --> Share
    SessionInfo --> Time
    SessionInfo --> Revert
```

### 3.2 Session 生命周期

```mermaid
stateDiagram-v2
    [*] --> Created: create()
    Created --> Active: 用户交互
    Active --> Active: 消息交换
    Active --> Compacting: 上下文过长
    Compacting --> Active: 压缩完成
    Active --> Shared: share()
    Shared --> Active: 继续交互
    Active --> Archived: archive()
    Archived --> [*]

    Active --> Forked: fork()
    Forked --> Active: 新会话

    Active --> Reverted: revert()
    Reverted --> Active: 恢复点
```

### 3.3 Session 事件系统

```mermaid
graph LR
    subgraph "Session 事件"
        CREATED[session.created]
        UPDATED[session.updated]
        DELETED[session.deleted]
        DIFF[session.diff]
        ERROR[session.error]
    end

    subgraph "订阅者"
        SERVER[Server SSE]
        TUI[TUI 界面]
        STORAGE[存储同步]
    end

    CREATED --> SERVER
    CREATED --> TUI
    UPDATED --> SERVER
    UPDATED --> TUI
    UPDATED --> STORAGE
    DELETED --> SERVER
    DELETED --> TUI
    DIFF --> SERVER
    ERROR --> SERVER
    ERROR --> TUI
```

### 3.4 消息处理流程

```mermaid
sequenceDiagram
    participant U as 用户
    participant S as Session
    participant P as Processor
    participant A as Agent
    participant L as LLM
    participant T as Tool

    U->>S: 发送消息
    S->>S: 创建 UserMessage
    S->>P: 处理消息
    P->>A: 获取 Agent 配置
    P->>P: 构建 System Prompt
    P->>L: 调用 LLM

    loop 工具调用循环
        L-->>P: 返回响应/工具调用
        alt 包含工具调用
            P->>T: 执行工具
            T-->>P: 返回结果
            P->>P: 创建 ToolResult Part
            P->>L: 继续对话
        else 文本响应
            P->>S: 创建 AssistantMessage
        end
    end

    S-->>U: 流式响应
```

## 4. 消息系统

### 4.1 消息类型

```mermaid
classDiagram
    class Message {
        <<abstract>>
        +string id
        +string sessionID
        +enum role
        +Time time
    }

    class UserMessage {
        +role: "user"
        +string agent
        +Model model
    }

    class AssistantMessage {
        +role: "assistant"
        +string parentID
        +Error error
        +Metadata metadata
    }

    class Metadata {
        +ModelUsage usage
        +string agent
        +Model model
        +ProviderMetadata provider
    }

    Message <|-- UserMessage
    Message <|-- AssistantMessage
    AssistantMessage --> Metadata
```

### 4.2 消息 Part 类型

```mermaid
graph TB
    subgraph "Part 类型"
        TEXT[text<br/>文本内容]
        IMAGE[image<br/>图片]
        FILE[file<br/>文件]
        TOOL_CALL[tool-call<br/>工具调用]
        TOOL_RESULT[tool-result<br/>工具结果]
        STEP_START[step-start<br/>步骤开始]
    end

    subgraph "Part 结构"
        PART_BASE[BasePart<br/>id, messageID, sessionID, time]
    end

    PART_BASE --> TEXT
    PART_BASE --> IMAGE
    PART_BASE --> FILE
    PART_BASE --> TOOL_CALL
    PART_BASE --> TOOL_RESULT
    PART_BASE --> STEP_START
```

## 5. Session 存储结构

```mermaid
graph TB
    subgraph "存储路径"
        ROOT[.opencode/]
        SESSIONS[sessions/]
        SESSION_DIR[{sessionID}/]
        MESSAGES[messages/]
        PARTS[parts/]
        SNAPSHOTS[snapshots/]
    end

    ROOT --> SESSIONS
    SESSIONS --> SESSION_DIR
    SESSION_DIR --> MESSAGES
    SESSION_DIR --> PARTS
    SESSION_DIR --> SNAPSHOTS

    subgraph "文件内容"
        SESSION_JSON[session.json<br/>会话元数据]
        MSG_JSON[{messageID}.json<br/>消息数据]
        PART_JSON[{partID}.json<br/>Part 数据]
        SNAP_FILE[{snapshotID}<br/>快照文件]
    end

    SESSION_DIR --> SESSION_JSON
    MESSAGES --> MSG_JSON
    PARTS --> PART_JSON
    SNAPSHOTS --> SNAP_FILE
```

## 6. 上下文压缩 (Compaction)

```mermaid
flowchart TD
    START[开始检查]
    CHECK[检查上下文长度]
    THRESHOLD{超过阈值?}
    COMPACT[启动压缩]
    SELECT[选择压缩代理]
    SUMMARIZE[生成摘要]
    REPLACE[替换旧消息]
    END[完成]

    START --> CHECK
    CHECK --> THRESHOLD
    THRESHOLD -- 是 --> COMPACT
    THRESHOLD -- 否 --> END
    COMPACT --> SELECT
    SELECT --> SUMMARIZE
    SUMMARIZE --> REPLACE
    REPLACE --> END
```

## 7. 会话执行主循环

```mermaid
flowchart TD
    START[接收用户输入]
    PARSE[解析输入]
    AGENT[选择 Agent]
    PROMPT[构建 Prompt]
    LLM[调用 LLM]
    RESPONSE{响应类型}

    TEXT[处理文本]
    TOOL[处理工具调用]
    PERM{权限检查}
    EXEC[执行工具]
    RESULT[收集结果]

    STREAM[流式输出]
    SAVE[保存消息]
    CHECK[检查是否完成]
    DONE{完成?}
    END[结束]

    START --> PARSE
    PARSE --> AGENT
    AGENT --> PROMPT
    PROMPT --> LLM
    LLM --> RESPONSE

    RESPONSE -- 文本 --> TEXT
    RESPONSE -- 工具调用 --> TOOL

    TEXT --> STREAM
    TOOL --> PERM
    PERM -- 允许 --> EXEC
    PERM -- 询问 --> ASK[询问用户]
    ASK --> EXEC
    PERM -- 拒绝 --> DENY[拒绝执行]
    EXEC --> RESULT
    DENY --> RESULT
    RESULT --> LLM

    STREAM --> SAVE
    SAVE --> CHECK
    CHECK --> DONE
    DONE -- 是 --> END
    DONE -- 否 --> LLM
```

## 8. 子代理调用流程

```mermaid
sequenceDiagram
    participant P as Primary Agent
    participant T as Task Tool
    participant S as SubAgent Session
    participant SA as SubAgent
    participant L as LLM

    P->>T: 调用 Task 工具
    T->>S: 创建子会话
    S->>SA: 选择子代理 (general/explore)
    SA->>L: 执行任务
    L-->>SA: 返回结果

    loop 子代理执行
        SA->>L: 继续执行
        L-->>SA: 返回响应
    end

    SA-->>S: 完成任务
    S-->>T: 返回结果
    T-->>P: 返回给主代理
```

## 9. 会话恢复与回滚

```mermaid
flowchart TD
    subgraph "会话恢复"
        LOAD[加载会话]
        MESSAGES[读取消息历史]
        REBUILD[重建上下文]
        CONTINUE[继续对话]
    end

    subgraph "会话回滚"
        SELECT[选择回滚点]
        SNAPSHOT[获取快照]
        RESTORE[恢复文件状态]
        TRUNCATE[截断消息]
        READY[准备继续]
    end

    LOAD --> MESSAGES
    MESSAGES --> REBUILD
    REBUILD --> CONTINUE

    SELECT --> SNAPSHOT
    SNAPSHOT --> RESTORE
    RESTORE --> TRUNCATE
    TRUNCATE --> READY
```

## 10. 关键文件

| 文件 | 功能 |
|------|------|
| `session/index.ts` | 会话管理核心 |
| `session/message-v2.ts` | 消息数据结构 |
| `session/prompt.ts` | Prompt 构建 |
| `session/processor.ts` | 消息处理器 |
| `session/compaction.ts` | 上下文压缩 |
| `session/llm.ts` | LLM 调用封装 |
| `session/system.ts` | System Prompt |
| `session/todo.ts` | Todo 管理 |
| `agent/agent.ts` | Agent 定义与管理 |

## 11. 相关文档

- [工具系统架构](./04-tool-system.md)
- [Provider 系统](./05-provider-system.md)
- [权限系统详解](./08-permission-system.md)
