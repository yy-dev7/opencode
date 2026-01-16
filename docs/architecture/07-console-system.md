# Console 后台管理系统架构

## 1. 概述

Console 是 OpenCode 的云端管理控制台，提供用户认证、API 密钥管理、计费、模型配置等功能。它是一个完整的 SaaS 后台系统。

## 2. 目录结构

```
packages/console/
├── app/                 # SolidStart Web 应用
│   ├── src/
│   │   ├── app.tsx      # 主应用
│   │   ├── routes/      # 路由页面
│   │   ├── components/  # UI 组件
│   │   └── lib/         # 工具函数
├── core/                # 后端业务逻辑
│   ├── src/
│   │   ├── account.ts   # 账户管理
│   │   ├── billing.ts   # 计费系统
│   │   ├── key.ts       # API 密钥
│   │   ├── model.ts     # 模型管理
│   │   ├── provider.ts  # Provider 管理
│   │   ├── user.ts      # 用户管理
│   │   ├── workspace.ts # 工作空间
│   │   ├── schema/      # 数据库 Schema
│   │   └── drizzle/     # Drizzle ORM
├── function/            # Cloudflare Workers
│   └── src/
│       └── api/         # API 端点
├── mail/                # 邮件服务
│   └── src/
│       └── templates/   # 邮件模板
└── resource/            # CDK 资源定义
```

## 3. 系统架构图

```mermaid
graph TB
    subgraph "客户端"
        WEB_APP[Console Web App<br/>SolidStart]
        CLI_CLIENT[OpenCode CLI]
    end

    subgraph "边缘层"
        CF_PAGES[Cloudflare Pages]
        CF_WORKERS[Cloudflare Workers]
    end

    subgraph "业务层"
        CORE[Console Core<br/>业务逻辑]

        subgraph "核心模块"
            ACCOUNT[Account<br/>账户]
            BILLING[Billing<br/>计费]
            KEY[Key<br/>API 密钥]
            MODEL[Model<br/>模型]
            PROVIDER[Provider<br/>提供商]
            USER[User<br/>用户]
            WORKSPACE[Workspace<br/>工作空间]
        end
    end

    subgraph "数据层"
        DB[(PlanetScale<br/>MySQL)]
        DRIZZLE[Drizzle ORM]
    end

    subgraph "外部服务"
        STRIPE[Stripe<br/>支付]
        AUTH[OpenAuth<br/>认证]
        EMAIL[Email Service<br/>邮件]
    end

    WEB_APP --> CF_PAGES
    CLI_CLIENT --> CF_WORKERS

    CF_PAGES --> CORE
    CF_WORKERS --> CORE

    CORE --> ACCOUNT
    CORE --> BILLING
    CORE --> KEY
    CORE --> MODEL
    CORE --> PROVIDER
    CORE --> USER
    CORE --> WORKSPACE

    CORE --> DRIZZLE
    DRIZZLE --> DB

    BILLING --> STRIPE
    USER --> AUTH
    CORE --> EMAIL
```

## 4. 数据库 Schema

```mermaid
erDiagram
    USER {
        string id PK
        string email
        string name
        timestamp created
        timestamp updated
    }

    ACCOUNT {
        string id PK
        string userID FK
        string type
        string providerAccountID
        string accessToken
        string refreshToken
    }

    WORKSPACE {
        string id PK
        string name
        string slug
        timestamp created
    }

    KEY {
        string id PK
        string workspaceID FK
        string name
        string keyHash
        boolean active
        timestamp created
        timestamp lastUsed
    }

    BILLING {
        string id PK
        string workspaceID FK
        string stripeCustomerID
        string stripePriceID
        string status
        decimal balance
    }

    MODEL {
        string id PK
        string providerID
        string name
        json pricing
        json limits
        boolean enabled
    }

    PROVIDER {
        string id PK
        string name
        string type
        boolean enabled
    }

    USER ||--o{ ACCOUNT : has
    WORKSPACE ||--o{ KEY : has
    WORKSPACE ||--o| BILLING : has
    PROVIDER ||--o{ MODEL : has
```

## 5. API 架构

