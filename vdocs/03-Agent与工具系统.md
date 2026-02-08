# Agent 与工具系统

## 1. Agent 运行时概述

OpenClaw 的 Agent 系统基于 **Pi Agent Core** 运行时，支持多种 LLM 提供商，提供会话管理、工具调用、Skills 注入等能力。Agent 运行时负责管理从用户请求到 LLM 交互的完整生命周期。

| **属性** | **详情** |
|:---:|:---:|
| **运行时** | Pi Agent Core（@mariozechner/pi-agent-core） |
| **嵌入模式** | Pi Embedded Runner |
| **会话存储** | JSONL Transcript 文件 |
| **模型支持** | 29+ 提供商（详见 [大模型支持](./10-大模型支持.md)） |
| **工具系统** | 声明式工具注册 + 动态执行 |

## 2. Agent 架构图

```mermaid
graph TB
    subgraph "**Agent 入口**"
        CMD[**agentCommand<br/>src/commands/agent.ts**]
        GWMETHOD[**Gateway agent 方法<br/>server-methods/agent.ts**]
    end

    subgraph "**Pi Embedded Runner**"
        RUNNER[**runEmbeddedPiAgent<br/>pi-embedded-runner/**]
        ATTEMPT[**attempt.ts<br/>单次运行尝试**]
        COMPACT[**compact.ts<br/>Transcript 压缩**]
    end

    subgraph "**Agent 上下文构建**"
        SYSPROMPT[**System Prompt<br/>system-prompt.ts**]
        SKILLLOAD[**Skills 加载<br/>skills/workspace.ts**]
        TOOLREG[**工具注册<br/>openclaw-tools.ts**]
        MODELCFG[**模型配置<br/>model resolution**]
    end

    subgraph "**工具系统 Tools**"
        T_SEARCH[**web_search<br/>网页搜索**]
        T_FETCH[**web_fetch<br/>网页抓取**]
        T_BROWSER[**browser<br/>浏览器控制**]
        T_IMAGE[**image_gen<br/>图片生成**]
        T_CRON[**cron<br/>定时任务**]
        T_PLUGIN[**插件工具<br/>Plugin Tools**]
        T_CHANNEL[**渠道工具<br/>Channel Tools**]
    end

    subgraph "**会话管理**"
        SESS[**Session Store<br/>config/sessions.ts**]
        TRANSCRIPT[**Transcript<br/>JSONL 文件**]
        TOKEN[**Token Tracking<br/>用量追踪**]
    end

    CMD --> RUNNER
    GWMETHOD --> RUNNER
    RUNNER --> ATTEMPT
    RUNNER --> COMPACT

    ATTEMPT --> SYSPROMPT
    ATTEMPT --> SKILLLOAD
    ATTEMPT --> TOOLREG
    ATTEMPT --> MODELCFG

    TOOLREG --> T_SEARCH
    TOOLREG --> T_FETCH
    TOOLREG --> T_BROWSER
    TOOLREG --> T_IMAGE
    TOOLREG --> T_CRON
    TOOLREG --> T_PLUGIN
    TOOLREG --> T_CHANNEL

    RUNNER --> SESS
    SESS --> TRANSCRIPT
    SESS --> TOKEN

    style CMD fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style GWMETHOD fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000

    style RUNNER fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style ATTEMPT fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style COMPACT fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000

    style SYSPROMPT fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style SKILLLOAD fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style TOOLREG fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style MODELCFG fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000

    style T_SEARCH fill:#fff3e1,stroke:#333,stroke-width:2px,color:#000
    style T_FETCH fill:#fff3e1,stroke:#333,stroke-width:2px,color:#000
    style T_BROWSER fill:#fff3e1,stroke:#333,stroke-width:2px,color:#000
    style T_IMAGE fill:#fff3e1,stroke:#333,stroke-width:2px,color:#000
    style T_CRON fill:#fff3e1,stroke:#333,stroke-width:2px,color:#000
    style T_PLUGIN fill:#fff3e1,stroke:#333,stroke-width:2px,color:#000
    style T_CHANNEL fill:#fff3e1,stroke:#333,stroke-width:2px,color:#000

    style SESS fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style TRANSCRIPT fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style TOKEN fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
```

## 3. Agent 生命周期时序图

