# 多 Agent 协作、编码能力与纠偏机制

## 1. 多 Agent 协作

### 1.1 概述

OpenClaw 支持**多 Agent 协作**，一个 Agent 可以通过 Subagent 系统**生成子代理**来委派任务，也可以通过消息工具**跨会话通信**。这不是简单的"Agent 列表"，而是真正的层级化任务分发机制。

| **属性** | **详情** |
|:---:|:---:|
| **子代理生成** | `sessions_spawn` 工具 |
| **跨会话通信** | `sessions_send` / `message` 工具 |
| **Agent 发现** | `agents_list` 工具 |
| **隔离方式** | 独立 Session + 专注 Prompt |
| **策略控制** | 白名单 + 沙箱限制 |

### 1.2 Subagent 架构图

```mermaid
graph TB
    subgraph SG1["主 Agent - Parent"]
        PA[**Parent Agent<br/>接收用户请求**]
        SPAWN[**sessions_spawn<br/>生成子代理工具**]
        SEND[**sessions_send<br/>跨会话消息工具**]
        LIST[**agents_list<br/>可用 Agent 列表**]
    end

    subgraph SG2["Subagent Registry"]
        REG[**subagent-registry.ts<br/>子代理注册中心**]
        TRACK[**运行追踪<br/>状态 / 超时 / 清理**]
    end

    subgraph SG3["子代理 - Subagents"]
        SA1[**Subagent 1<br/>专注任务 A**]
        SA2[**Subagent 2<br/>专注任务 B**]
        SA3[**Subagent 3<br/>专注任务 C**]
    end

    subgraph SG4["结果回传"]
        ANN[**subagent-announce.ts<br/>结果公告**]
        MERGE[**Parent 汇总结果<br/>整合回复用户**]
    end

    PA --> SPAWN
    PA --> SEND
    PA --> LIST

    SPAWN --> REG
    REG --> TRACK
    TRACK --> SA1
    TRACK --> SA2
    TRACK --> SA3

    SA1 --> ANN
    SA2 --> ANN
    SA3 --> ANN
    ANN --> MERGE
    MERGE --> PA

    style PA fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style SPAWN fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style SEND fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style LIST fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000

    style REG fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style TRACK fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000

    style SA1 fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style SA2 fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style SA3 fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000

    style ANN fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style MERGE fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
```

### 1.3 Subagent 生成与执行时序图

```mermaid
sequenceDiagram
    participant USER as **用户**
    participant PARENT as **Parent Agent**
    participant REGISTRY as **Subagent Registry**
    participant SUB as **Subagent**
    participant LLM as **LLM Provider**

    USER->>PARENT: **1. 复杂请求<br/>"分析这三个文件并汇总"**
    PARENT->>PARENT: **2. 判断需要分拆任务**

    rect rgb(240, 248, 255)
    Note over PARENT,REGISTRY: **Phase 1: 生成子代理**
    PARENT->>REGISTRY: **3. sessions_spawn<br/>task="分析文件A"<br/>model=claude / timeout=60s**
    REGISTRY->>REGISTRY: **4. 生成 Session Key<br/>agent:id:subagent:uuid**
    REGISTRY->>SUB: **5. 启动子代理<br/>buildSubagentSystemPrompt**
    end

    rect rgb(240, 255, 240)
    Note over SUB,LLM: **Phase 2: 子代理执行**
    SUB->>LLM: **6. 专注 Prompt<br/>"Stay focused - 分析文件A"**
    LLM-->>SUB: **7. 执行并返回结果**
    end

    rect rgb(255, 250, 240)
    Note over SUB,PARENT: **Phase 3: 结果回传**
    SUB->>REGISTRY: **8. 任务完成<br/>返回分析结果**
    REGISTRY->>PARENT: **9. subagent-announce<br/>公告结果给 Parent**
    end

    PARENT->>PARENT: **10. 汇总所有子代理结果**
    PARENT-->>USER: **11. 返回整合后的分析报告**
```

### 1.4 子代理 Session Key 格式

```text
agent:{agentId}:subagent:{uuid}
```

**示例**: `agent:default:subagent:a1b2c3d4-e5f6-7890-abcd-ef1234567890`

### 1.5 跨会话通信

| **工具** | **功能** | **实现文件** |
|:---|:---|:---|
| `sessions_spawn` | 生成子代理执行独立任务 | `src/agents/tools/sessions-spawn-tool.ts` |
| `sessions_send` | 向其他会话发送消息 | `src/agents/tools/sessions-send-tool.ts` |
| `message` | 通用消息发送（跨渠道/跨会话） | `src/agents/tools/message-tool.ts` |
| `agents_list` | 列出可用的 Agent 配置 | `src/agents/tools/agents-list-tool.ts` |

