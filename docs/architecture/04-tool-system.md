# 工具系统架构

## 1. 概述

工具系统是 OpenCode 的核心能力层，允许 AI 代理执行各种操作，包括文件读写、命令执行、代码搜索、网络请求等。

## 2. 工具列表

### 2.1 核心工具

| 工具 ID | 文件 | 功能描述 |
|---------|------|----------|
| `bash` | `bash.ts` | 执行 Shell 命令 |
| `read` | `read.ts` | 读取文件内容 |
| `edit` | `edit.ts` | 编辑文件 |
| `write` | `write.ts` | 写入文件 |
| `glob` | `glob.ts` | 文件模式匹配 |
| `grep` | `grep.ts` | 文本搜索 |
| `ls` | `ls.ts` | 目录列表 |

### 2.2 高级工具

| 工具 ID | 文件 | 功能描述 |
|---------|------|----------|
| `task` | `task.ts` | 子代理任务 |
| `webfetch` | `webfetch.ts` | 网页内容获取 |
| `websearch` | `websearch.ts` | 网络搜索 |
| `codesearch` | `codesearch.ts` | 代码搜索 |
| `skill` | `skill.ts` | 技能调用 |
| `question` | `question.ts` | 向用户提问 |

### 2.3 状态管理工具

| 工具 ID | 文件 | 功能描述 |
|---------|------|----------|
| `todowrite` | `todo.ts` | 写入 Todo |
| `todoread` | `todo.ts` | 读取 Todo |
| `plan_enter` | `plan.ts` | 进入规划模式 |
| `plan_exit` | `plan.ts` | 退出规划模式 |

### 2.4 实验性工具

| 工具 ID | 文件 | 功能描述 |
|---------|------|----------|
| `batch` | `batch.ts` | 批量操作 |
| `lsp` | `lsp.ts` | LSP 集成 |
| `multiedit` | `multiedit.ts` | 多文件编辑 |
| `patch` | `patch.ts` | 补丁应用 |

## 3. 工具系统架构图

```mermaid
graph TB
    subgraph "Tool Registry"
        REGISTRY[ToolRegistry]
        STATE[Instance State<br/>工具状态]
        CUSTOM[Custom Tools<br/>自定义工具]
    end

    subgraph "核心工具"
        BASH[BashTool]
        READ[ReadTool]
        EDIT[EditTool]
        WRITE[WriteTool]
        GLOB[GlobTool]
        GREP[GrepTool]
    end

    subgraph "高级工具"
        TASK[TaskTool]
        WEBFETCH[WebFetchTool]
        WEBSEARCH[WebSearchTool]
        QUESTION[QuestionTool]
        SKILL[SkillTool]
    end

    subgraph "状态工具"
        TODO_W[TodoWriteTool]
        TODO_R[TodoReadTool]
        PLAN_E[PlanEnterTool]
        PLAN_X[PlanExitTool]
    end

    subgraph "插件工具"
        PLUGIN[Plugin Tools]
        CONFIG_TOOLS[Config Tools<br/>.opencode/tools/]
    end

    REGISTRY --> STATE
    STATE --> CUSTOM

    REGISTRY --> BASH
    REGISTRY --> READ
    REGISTRY --> EDIT
    REGISTRY --> WRITE
    REGISTRY --> GLOB
    REGISTRY --> GREP

    REGISTRY --> TASK
    REGISTRY --> WEBFETCH
    REGISTRY --> WEBSEARCH
    REGISTRY --> QUESTION
    REGISTRY --> SKILL

    REGISTRY --> TODO_W
    REGISTRY --> TODO_R
    REGISTRY --> PLAN_E
    REGISTRY --> PLAN_X

    CUSTOM --> PLUGIN
    CUSTOM --> CONFIG_TOOLS
```

## 4. 工具定义结构

```mermaid
classDiagram
    class ToolInfo {
        +string id
        +init(ctx: InitContext) Promise~ToolImpl~
    }

    class InitContext {
        +Agent.Info agent
    }

    class ToolImpl {
        +string description
        +ZodType parameters
        +execute(args, ctx) Promise~Result~
        +formatValidationError(error) string
    }

    class ToolContext {
        +string sessionID
        +string messageID
        +string agent
        +AbortSignal abort
        +string callID
        +object extra
        +metadata(input) void
        +ask(input) Promise~void~
    }

    class ToolResult {
        +string title
        +object metadata
        +string output
        +FilePart[] attachments
    }

    ToolInfo --> InitContext
    ToolInfo --> ToolImpl
    ToolImpl --> ToolContext
    ToolImpl --> ToolResult
```

## 5. 工具执行流程

```mermaid
sequenceDiagram
    participant L as LLM
    participant P as Processor
    participant R as Registry
    participant T as Tool
    participant C as Context

    L->>P: 工具调用请求
    P->>R: 获取工具定义
    R-->>P: 返回 ToolInfo
    P->>T: init(agent)
    T-->>P: 返回 ToolImpl
    P->>P: 验证参数 (Zod)

    alt 参数有效
        P->>C: 创建执行上下文
        P->>T: execute(args, ctx)
        T->>T: 执行操作
        T-->>P: 返回结果
        P->>P: 截断输出 (Truncate)
        P-->>L: 返回工具结果
    else 参数无效
        P-->>L: 返回验证错误
    end
```

## 6. 工具注册流程

```mermaid
flowchart TD
    START[启动]

    subgraph "内置工具"
        BUILTIN[加载内置工具]
    end

    subgraph "配置工具"
        SCAN[扫描 tools/*.ts]
        IMPORT[动态导入]
        WRAP[包装为 Tool.Info]
    end

    subgraph "插件工具"
        PLUGINS[加载插件]
        PLUGIN_TOOLS[提取工具定义]
    end

    MERGE[合并工具列表]
    READY[工具就绪]

    START --> BUILTIN
    START --> SCAN
    START --> PLUGINS

    SCAN --> IMPORT
    IMPORT --> WRAP

    PLUGINS --> PLUGIN_TOOLS

    BUILTIN --> MERGE
    WRAP --> MERGE
    PLUGIN_TOOLS --> MERGE
    MERGE --> READY
```