```mermaid
sequenceDiagram
    participant USER as **用户请求**
    participant RUNNER as **Pi Embedded Runner**
    participant CTX as **上下文构建器**
    participant LLM as **LLM Provider**
    participant TOOLSYS as **工具系统**
    participant STORE as **Session Store**

    USER->>RUNNER: **1. 触发 Agent 运行<br/>message + sessionKey**

    rect rgb(240, 248, 255)
    Note over RUNNER,CTX: **Phase 1: 初始化**
    RUNNER->>CTX: **2. 加载 Skills 快照**
    CTX-->>RUNNER: **3. 返回 Skills 列表**
    RUNNER->>CTX: **4. 构建 System Prompt<br/>注入 Skills + 规则**
    CTX-->>RUNNER: **5. 返回完整 Prompt**
    RUNNER->>CTX: **6. 注册工具集<br/>createOpenClawTools**
    CTX-->>RUNNER: **7. 返回工具定义**
    RUNNER->>CTX: **8. 解析模型配置<br/>Provider + Model + Thinking Level**
    CTX-->>RUNNER: **9. 返回模型配置**
    end

    rect rgb(240, 255, 240)
    Note over RUNNER,LLM: **Phase 2: 执行**
    RUNNER->>STORE: **10. 加载会话历史<br/>Transcript**
    STORE-->>RUNNER: **11. 返回历史消息**
    RUNNER->>LLM: **12. 发送请求<br/>System Prompt + History + Message + Tools**
    LLM-->>RUNNER: **13. 流式响应 - 文本片段**

    alt **LLM 请求工具调用**
        LLM-->>RUNNER: **14. tool_use 请求<br/>工具名 + 参数**
        RUNNER->>TOOLSYS: **15. 执行工具**
        TOOLSYS-->>RUNNER: **16. 工具结果**
        RUNNER->>LLM: **17. 返回工具结果<br/>继续对话**
        LLM-->>RUNNER: **18. 最终回复**
    end
    end

    rect rgb(255, 250, 240)
    Note over RUNNER,STORE: **Phase 3: 收尾**
    RUNNER->>STORE: **19. 保存 Transcript<br/>含 token 用量**
    RUNNER->>STORE: **20. 更新 Session 元数据**
    RUNNER-->>USER: **21. 返回最终结果**
    end
```

## 4. 工具系统架构

### 4.1 工具注册流程

工具在 `src/agents/openclaw-tools.ts` 中统一注册，分为以下几类：

| **工具类别** | **工具名** | **说明** | **实现文件** |
|:---:|:---|:---|:---|
| **Web 工具** | `web_search` | 网页搜索 | `tools/web-search.ts` |
| **Web 工具** | `web_fetch` | 网页内容抓取 | `tools/web-fetch.ts` |
| **浏览器** | `browser` | 浏览器自动化控制 | `tools/browser-tool.ts` |
| **媒体** | `image_gen` | 图片生成 | `tools/image-gen.ts` |
| **系统** | `cron` | 定时任务管理 | `tools/cron-tool.ts` |
| **会话** | `session_status` | 会话状态查询 | `tools/session-status.ts` |
| **渠道** | `discord_actions` | Discord 操作 | `tools/channel-actions.ts` |
| **渠道** | `telegram_actions` | Telegram 操作 | `tools/channel-actions.ts` |
| **渠道** | `slack_actions` | Slack 操作 | `tools/channel-actions.ts` |
| **插件** | 动态注册 | 插件注入的工具 | `plugins/tools.ts` |

### 4.2 工具执行流程

```mermaid
graph LR
    subgraph "**Agent 请求工具**"
        REQ[**tool_use 请求<br/>工具名 + JSON 参数**]
    end

    subgraph "**工具路由**"
        VALIDATE[**参数验证<br/>TypeBox Schema**]
        GATE[**权限检查<br/>Action Gating**]
        RESOLVE[**工具解析<br/>Registry Lookup**]
    end

    subgraph "**工具执行**"
        EXEC[**execute 函数**]
        RESULT[**结果处理<br/>图片清理 / 安全包装**]
    end

    subgraph "**结果返回**"
        STREAM[**流式推送<br/>tool 事件**]
        PERSIST[**持久化<br/>Transcript 记录**]
    end

    REQ --> VALIDATE
    VALIDATE --> GATE
    GATE --> RESOLVE
    RESOLVE --> EXEC
    EXEC --> RESULT
    RESULT --> STREAM
    RESULT --> PERSIST

    style REQ fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style VALIDATE fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style GATE fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style RESOLVE fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style EXEC fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style RESULT fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style STREAM fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style PERSIST fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
```