```mermaid
flowchart TD
    subgraph "API 端点"
        AUTH_API[/auth/*<br/>认证]
        USER_API[/user/*<br/>用户]
        WORKSPACE_API[/workspace/*<br/>工作空间]
        KEY_API[/key/*<br/>API 密钥]
        BILLING_API[/billing/*<br/>计费]
        MODEL_API[/model/*<br/>模型]
        PROVIDER_API[/provider/*<br/>提供商]
    end

    subgraph "中间件"
        AUTH_MW[认证中间件]
        RATE_LIMIT[速率限制]
        CORS_MW[CORS]
        LOG_MW[日志]
    end

    REQUEST[请求] --> CORS_MW
    CORS_MW --> LOG_MW
    LOG_MW --> RATE_LIMIT
    RATE_LIMIT --> AUTH_MW

    AUTH_MW --> AUTH_API
    AUTH_MW --> USER_API
    AUTH_MW --> WORKSPACE_API
    AUTH_MW --> KEY_API
    AUTH_MW --> BILLING_API
    AUTH_MW --> MODEL_API
    AUTH_MW --> PROVIDER_API
```

## 6. 认证流程

```mermaid
sequenceDiagram
    participant U as 用户
    participant C as Console App
    participant A as OpenAuth
    participant API as API Server
    participant DB as Database

    U->>C: 访问 Console
    C->>A: 重定向到认证
    U->>A: 输入凭据
    A->>A: 验证凭据
    A-->>C: 返回 Token
    C->>API: 携带 Token 请求
    API->>API: 验证 Token
    API->>DB: 查询/创建用户
    DB-->>API: 用户数据
    API-->>C: 返回数据
    C-->>U: 显示 Dashboard
```

## 7. 计费系统

```mermaid
flowchart TD
    subgraph "计费流程"
        USAGE[使用量记录]
        AGGREGATE[聚合统计]
        CALCULATE[计算费用]
        CHARGE[扣费/充值]
    end

    subgraph "Stripe 集成"
        CUSTOMER[创建 Customer]
        PAYMENT[支付方法]
        INVOICE[发票]
        WEBHOOK[Webhook 处理]
    end

    subgraph "余额管理"
        BALANCE[余额查询]
        TOPUP[充值]
        DEDUCT[扣款]
        ALERT[余额警告]
    end

    USAGE --> AGGREGATE
    AGGREGATE --> CALCULATE
    CALCULATE --> CHARGE

    TOPUP --> CUSTOMER
    CUSTOMER --> PAYMENT
    PAYMENT --> INVOICE
    INVOICE --> WEBHOOK
    WEBHOOK --> BALANCE

    CHARGE --> DEDUCT
    DEDUCT --> ALERT
```

## 8. API 密钥管理

```mermaid
stateDiagram-v2
    [*] --> Created: 创建密钥
    Created --> Active: 激活
    Active --> Active: 使用
    Active --> Revoked: 撤销
    Active --> Expired: 过期
    Revoked --> [*]
    Expired --> [*]

    note right of Active
        - 记录使用次数
        - 记录最后使用时间
        - 速率限制检查
    end note
```

## 9. 工作空间管理

```mermaid
classDiagram
    class Workspace {
        +string id
        +string name
        +string slug
        +Member[] members
        +Key[] keys
        +Billing billing
        +create() Workspace
        +update(data) void
        +delete() void
        +addMember(user) void
        +removeMember(user) void
    }

    class Member {
        +string userID
        +string role
        +timestamp joined
    }

    class Key {
        +string id
        +string name
        +string prefix
        +boolean active
        +create() Key
        +revoke() void
    }

    class Billing {
        +decimal balance
        +string status
        +topup(amount) void
        +getUsage() Usage
    }

    Workspace --> Member
    Workspace --> Key
    Workspace --> Billing
```

## 10. Console Web App 路由