## 7. 权限检查流程

```mermaid
flowchart TD
    CALL[工具调用]
    GET_PERM[获取权限规则]
    CHECK{检查权限}

    ALLOW[允许执行]
    ASK[询问用户]
    DENY[拒绝执行]

    EXEC[执行工具]
    RESULT[返回结果]
    ERROR[返回错误]

    CALL --> GET_PERM
    GET_PERM --> CHECK

    CHECK -- allow --> ALLOW
    CHECK -- ask --> ASK
    CHECK -- deny --> DENY

    ALLOW --> EXEC
    ASK --> USER_RESP{用户响应}
    USER_RESP -- 允许 --> EXEC
    USER_RESP -- 拒绝 --> DENY

    EXEC --> RESULT
    DENY --> ERROR
```

## 8. 输出截断系统

```mermaid
flowchart TD
    OUTPUT[工具输出]
    CHECK{检查大小}
    THRESHOLD[阈值判断]

    PASS[直接返回]
    TRUNCATE[截断处理]
    WRITE_FILE[写入临时文件]
    ADD_META[添加元数据]

    OUTPUT --> CHECK
    CHECK --> THRESHOLD

    THRESHOLD -- 小于阈值 --> PASS
    THRESHOLD -- 大于阈值 --> TRUNCATE

    TRUNCATE --> WRITE_FILE
    WRITE_FILE --> ADD_META
    ADD_META --> RETURN[返回截断内容]
```

## 9. 核心工具详解

### 9.1 Bash 工具

```mermaid
graph TB
    subgraph "Bash Tool"
        INPUT[命令输入]
        VALIDATE[验证命令]
        PTY[PTY 执行]
        STREAM[流式输出]
        TIMEOUT[超时控制]
        RESULT[结果收集]
    end

    INPUT --> VALIDATE
    VALIDATE --> PTY
    PTY --> STREAM
    STREAM --> TIMEOUT
    TIMEOUT --> RESULT
```

### 9.2 Edit 工具

```mermaid
flowchart TD
    INPUT[编辑请求]
    READ[读取原文件]
    FIND[查找目标字符串]

    FOUND{找到?}
    UNIQUE{唯一?}

    REPLACE[执行替换]
    WRITE[写入文件]
    DIFF[生成 Diff]

    ERROR_NOT_FOUND[错误: 未找到]
    ERROR_MULTIPLE[错误: 多个匹配]

    INPUT --> READ
    READ --> FIND
    FIND --> FOUND

    FOUND -- 否 --> ERROR_NOT_FOUND
    FOUND -- 是 --> UNIQUE

    UNIQUE -- 否 --> ERROR_MULTIPLE
    UNIQUE -- 是 --> REPLACE

    REPLACE --> WRITE
    WRITE --> DIFF
```

### 9.3 Task 工具 (子代理)

```mermaid
sequenceDiagram
    participant P as Primary Agent
    participant T as TaskTool
    participant S as SubSession
    participant A as SubAgent
    participant L as LLM

    P->>T: 调用 Task
    T->>T: 解析参数
    T->>S: 创建子会话
    S->>A: 选择子代理类型
    A->>L: 执行任务

    loop 执行循环
        L-->>A: 返回响应
        alt 需要更多步骤
            A->>L: 继续执行
        else 完成
            A-->>S: 返回结果
        end
    end

    S-->>T: 会话结果
    T-->>P: 返回给主代理
```

## 10. 自定义工具开发

### 10.1 工具定义示例

```typescript
// .opencode/tools/my-tool.ts
import { z } from "zod"
import type { ToolDefinition } from "@opencode-ai/plugin"

export default {
  description: "My custom tool description",
  args: {
    input: z.string().describe("Input parameter"),
  },
  execute: async (args, ctx) => {
    // 工具实现逻辑
    return `Result: ${args.input}`
  },
} satisfies ToolDefinition
```

### 10.2 工具加载路径

```mermaid
graph TB
    subgraph "加载顺序"
        A[1. 内置工具]
        B[2. 项目 tools/*.ts]
        C[3. 全局配置 tools/*.ts]
        D[4. 插件工具]
    end

    A --> B --> C --> D
```

## 11. 工具上下文

```mermaid
classDiagram
    class ToolContext {
        +string sessionID
        +string messageID
        +string agent
        +AbortSignal abort
        +string callID
        +object extra
    }

    class ContextMethods {
        +metadata(input) void
        +ask(input) Promise~void~
    }

    class MetadataInput {
        +string title
        +object metadata
    }

    class AskInput {
        +string permission
        +string pattern
        +string action
        +string reason
    }

    ToolContext --> ContextMethods
    ContextMethods --> MetadataInput
    ContextMethods --> AskInput
```

## 12. 关键文件

| 文件 | 功能 |
|------|------|
| `tool/registry.ts` | 工具注册中心 |
| `tool/tool.ts` | 工具基础定义 |
| `tool/truncation.ts` | 输出截断 |
| `tool/bash.ts` | Bash 执行 |
| `tool/edit.ts` | 文件编辑 |
| `tool/read.ts` | 文件读取 |
| `tool/task.ts` | 子代理任务 |
| `tool/question.ts` | 用户提问 |

## 13. 相关文档

- [会话与代理系统](./03-session-agent.md)
- [Provider 系统](./05-provider-system.md)
- [权限系统详解](./08-permission-system.md)