## 5. 会话管理

### 5.1 会话存储结构

```text
~/.openclaw/
├── sessions/
│   └── {agentId}/
│       ├── store.json           # 会话元数据索引
│       └── transcripts/
│           ├── {sessionId}.jsonl # 对话记录（JSONL 格式）
│           └── ...
```

### 5.2 会话 Key 解析规则

| **来源** | **Session Key 格式** | **示例** |
|:---|:---|:---|
| CLI 终端 | `global` | `global` |
| Telegram 私聊 | `telegram:{userId}` | `telegram:123456` |
| Telegram 群组 | `telegram:{groupId}:{threadId}` | `telegram:-1001:42` |
| WhatsApp | `whatsapp:{phoneNumber}` | `whatsapp:8613800138000` |
| Discord | `discord:{channelId}` | `discord:98765` |
| Slack | `slack:{channelId}` | `slack:C0123` |
| Gateway WebChat | `internal:{connectionId}` | `internal:uuid-xxx` |

### 5.3 会话操作

| **操作** | **说明** |
|:---|:---|
| `list` | 列出会话，支持按渠道/标签过滤 |
| `preview` | 预览会话 Transcript |
| `resolve` | 从各种标识符解析 Session Key |
| `patch` | 更新元数据（模型、思考级别等） |
| `reset` | 重置会话（新 SessionId、清除 Transcript） |
| `delete` | 删除会话和 Transcript 文件 |
| `compact` | 压缩 Transcript（用 LLM 摘要替换早期消息） |

## 6. 模型配置

OpenClaw 支持多种 LLM 提供商和模型，通过分层配置解析：

```mermaid
graph TB
    subgraph "**模型解析流程**"
        INPUT[**输入参数<br/>provider + model + thinking**]
        SESS_CFG[**会话配置<br/>Session 级别覆盖**]
        AGENT_CFG[**Agent 配置<br/>Agent 级别默认**]
        GLOBAL_CFG[**全局配置<br/>openclaw.json**]
        RESOLVE[**最终模型配置**]
    end

    subgraph "**支持的 Provider**"
        P1[**Anthropic<br/>Claude 系列**]
        P2[**OpenAI<br/>GPT 系列**]
        P3[**Google<br/>Gemini 系列**]
        P4[**AWS Bedrock**]
        P5[**Groq**]
        P6[**OpenRouter**]
        P7[**Copilot Proxy**]
    end

    INPUT --> SESS_CFG
    SESS_CFG --> AGENT_CFG
    AGENT_CFG --> GLOBAL_CFG
    GLOBAL_CFG --> RESOLVE

    RESOLVE --> P1
    RESOLVE --> P2
    RESOLVE --> P3
    RESOLVE --> P4
    RESOLVE --> P5
    RESOLVE --> P6
    RESOLVE --> P7

    style INPUT fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style SESS_CFG fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style AGENT_CFG fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style GLOBAL_CFG fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style RESOLVE fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000

    style P1 fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style P2 fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style P3 fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style P4 fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style P5 fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style P6 fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style P7 fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
```

## 7. 关键文件索引

| **文件** | **职责** |
|:---|:---|
| `src/commands/agent.ts` | Agent 命令执行入口 |
| `src/gateway/server-methods/agent.ts` | Gateway Agent 方法处理器 |
| `src/agents/pi-embedded-runner/run/attempt.ts` | 单次 Agent 运行尝试 |
| `src/agents/pi-embedded-runner/compact.ts` | Transcript 压缩 |
| `src/agents/system-prompt.ts` | System Prompt 构建 |
| `src/agents/openclaw-tools.ts` | 工具注册中心 |
| `src/agents/tools/common.ts` | 工具公共工具函数 |
| `src/config/sessions.ts` | 会话存储管理 |
| `src/plugins/tools.ts` | 插件工具加载 |
| `src/plugins/registry.ts` | 插件注册中心 |