```mermaid
graph TB
    subgraph "公开路由"
        LOGIN[/login]
        SIGNUP[/signup]
        CALLBACK[/callback]
    end

    subgraph "受保护路由"
        DASHBOARD[/dashboard]
        WORKSPACE_PAGE[/workspace/:id]
        KEYS[/workspace/:id/keys]
        BILLING_PAGE[/workspace/:id/billing]
        SETTINGS[/settings]
        MODELS_PAGE[/models]
    end

    subgraph "管理员路由"
        ADMIN[/admin]
        ADMIN_USERS[/admin/users]
        ADMIN_PROVIDERS[/admin/providers]
        ADMIN_MODELS[/admin/models]
    end

    LOGIN --> DASHBOARD
    SIGNUP --> DASHBOARD
    CALLBACK --> DASHBOARD

    DASHBOARD --> WORKSPACE_PAGE
    WORKSPACE_PAGE --> KEYS
    WORKSPACE_PAGE --> BILLING_PAGE
    DASHBOARD --> SETTINGS
    DASHBOARD --> MODELS_PAGE
```

## 11. Cloudflare Workers 函数

```mermaid
graph TB
    subgraph "Worker Functions"
        API_WORKER[API Worker<br/>主 API 服务]
        WEBHOOK_WORKER[Webhook Worker<br/>处理回调]
        CRON_WORKER[Cron Worker<br/>定时任务]
    end

    subgraph "功能"
        AUTH_FUNC[认证处理]
        BILLING_FUNC[计费处理]
        USAGE_FUNC[使用量统计]
        NOTIFY_FUNC[通知服务]
    end

    API_WORKER --> AUTH_FUNC
    API_WORKER --> BILLING_FUNC
    API_WORKER --> USAGE_FUNC

    WEBHOOK_WORKER --> BILLING_FUNC
    WEBHOOK_WORKER --> NOTIFY_FUNC

    CRON_WORKER --> USAGE_FUNC
    CRON_WORKER --> NOTIFY_FUNC
```

## 12. 邮件系统

```mermaid
flowchart TD
    subgraph "触发事件"
        SIGNUP_EVENT[注册成功]
        BILLING_EVENT[计费事件]
        KEY_EVENT[密钥事件]
        ALERT_EVENT[警告事件]
    end

    subgraph "邮件模板"
        WELCOME[欢迎邮件]
        INVOICE_MAIL[发票邮件]
        KEY_CREATED[密钥创建]
        LOW_BALANCE[余额不足]
    end

    subgraph "发送"
        RENDER[渲染模板]
        SEND[发送邮件]
    end

    SIGNUP_EVENT --> WELCOME
    BILLING_EVENT --> INVOICE_MAIL
    KEY_EVENT --> KEY_CREATED
    ALERT_EVENT --> LOW_BALANCE

    WELCOME --> RENDER
    INVOICE_MAIL --> RENDER
    KEY_CREATED --> RENDER
    LOW_BALANCE --> RENDER

    RENDER --> SEND
```

## 13. 基础设施配置

```mermaid
graph TB
    subgraph "SST 部署"
        SST[SST Framework]

        subgraph "资源"
            PAGES[Cloudflare Pages]
            WORKERS[Cloudflare Workers]
            KV[Cloudflare KV]
            D1[Cloudflare D1]
        end

        subgraph "外部资源"
            PLANETSCALE[PlanetScale DB]
            STRIPE_RES[Stripe]
        end
    end

    SST --> PAGES
    SST --> WORKERS
    SST --> KV
    SST --> D1
    SST --> PLANETSCALE
    SST --> STRIPE_RES
```

## 14. 关键文件

### core 模块

| 文件 | 功能 |
|------|------|
| `account.ts` | 账户管理 |
| `billing.ts` | 计费系统 |
| `key.ts` | API 密钥管理 |
| `model.ts` | 模型配置 |
| `provider.ts` | Provider 管理 |
| `user.ts` | 用户管理 |
| `workspace.ts` | 工作空间管理 |
| `schema/*.sql.ts` | 数据库 Schema |

### app 模块

| 文件 | 功能 |
|------|------|
| `app.tsx` | 主应用 |
| `routes/` | 页面路由 |
| `components/` | UI 组件 |
| `lib/` | 工具函数 |

## 15. 相关文档

- [UI 与客户端架构](./06-ui-client.md)
- [Provider 系统](./05-provider-system.md)
