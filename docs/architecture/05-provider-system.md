# Provider 系统架构

## 1. 概述

Provider 系统是 OpenCode 与各种 AI 模型提供商之间的抽象层，支持 20+ 个 AI 提供商，提供统一的接口进行模型调用。

## 2. 支持的 Provider

### 2.1 主要 Provider

| Provider ID | SDK 包 | 描述 |
|-------------|--------|------|
| `anthropic` | `@ai-sdk/anthropic` | Claude 模型 |
| `openai` | `@ai-sdk/openai` | GPT 模型 |
| `google` | `@ai-sdk/google` | Gemini 模型 |
| `opencode` | 内置 | OpenCode 托管服务 |

### 2.2 云服务 Provider

| Provider ID | SDK 包 | 描述 |
|-------------|--------|------|
| `azure` | `@ai-sdk/azure` | Azure OpenAI |
| `amazon-bedrock` | `@ai-sdk/amazon-bedrock` | AWS Bedrock |
| `google-vertex` | `@ai-sdk/google-vertex` | Google Vertex AI |
| `vercel` | `@ai-sdk/vercel` | Vercel AI |

### 2.3 第三方 Provider

| Provider ID | SDK 包 | 描述 |
|-------------|--------|------|
| `openrouter` | `@openrouter/ai-sdk-provider` | OpenRouter |
| `groq` | `@ai-sdk/groq` | Groq |
| `mistral` | `@ai-sdk/mistral` | Mistral AI |
| `cohere` | `@ai-sdk/cohere` | Cohere |
| `deepinfra` | `@ai-sdk/deepinfra` | DeepInfra |
| `togetherai` | `@ai-sdk/togetherai` | Together AI |
| `perplexity` | `@ai-sdk/perplexity` | Perplexity |
| `cerebras` | `@ai-sdk/cerebras` | Cerebras |
| `xai` | `@ai-sdk/xai` | xAI (Grok) |

### 2.4 代码助手 Provider

| Provider ID | SDK 包 | 描述 |
|-------------|--------|------|
| `github-copilot` | 自定义 | GitHub Copilot |
| `gitlab` | `@gitlab/gitlab-ai-provider` | GitLab AI |

## 3. Provider 架构图

```mermaid
graph TB
    subgraph "Provider 系统"
        PROVIDER_NS[Provider Namespace]
        CONFIG[Config 配置]
        AUTH[Auth 认证]
        MODELS[Models 模型定义]
    end

    subgraph "SDK 层"
        AI_SDK[Vercel AI SDK]

        subgraph "Provider SDK"
            ANTHROPIC[anthropic]
            OPENAI[openai]
            GOOGLE[google]
            AZURE[azure]
            BEDROCK[bedrock]
            OTHER[其他 Provider...]
        end
    end

    subgraph "应用层"
        SESSION[Session]
        AGENT[Agent]
    end

    SESSION --> PROVIDER_NS
    AGENT --> PROVIDER_NS

    PROVIDER_NS --> CONFIG
    PROVIDER_NS --> AUTH
    PROVIDER_NS --> MODELS

    PROVIDER_NS --> AI_SDK
    AI_SDK --> ANTHROPIC
    AI_SDK --> OPENAI
    AI_SDK --> GOOGLE
    AI_SDK --> AZURE
    AI_SDK --> BEDROCK
    AI_SDK --> OTHER
```

## 4. Provider 数据结构

```mermaid
classDiagram
    class ProviderInfo {
        +string id
        +string name
        +string sdk
        +string[] env
        +string[] api
        +ModelInfo[] models
        +object options
    }

    class ModelInfo {
        +string id
        +string name
        +string providerID
        +ModelCost cost
        +ModelLimit limit
        +string[] capabilities
    }

    class ModelCost {
        +number input
        +number output
        +number cache_read
        +number cache_write
    }

    class ModelLimit {
        +number context
        +number output
    }

    ProviderInfo --> ModelInfo
    ModelInfo --> ModelCost
    ModelInfo --> ModelLimit
```