### 1.6 安全策略

| **策略** | **说明** |
|:---|:---|
| `tools.agentToAgent.enabled` | 是否启用 Agent 间通信 |
| `tools.agentToAgent.allow` | 允许通信的 Agent 白名单 |
| `subagents.allowAgents` | 子代理可以使用的 Agent 列表 |
| 沙箱隔离 | 沙箱 Agent 只能向自己生成的子会话发消息 |
| 清理策略 | 子代理完成后 delete 或 keep Session |

## 2. 编码能力

### 2.1 概述

OpenClaw 通过 **Pi Agent Core** 提供完整的代码读写和执行能力，支持文件操作、Shell 命令执行、进程管理，并具备沙箱隔离机制。

### 2.2 编码工具集

```mermaid
graph TB
    subgraph SG1["文件操作工具"]
        READ[**read<br/>读取文件内容**]
        WRITE[**write<br/>创建/覆盖文件**]
        EDIT[**edit<br/>精确编辑文件**]
        PATCH[**apply_patch<br/>多文件补丁**]
    end

    subgraph SG2["搜索定位工具"]
        GREP[**grep<br/>搜索文件内容**]
        FIND[**find<br/>按 Glob 查找文件**]
        LS[**ls<br/>列出目录内容**]
    end

    subgraph SG3["执行工具"]
        EXEC[**exec<br/>Shell 命令执行**]
        PROC[**process<br/>后台进程管理**]
        PTY[**PTY 模式<br/>交互式终端**]
    end

    subgraph SG4["沙箱隔离"]
        SANDBOX[**Sandbox<br/>Docker 容器**]
        PATHVAL[**路径验证<br/>防逃逸检查**]
        POLICY[**工具策略<br/>allowlist / denylist**]
    end

    subgraph SG5["Coding Agent 集成"]
        CODEX[**Codex CLI**]
        CLAUDE[**Claude Code**]
        PICODE[**Pi Coding Agent**]
        OPENCODE[**OpenCode**]
    end

    READ --> SANDBOX
    WRITE --> SANDBOX
    EDIT --> SANDBOX
    EXEC --> SANDBOX
    SANDBOX --> PATHVAL
    PATHVAL --> POLICY

    style READ fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style WRITE fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style EDIT fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style PATCH fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000

    style GREP fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style FIND fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style LS fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000

    style EXEC fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style PROC fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style PTY fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000

    style SANDBOX fill:#fff3e1,stroke:#333,stroke-width:2px,color:#000
    style PATHVAL fill:#fff3e1,stroke:#333,stroke-width:2px,color:#000
    style POLICY fill:#fff3e1,stroke:#333,stroke-width:2px,color:#000

    style CODEX fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style CLAUDE fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style PICODE fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style OPENCODE fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
```

### 2.3 编码工具详细说明

| **工具** | **功能** | **来源** |
|:---|:---|:---|
| `read` | 读取文件内容 | Pi Agent Core |
| `write` | 创建或覆盖文件 | Pi Agent Core |
| `edit` | 精确编辑文件（老文本替换为新文本） | Pi Agent Core |
| `apply_patch` | 多文件批量补丁 | Pi Agent Core |
| `grep` | 搜索文件内容（正则支持） | Pi Agent Core |
| `find` | 按 Glob 模式查找文件 | Pi Agent Core |
| `ls` | 列出目录内容 | Pi Agent Core |
| `exec` | 执行 Shell 命令 | Pi Agent Core |
| `process` | 管理后台进程 | Pi Agent Core |

### 2.4 代码编写时序图

```mermaid
sequenceDiagram
    participant USER as **用户**
    participant AGENT as **Agent**
    participant FS as **文件系统**
    participant SHELL as **Shell 执行器**

    USER->>AGENT: **1. "创建一个 Python Web 服务器"**

    rect rgb(240, 248, 255)
    Note over AGENT,FS: **Phase 1: 理解并规划**
    AGENT->>FS: **2. ls - 查看当前目录**
    FS-->>AGENT: **3. 返回目录内容**
    AGENT->>FS: **4. find - 查找已有文件**
    FS-->>AGENT: **5. 返回文件列表**
    end

    rect rgb(240, 255, 240)
    Note over AGENT,FS: **Phase 2: 编写代码**
    AGENT->>FS: **6. write - 创建 server.py**
    FS-->>AGENT: **7. 文件创建成功**
    AGENT->>FS: **8. write - 创建 requirements.txt**
    FS-->>AGENT: **9. 文件创建成功**
    end

    rect rgb(255, 250, 240)
    Note over AGENT,SHELL: **Phase 3: 安装依赖并运行**
    AGENT->>SHELL: **10. exec - pip install -r requirements.txt**
    SHELL-->>AGENT: **11. 安装完成**
    AGENT->>SHELL: **12. exec - python server.py**
    SHELL-->>AGENT: **13. 服务器启动成功**
    end

    AGENT-->>USER: **14. "服务器已创建并运行在 localhost:8000"**
```

### 2.5 沙箱安全机制

| **机制** | **说明** |
|:---|:---|
| **路径验证** | 所有文件操作校验路径不超出 workspace 边界 |
| **工作区访问** | 可配置 `none`（禁止）/ `ro`（只读）/ `rw`（读写） |
| **Docker 沙箱** | 支持在 Docker 容器内运行命令 |
| **工具策略** | allowlist / denylist 控制可用工具 |
| **参数标准化** | 自动适配 Claude Code 参数格式到标准格式 |

## 3. 纠偏机制 — 持续干活不出现偏差

### 3.1 纠偏体系总览

```mermaid
graph TB
    subgraph SG1["Prompt 层 - 事前引导"]
        SYSP[**System Prompt<br/>安全规则 + 行为边界**]
        SOUL[**SOUL.md<br/>人格和行为准则**]
        SKILL[**Skills 引导<br/>按描述匹配 + 延迟加载**]
    end

    subgraph SG2["运行时 - 事中监控"]
        HOOK[**Hook 系统<br/>before/after 钩子**]
        TOKEN[**Token 管理<br/>上下文窗口控制**]
        COMPACT[**Session Compaction<br/>摘要压缩防遗忘**]
    end

    subgraph SG3["Subagent - 任务聚焦"]
        FOCUS[**专注 Prompt<br/>Stay focused 指令**]
        TIMEOUT[**超时控制<br/>防止无限运行**]
        CLEANUP[**自动清理<br/>任务完成后回收**]
    end

    subgraph SG4["验证层 - 事后校验"]
        VALIDATE[**参数验证<br/>所有输入 Schema 校验**]
        TOOLPOL[**工具策略<br/>可用工具白名单**]
        REPAIR[**自动修复<br/>Session 文件损坏修复**]
        RESET[**压缩失败重置<br/>resetSessionAfterCompactionFailure**]
    end

    SYSP --> HOOK
    SOUL --> HOOK
    SKILL --> HOOK
    HOOK --> COMPACT
    TOKEN --> COMPACT
    COMPACT --> VALIDATE
    FOCUS --> TIMEOUT
    TIMEOUT --> CLEANUP

    style SYSP fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style SOUL fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style SKILL fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000

    style HOOK fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style TOKEN fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style COMPACT fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000

    style FOCUS fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style TIMEOUT fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style CLEANUP fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000

    style VALIDATE fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style TOOLPOL fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style REPAIR fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style RESET fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
```

### 3.2 四层纠偏机制详解

#### 第一层：System Prompt 安全护栏（事前）

System Prompt 包含明确的安全规则，从源头约束 Agent 行为：

```text
# 安全规则（内置于 system-prompt.ts）
- 你没有独立目标：不追求自我保存、复制、资源获取或权力扩张
- 不操纵或说服任何人扩展访问或禁用安全措施
- 不复制自己或修改系统提示、安全规则、工具策略
  除非用户明确请求
```

此外，`SOUL.md` 文件定义 Agent 的人格和行为准则，可通过 Hook 系统动态覆盖。

#### 第二层：Session Compaction — 防遗忘机制（事中）

当对话历史接近上下文窗口限制时，Compaction 机制确保 Agent 不会"忘记"重要信息：

```mermaid
sequenceDiagram
    participant AGENT as **Agent Runtime**
    participant COMPACT as **Compaction 引擎**
    participant MEMORY as **Memory 系统**
    participant LLM as **LLM Provider**
    participant STORE as **Session Store**

    AGENT->>AGENT: **1. 检测 Token 接近上限<br/>reserveTokensFloor = 9000**

    rect rgb(240, 248, 255)
    Note over AGENT,MEMORY: **预压缩记忆刷新**
    AGENT->>MEMORY: **2. 触发记忆刷新<br/>memoryFlush.softThresholdTokens**
    MEMORY->>LLM: **3. Agentic Turn<br/>提取重要信息存入记忆**
    LLM-->>MEMORY: **4. 关键信息已保存**
    end

    rect rgb(240, 255, 240)
    Note over COMPACT,LLM: **执行 Compaction**
    COMPACT->>LLM: **5. 摘要旧消息<br/>保留核心上下文**
    LLM-->>COMPACT: **6. 返回压缩摘要**
    COMPACT->>STORE: **7. 替换旧消息为摘要<br/>保留最近消息完整**
    end

    rect rgb(255, 250, 240)
    Note over AGENT,AGENT: **继续执行**
    AGENT->>AGENT: **8. 使用压缩后的上下文继续<br/>关键信息不丢失**
    end
```