## 5. Provider 加载流程

```mermaid
flowchart TD
    START[开始加载]

    subgraph "配置阶段"
        LOAD_CONFIG[加载配置]
        LOAD_MODELS[加载模型定义]
        LOAD_PLUGINS[加载插件 Provider]
    end

    subgraph "认证阶段"
        CHECK_ENV[检查环境变量]
        CHECK_AUTH[检查 Auth 存储]
        CHECK_CONFIG[检查配置文件]
    end

    subgraph "初始化阶段"
        CREATE_SDK[创建 SDK 实例]
        APPLY_OPTIONS[应用选项]
        REGISTER[注册 Provider]
    end

    START --> LOAD_CONFIG
    LOAD_CONFIG --> LOAD_MODELS
    LOAD_MODELS --> LOAD_PLUGINS

    LOAD_PLUGINS --> CHECK_ENV
    CHECK_ENV --> CHECK_AUTH
    CHECK_AUTH --> CHECK_CONFIG

    CHECK_CONFIG --> CREATE_SDK
    CREATE_SDK --> APPLY_OPTIONS
    APPLY_OPTIONS --> REGISTER
```

## 6. 模型获取流程

```mermaid
sequenceDiagram
    participant A as Agent
    participant P as Provider
    participant C as Config
    participant AU as Auth
    participant S as SDK

    A->>P: getModel(providerID, modelID)
    P->>C: 获取 Provider 配置
    P->>AU: 获取认证信息
    P->>P: 合并选项

    alt 内置 Provider
        P->>S: 创建 SDK 实例
        S-->>P: 返回 SDK
    else 自定义 Provider
        P->>P: 动态加载 SDK
        P-->>P: 返回 SDK
    end

    P->>S: sdk.languageModel(modelID)
    S-->>P: 返回 Language Model
    P-->>A: 返回模型实例
```

## 7. Provider 认证系统

```mermaid
graph TB
    subgraph "认证来源优先级"
        ENV[1. 环境变量<br/>ANTHROPIC_API_KEY 等]
        AUTH_STORE[2. Auth 存储<br/>opencode auth]
        CONFIG_FILE[3. 配置文件<br/>.opencode/config.json]
        PLUGIN[4. 插件提供]
    end

    subgraph "认证类型"
        API_KEY[API Key]
        OAUTH[OAuth Token]
        AWS_CREDS[AWS Credentials]
        CUSTOM[自定义认证]
    end

    ENV --> API_KEY
    AUTH_STORE --> API_KEY
    AUTH_STORE --> OAUTH
    CONFIG_FILE --> API_KEY
    CONFIG_FILE --> CUSTOM
    PLUGIN --> CUSTOM
    ENV --> AWS_CREDS
```

## 8. Provider 配置结构

```yaml
# .opencode/config.json
{
  "provider": {
    "anthropic": {
      "disabled": false,
      "options": {
        "headers": {
          "anthropic-beta": "..."
        }
      }
    },
    "openai": {
      "options": {
        "baseURL": "https://custom-endpoint.com"
      }
    },
    "custom-provider": {
      "name": "Custom Provider",
      "sdk": "@ai-sdk/openai-compatible",
      "env": ["CUSTOM_API_KEY"],
      "models": {
        "custom-model": {
          "name": "Custom Model",
          "cost": { "input": 0.001, "output": 0.002 },
          "limit": { "context": 128000 }
        }
      }
    }
  }
}
```

## 9. 模型选择流程