**Compaction 关键参数**：

| **参数** | **默认值** | **说明** |
|:---|:---:|:---|
| `reserveTokensFloor` | 9000 | 保留最近消息的最小 Token 数 |
| `memoryFlush.softThresholdTokens` | 可配置 | 触发记忆刷新的阈值 |
| `compactionCount` | 自动跟踪 | 已执行的压缩次数 |
| `mode` | `default` | 压缩模式：default / safeguard |

#### 第三层：Subagent 专注约束

子代理通过专门的 Prompt 保持任务聚焦：

```text
# buildSubagentSystemPrompt 生成的 Prompt：
Stay focused - Do your assigned task, nothing else.
不要偏离分配的任务，不要主动扩展范围。
```

同时配合**超时控制**和**自动清理**：
- 每个子代理可设置独立超时时间
- 完成后根据策略自动 delete 或归档 Session

#### 第四层：验证与自动修复（事后）

| **机制** | **触发条件** | **处理方式** |
|:---|:---|:---|
| **参数验证** | 每次 Gateway 方法调用 | Schema 校验，拒绝无效输入 |
| **工具策略验证** | 每次工具调用 | `resolveEffectiveToolPolicy` 检查白名单 |
| **配置验证** | 配置加载时 | `validateConfigObject` 确保配置合法 |
| **Session 文件修复** | 文件损坏时 | `session-file-repair.ts` 自动修复 |
| **写锁保护** | 并发写入时 | `session-write-lock.ts` 防止冲突 |
| **压缩失败重置** | Compaction 异常 | `resetSessionAfterCompactionFailure` 重置 |
| **历史截断** | Provider 限制 | `limitHistoryTurns` 适配不同模型 |

### 3.3 纠偏时序图 - 完整生命周期

```mermaid
sequenceDiagram
    participant USER as **用户请求**
    participant PROMPT as **System Prompt**
    participant AGENT as **Agent Runtime**
    participant HOOKS as **Hook 系统**
    participant VALID as **验证层**
    participant COMPACT as **Compaction**

    rect rgb(240, 248, 255)
    Note over PROMPT,AGENT: **事前: Prompt 引导**
    PROMPT->>AGENT: **1. 注入安全规则**
    PROMPT->>AGENT: **2. 注入 SOUL.md 人格**
    PROMPT->>AGENT: **3. 注入 Skills 描述**
    end

    USER->>AGENT: **4. 用户请求**

    rect rgb(240, 255, 240)
    Note over AGENT,HOOKS: **事中: Hook 监控**
    HOOKS->>AGENT: **5. before_agent_start<br/>注入额外上下文**
    AGENT->>VALID: **6. 工具调用 - 策略校验**
    VALID-->>AGENT: **7. 验证通过/拒绝**
    HOOKS->>AGENT: **8. before_tool_call<br/>可拦截工具调用**
    HOOKS->>AGENT: **9. after_tool_call<br/>记录工具结果**
    end

    rect rgb(255, 250, 240)
    Note over AGENT,COMPACT: **Token 管理**
    AGENT->>AGENT: **10. 检测 Token 用量**
    AGENT->>COMPACT: **11. 触发 Compaction<br/>记忆刷新 + 摘要压缩**
    COMPACT-->>AGENT: **12. 上下文压缩完成**
    end

    AGENT-->>USER: **13. 返回结果**

    rect rgb(245, 240, 255)
    Note over HOOKS,HOOKS: **事后: Hook 记录**
    HOOKS->>HOOKS: **14. agent_end<br/>分析完成的对话**
    HOOKS->>HOOKS: **15. message_sent<br/>记录发送的消息**
    end
```

## 4. 关键文件索引

| **文件** | **职责** |
|:---|:---|
| `src/agents/subagent-registry.ts` | 子代理注册中心 |
| `src/agents/subagent-announce.ts` | 子代理结果公告 |
| `src/agents/tools/sessions-spawn-tool.ts` | 子代理生成工具 |
| `src/agents/tools/sessions-send-tool.ts` | 跨会话消息工具 |
| `src/agents/tools/message-tool.ts` | 通用消息工具 |
| `src/agents/tools/agents-list-tool.ts` | Agent 列表工具 |
| `src/agents/pi-tools.ts` | Pi Agent Core 工具创建 |
| `src/agents/pi-tools.read.ts` | 文件操作封装 |
| `src/agents/sandbox/` | 沙箱隔离实现 |
| `src/agents/system-prompt.ts` | System Prompt 构建（含安全规则） |
| `src/agents/pi-embedded-runner/compact.ts` | Session Compaction |
| `src/config/sessions.ts` | Session 存储管理 |
| `skills/coding-agent/SKILL.md` | Coding Agent Skill |