```mermaid
flowchart TD
    START[选择模型]

    subgraph "用户指定"
        CLI_ARG[CLI 参数<br/>--model]
        CONFIG_DEFAULT[配置默认值<br/>config.model]
        AGENT_MODEL[Agent 指定<br/>agent.model]
    end

    subgraph "自动选择"
        LIST_AVAILABLE[列出可用模型]
        FILTER_CAPS[过滤能力需求]
        SORT_PRIORITY[按优先级排序]
        SELECT_FIRST[选择首个]
    end

    FINAL[返回模型]

    START --> CLI_ARG
    CLI_ARG -- 未指定 --> CONFIG_DEFAULT
    CONFIG_DEFAULT -- 未指定 --> AGENT_MODEL
    AGENT_MODEL -- 未指定 --> LIST_AVAILABLE

    CLI_ARG -- 已指定 --> FINAL
    CONFIG_DEFAULT -- 已指定 --> FINAL
    AGENT_MODEL -- 已指定 --> FINAL

    LIST_AVAILABLE --> FILTER_CAPS
    FILTER_CAPS --> SORT_PRIORITY
    SORT_PRIORITY --> SELECT_FIRST
    SELECT_FIRST --> FINAL
```

## 10. Provider Transform 系统

```mermaid
flowchart TD
    subgraph "Transform 管道"
        INPUT[LLM 请求]
        T1[Provider 特定转换]
        T2[模型特定转换]
        T3[选项合并]
        OUTPUT[最终请求]
    end

    subgraph "转换类型"
        HEADERS[Header 注入]
        PARAMS[参数映射]
        TOOLS[工具格式转换]
        MESSAGES[消息格式转换]
    end

    INPUT --> T1
    T1 --> T2
    T2 --> T3
    T3 --> OUTPUT

    T1 --> HEADERS
    T1 --> PARAMS
    T2 --> TOOLS
    T2 --> MESSAGES
```

## 11. 自定义 Provider 开发

### 11.1 通过配置添加

```json
{
  "provider": {
    "my-provider": {
      "name": "My Provider",
      "sdk": "@ai-sdk/openai-compatible",
      "env": ["MY_PROVIDER_API_KEY"],
      "options": {
        "baseURL": "https://api.my-provider.com/v1"
      },
      "models": {
        "my-model": {
          "name": "My Model",
          "cost": { "input": 0.001, "output": 0.002 },
          "limit": { "context": 32000, "output": 4096 }
        }
      }
    }
  }
}
```

### 11.2 通过插件添加

```typescript
// plugin.ts
export default async (input: PluginInput): Promise<Hooks> => {
  return {
    provider: {
      "my-provider": {
        id: "my-provider",
        name: "My Provider",
        sdk: "@ai-sdk/openai-compatible",
        env: ["MY_API_KEY"],
        models: {
          "model-1": {
            id: "model-1",
            name: "Model 1",
            cost: { input: 0.001, output: 0.002 },
            limit: { context: 32000 },
          },
        },
      },
    },
  }
}
```

## 12. 模型能力检测

```mermaid
graph TB
    subgraph "能力类型"
        VISION[vision<br/>图像理解]
        TOOL_USE[tool_use<br/>工具调用]
        STREAMING[streaming<br/>流式响应]
        THINKING[thinking<br/>思考模式]
        CACHE[cache<br/>上下文缓存]
    end

    subgraph "检测方式"
        MODEL_DEF[模型定义]
        RUNTIME[运行时检测]
        CONFIG[配置覆盖]
    end

    MODEL_DEF --> VISION
    MODEL_DEF --> TOOL_USE
    MODEL_DEF --> STREAMING
    RUNTIME --> THINKING
    CONFIG --> CACHE
```

## 13. 关键文件

| 文件 | 功能 |
|------|------|
| `provider/provider.ts` | Provider 核心逻辑 |
| `provider/models.ts` | 模型定义与发现 |
| `provider/auth.ts` | Provider 认证 |
| `provider/transform.ts` | 请求转换 |
| `auth/index.ts` | 认证存储管理 |

## 14. 相关文档

- [会话与代理系统](./03-session-agent.md)
- [配置系统详解](./09-config-system.md)
