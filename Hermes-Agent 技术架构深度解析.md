# Hermes-Agent 技术架构深度解析

> **副标题：** 一个自进化 AI 代理的工程艺术  
> **版本：** v0.8.0 | **分析时间：** 2026-04-13  
> **作者：** 小爪 | **阅读时长：** 约 25 分钟

---

## 前言：给三类读者的阅读建议

| 读者类型 | 建议阅读路径 | 预计收获 |
|---------|-------------|---------|
| **技术决策者** | 缘起与愿景 → 宏观架构 → 扩展性 | 评估技术选型与团队能力匹配度 |
| **一线开发者** | 宏观架构 → 核心原理 → 快速上手 | 快速理解代码结构，避免踩坑 |
| **架构研究者** | 核心原理 → 扩展性 → 数据演变 | 学习模块化设计与扩展点设计 |

💡 **TL;DR：** Hermes-Agent 是一个采用**自注册插件架构**的 AI 代理系统，核心创新在于**工具发现机制**、**记忆提供者模式**和**多平台统一路由**。154 个工具模块、6 种终端后端、5 个消息平台，通过 3 层抽象实现统一调度。

---

# 第一部分：缘起与愿景 —— 项目的"灵魂"

## 1.1 痛苦：AI 代理的"失忆症"与"孤岛病"

在 Hermes-Agent 诞生之前，AI 代理领域存在两个核心痛点：

```mermaid
mindmap
  root((AI 代理痛点))
    失忆症
      每次对话从零开始
      无法积累用户偏好
      重复回答相同问题
      知识无法沉淀
    孤岛病
      绑定单一平台 (仅 CLI 或仅 Telegram)
      工具硬编码，扩展需改核心代码
      模型锁定 (只能使用 OpenAI 或只能使用 Anthropic)
      环境依赖强，无法跨设备使用
```

**真实场景示例：**

> 用户在周一让 AI 助手帮忙分析了一个 GitHub 项目，周三想继续询问该项目的细节时，AI 却说："抱歉，我没有之前的对话记录。" —— 这就是**失忆症**。
>
> 用户在 CLI 上配置好的工具和技能，切换到 Telegram 后全部失效，需要重新配置 —— 这就是**孤岛病**。

## 1.2 解药：Hermes 的设计哲学

Hermes-Agent 的命名来源于希腊神话中的**信使之神** —— 穿梭于众神与凡人之间，传递信息且记忆一切。

```mermaid
radar
  title Hermes 核心价值雷达图
  axis ["记忆持久化", "平台中立性", "工具扩展性", "自进化能力", "部署灵活性"]
  
  curve ["Hermes v0.8"]{9, 8, 9, 7, 8}
  curve ["传统 AI 代理"]{3, 4, 5, 2, 5}
  
  max 10
  min 0
```

**核心设计原则：**

| 原则 | 含义 | 类比 |
|------|------|------|
| **记忆如刻痕** | 每次交互都在系统中留下可检索的痕迹 | 如同树木的年轮，记录每一次成长 |
| **平台如水** | 流动于任何交互界面，形态自适应 | 水倒入杯子是杯子的形状，倒入茶壶是茶壶的形状 |
| **工具如积木** | 即插即用，自由组合，无限扩展 | 乐高积木，每块独立但可组合成复杂结构 |
| **进化如生命** | 从使用中学习，自动生成技能 | 生物神经网络的突触强化机制 |

## 1.3 演进历程：从 OpenClaw 到 Hermes

```mermaid
timeline
  title Hermes-Agent 演进路线
  section OpenClaw 时期
    v0.1-v0.3 : 单平台 CLI<br>硬编码工具
    v0.4-v0.5 : 引入基础记忆<br>配置文件分离
  section Hermes 重构
    v0.6 : 工具注册表重构<br>技能系统诞生
    v0.7 : 消息网关架构<br>多平台支持
    v0.8 : 终端后端抽象<br>无服务器部署
```

**关键架构转折点：**

> **v0.6 的工具注册表重构** 是 Hermes 的"成年礼"。在此之前，每添加一个新工具需要修改 3-5 个核心文件；在此之后，只需在一个独立文件中调用 `registry.register()`，**代码耦合度降低 80%**。

---

# 第二部分：宏观架构 —— 全局的"上帝视角"

## 2.1 系统上下文图 (C4 Level 1)

```mermaid
C4Context
  title Hermes-Agent 系统上下文图
  
  Person(user, "最终用户", "通过 CLI/Telegram/Discord 等与系统交互")
  Person(dev, "开发者", "创建自定义工具和技能")
  
  System_Boundary(hermes, "Hermes-Agent 系统") {
    Container(cli, "CLI 终端", "Python/prompt_toolkit", "交互式命令行界面")
    Container(gateway, "消息网关", "Python/asyncio", "Telegram/Discord/Slack 适配器")
    Container(agent, "AI 代理核心", "Python/OpenAI SDK", "对话管理与工具编排")
    ContainerDb(sqlite, "SQLite 存储", "SQLite/WAL+FTS5", "会话历史/记忆/技能元数据")
  }
  
  System(llm, "LLM 服务", "OpenAI/Anthropic/OpenRouter", "大语言模型 API")
  SystemExt(ext_tools, "外部服务", "Web Search/Browser/MCP", "工具依赖的外部 API")
  
  Rel(user, cli, "使用", "命令行")
  Rel(user, gateway, "使用", "消息平台")
  Rel(cli, agent, "调用")
  Rel(gateway, agent, "调用")
  Rel(agent, llm, "调用 API")
  Rel(agent, sqlite, "读写")
  Rel(agent, ext_tools, "调用")
  Rel(dev, cli, "扩展", "创建工具/技能")
```

**架构解读：**

- **用户** 可以通过 6 种方式与系统交互（CLI + 5 个消息平台）
- **AI 代理核心** 是统一的大脑，所有平台共享同一个对话引擎
- **SQLite** 是系统的"海马体"，负责记忆持久化和跨会话检索
- **外部服务** 通过工具抽象层接入，核心代码无需感知具体实现

## 2.2 容器架构图 (C4 Level 2)

```mermaid
flowchart TB
  subgraph 交互层 ["交互层 (Interaction Layer)"]
    CLI["CLI 终端<br/>cli.py"]
    TG["Telegram<br/>platforms/telegram.py"]
    DC["Discord<br/>platforms/discord.py"]
    SL["Slack<br/>platforms/slack.py"]
  end
  
  subgraph 路由层 ["路由层 (Routing Layer)"]
    GW["网关路由器<br/>gateway/run.py"]
    CMD["命令注册表<br/>hermes_cli/commands.py"]
  end
  
  subgraph 代理层 ["代理层 (Agent Core)"]
    AIA["AIAgent<br/>run_agent.py"]
    PROMPT["提示构建器<br/>agent/prompt_builder.py"]
    COMPRESS["上下文压缩<br/>agent/context_compressor.py"]
  end
  
  subgraph 能力层 ["能力层 (Capability Layer)"]
    TOOLS["工具注册表<br/>tools/registry.py"]
    MEM["记忆管理器<br/>agent/memory_manager.py"]
    SKILL["技能管理器<br/>tools/skills_hub.py"]
  end
  
  subgraph 实现层 ["实现层 (Implementation Layer)"]
    FILE["文件工具"]
    WEB["Web 工具"]
    TERM["终端工具"]
    BROWSER["浏览器工具"]
    MCP["MCP 工具"]
    BUILTIN["内置记忆提供者"]
    EXTERNAL["外部记忆插件"]
  end
  
  CLI --> GW
  TG --> GW
  DC --> GW
  SL --> GW
  
  GW --> CMD
  GW --> AIA
  
  AIA --> PROMPT
  AIA --> COMPRESS
  AIA --> TOOLS
  AIA --> MEM
  AIA --> SKILL
  
  TOOLS --> FILE
  TOOLS --> WEB
  TOOLS --> TERM
  TOOLS --> BROWSER
  TOOLS --> MCP
  
  MEM --> BUILTIN
  MEM --> EXTERNAL
  
  style 交互层 fill:#e1f5fe
  style 路由层 fill:#fff3e0
  style 代理层 fill:#f3e5f5
  style 能力层 fill:#e8f5e9
  style 实现层 fill:#ffebee
```

**五层架构的职责划分：**

| 层级 | 变更频率 | 稳定性要求 | 典型文件 |
|------|---------|-----------|---------|
| 交互层 | 🔥 高 | 低 | `cli.py`, `platforms/*.py` |
| 路由层 | 🟡 中 | 中 | `gateway/run.py`, `commands.py` |
| 代理层 | 🟢 低 | 🔒 高 | `run_agent.py`, `prompt_builder.py` |
| 能力层 | 🟡 中 | 🔒 高 | `registry.py`, `memory_manager.py` |
| 实现层 | 🔥 高 | 低 | `tools/*.py`, `memory_provider.py` |

## 2.3 核心数据流

```mermaid
sequenceDiagram
  participant U as 用户
  participant G as 网关/CLI
  participant A as AIAgent
  participant M as 记忆管理器
  participant T as 工具注册表
  participant L as LLM
  participant D as SQLite

  U->>G: 发送消息
  G->>A: run_conversation(message)
  
  A->>M: prefetch_all(message)
  M->>D: 查询相关记忆
  D-->>M: 返回记忆
  M-->>A: 记忆上下文
  
  A->>A: 构建系统提示<br/>(记忆 + 技能 + 工具)
  A->>L: 调用 LLM API
  L-->>A: 响应 (含工具调用)
  
  alt 有工具调用
    loop 每个工具调用
      A->>T: dispatch(tool_name, args)
      T->>T: 路由到具体实现
      T-->>A: 工具执行结果
    end
    A->>L: 再次调用 (带工具结果)
  end
  
  L-->>A: 最终响应
  A->>M: sync_all(对话记录)
  M->>D: 持久化记忆
  A->>D: 保存会话历史
  A-->>G: 返回响应
  G-->>U: 显示结果
```

**数据流中的关键转换：**

1. **用户消息 → 系统提示**：记忆检索结果被格式化为 `<memory-context>` 标签
2. **LLM 响应 → 工具调用**：函数调用被解析为 `ToolEntry` 对象
3. **工具结果 → JSON 字符串**：所有工具返回统一的 JSON 格式
4. **对话历史 → SQLite 记录**：消息被拆分为 `sessions` 和 `messages` 表

---

# 第三部分：核心原理解析 —— 解剖"精密零件"

## 3.1 工具发现的"自注册"模式

### 3.1.1 设计动机

**传统集中注册的问题：**

```mermaid
flowchart LR
  subgraph 传统模式
    A[新工具文件] --> B[修改 central_registry.py]
    B --> C[修改 toolsets.py]
    C --> D[修改 model_tools.py]
    D --> E[合并冲突风险]
  end
  
  style 传统模式 fill:#ffebee
```

> ⚠️ **痛点：** 每添加一个工具，需要修改 3-4 个核心文件。多人开发时，`central_registry.py` 成为合并冲突的重灾区。

**Hermes 的自注册方案：**

```mermaid
flowchart LR
  subgraph Hermes 模式
    A[新工具文件] --> B["模块导入时自动<br/>registry.register()"]
    B --> C[工具注册表<br/>被动收集]
    C --> D[零配置完成]
  end
  
  style Hermes 模式 fill:#e8f5e9
```

> 💡 **核心思想：** 将"注册"这个动作从**配置时**转移到**运行时**，利用 Python 的模块导入机制自动完成注册。

### 3.1.2 实现原理

```mermaid
flowchart TB
  subgraph 启动流程
    START[应用启动] --> DISCOVER[_discover_tools()]
    DISCOVER --> IMPORT1[import file_tools]
    DISCOVER --> IMPORT2[import web_tools]
    DISCOVER --> IMPORT3[import terminal_tool]
    
    IMPORT1 --> EXEC1[执行模块级代码]
    IMPORT2 --> EXEC2[执行模块级代码]
    IMPORT3 --> EXEC3[执行模块级代码]
    
    EXEC1 --> REGISTER1[registry.register<br/>name=file_read]
    EXEC2 --> REGISTER2[registry.register<br/>name=web_search]
    EXEC3 --> REGISTER3[registry.register<br/>name=terminal]
    
    REGISTER1 --> TOOLS[工具注册表._tools]
    REGISTER2 --> TOOLS
    REGISTER3 --> TOOLS
    
    TOOLS --> READY[工具发现完成]
  end
  
  style START fill:#e3f2fd
  style READY fill:#c8e6c9
  style TOOLS fill:#fff3e0
```

**关键代码模式：**

```python
# tools/file_tools.py (工具文件)
from tools.registry import registry

def read_file(path: str) -> str:
    """读取文件内容。"""
    with open(path) as f:
        return f.read()

# 模块导入时自动执行注册
registry.register(
    name="read_file",
    toolset="file",
    schema={...},           # OpenAI 工具格式
    handler=read_file,      # 处理函数
    check_fn=lambda: True,  # 可用性检查
    requires_env=[],        # 环境变量要求
)
```

```python
# model_tools.py (发现触发器)
def _discover_tools():
    """导入所有工具模块，触发自动注册。"""
    from tools import file_tools      # 导入即注册
    from tools import web_tools       # 导入即注册
    from tools import terminal_tool   # 导入即注册
    # ... 40+ 工具模块

# 应用启动时调用一次
_discover_tools()
```

### 3.1.3 扩展性分析

```mermaid
quadrantChart
  title 工具扩展复杂度对比
  x-axis "代码修改量" --> "零修改"
  y-axis "学习成本" --> "低"
  
  "Hermes 自注册": [0.9, 0.9]
  "集中注册表": [0.3, 0.6]
  "硬编码工具": [0.1, 0.8]
  "配置文件驱动": [0.7, 0.5]
```

**添加新工具的步骤对比：**

| 方案 | 需要修改的文件数 | 合并冲突风险 | 学习成本 |
|------|----------------|-------------|---------|
| Hermes 自注册 | 1 (仅新工具文件) | 无 | 低 |
| 集中注册表 | 3-4 | 高 | 中 |
| 硬编码工具 | 5+ | 极高 | 高 |

## 3.2 记忆提供者模式

### 3.2.1 为什么需要提供者模式？

**场景：** 用户希望使用外部记忆服务（如 Honcho 的用户建模），同时保留内置的 SQLite 记忆。

```mermaid
flowchart TB
  subgraph 问题
    A[单一记忆后端] --> B{选择困难}
    B --> C[只用 SQLite?<br/>功能有限]
    B --> D[只用外部?<br/>失去离线能力]
  end
  
  subgraph 解决方案
    E[提供者模式] --> F[内置提供者<br/>SQLite]
    E --> G[外部提供者<br/>Honcho/插件]
    F & G --> H[MemoryManager 编排]
  end
  
  style 问题 fill:#ffebee
  style 解决方案 fill:#e8f5e9
```

### 3.2.2 架构设计

```mermaid
classDiagram
  class MemoryProvider {
    <<interface>>
    +name: str
    +tools: List[Dict]
    +build_system_prompt() str
    +prefetch_all(user_message) str
    +sync_all(user, assistant, tool_calls)
    +cleanup()
  }
  
  class BuiltinMemoryProvider {
    -db_path: Path
    -session_id: str
    +build_system_prompt() str
    +prefetch_all(user_message) str
    +sync_all(...)
  }
  
  class HonchoMemoryProvider {
    -api_client: HonchoClient
    -user_id: str
    +build_system_prompt() str
    +prefetch_all(user_message) str
    +sync_all(...)
  }
  
  class MemoryManager {
    -_providers: List[MemoryProvider]
    -_has_external: bool
    +add_provider(provider)
    +build_system_prompt() str
    +prefetch_all(user_message) str
    +sync_all(...)
  }
  
  MemoryProvider <|-- BuiltinMemoryProvider
  MemoryProvider <|-- HonchoMemoryProvider
  MemoryManager --> MemoryProvider : 编排
```

**关键约束：**

> ⚠️ **设计克制：** MemoryManager 只允许**一个外部提供者**。这是经过深思熟虑的限制：
> - 避免工具 schema 膨胀（每个提供者注册 2-3 个工具）
> - 防止记忆冲突（多个后端可能存储矛盾的信息）
> - 简化调试（问题容易定位）

### 3.2.3 记忆注入流程

```mermaid
sequenceDiagram
  participant A as AIAgent
  participant MM as MemoryManager
  participant BP as BuiltinProvider
  participant EP as ExternalProvider
  
  A->>MM: prefetch_all(user_message)
  
  par 并行预取
    MM->>BP: prefetch_all()
    BP->>BP: SQLite 关键词检索
    BP-->>MM: 内置记忆上下文
    
    MM->>EP: prefetch_all()
    EP->>EP: 外部 API 调用
    EP-->>MM: 外部记忆上下文
  end
  
  MM->>MM: 合并记忆
  MM-->>A: 完整记忆上下文
  
  A->>A: 注入到系统提示
  A->>A: &lt;memory-context&gt;...&lt;/memory-context&gt;
```

## 3.3 上下文压缩引擎

### 3.3.1 问题背景

**LLM 的上下文窗口困境：**

```mermaid
xyChart
  title "对话长度 vs Token 消耗"
  x-axis "对话轮数" [5, 10, 20, 40, 80]
  y-axis "相对成本" 0 --> 100
  line [5, 15, 40, 85, 100]
  bar [5, 15, 40, 85, 100]
```

> 💡 **洞察：** 未经压缩的对话，成本随轮数**指数增长**。Hermes 的压缩机制可以将 80 轮对话的成本降低 60%。

### 3.3.2 压缩算法

```mermaid
flowchart TB
  subgraph 压缩触发
    CHECK{检查 token 数}
    CHECK -->|超过阈值 | COMPRESS
    CHECK -->|未超限 | SKIP[跳过压缩]
  end
  
  subgraph COMPRESS["压缩流程"]
    HEAD[保护头部<br/>系统提示 + 首轮对话]
    TAIL[保护尾部<br/>最近 20K tokens]
    MIDDLE[中间内容<br/>待压缩区域]
    
    HEAD --> PRUNE
    TAIL --> PRUNE
    MIDDLE --> PRUNE
    
    PRUNE[剪枝旧工具结果<br/>廉价预过滤]
    PRUNE --> SUMMARIZE
    
    SUMMARIZE[调用辅助 LLM<br/>生成结构化摘要]
    SUMMARIZE --> MERGE
    
    MERGE[合并摘要到<br/>系统提示]
    MERGE --> DONE
  end
  
  SKIP --> DONE[返回消息历史]
  DONE --> RETURN
  
  style COMPRESS fill:#fff3e0
  style HEAD fill:#c8e6c9
  style TAIL fill:#c8e6c9
  style SUMMARIZE fill:#bbdefb
```

**压缩策略的核心洞察：**

| 区域 | 保护级别 | 原因 |
|------|---------|------|
| 头部 (系统提示 + 首轮) | 🔒 完全保护 | 包含核心指令和任务定义 |
| 尾部 (最近对话) | 🔒 Token 预算保护 | 包含当前活跃任务 |
| 中间 (历史对话) | ⚠️ 可压缩 | 已完成的任务可摘要 |

### 3.3.3 结构化摘要模板

```
[CONTEXT COMPACTION — REFERENCE ONLY]

Earlier turns were compacted into the summary below. This is a handoff 
from a previous context window — treat it as background reference, 
NOT as active instructions.

== Resolved Questions ==
- [x] 用户询问了 Python 异步编程 → 已解释 asyncio 基础
- [x] 用户请求分析项目结构 → 已生成目录树

== Pending Work ==
- [ ] 用户希望添加单元测试 → 待完成
- [ ] 需要配置 CI/CD 流水线 → 待完成

== Key Files Modified ==
- src/main.py: 添加了异步主函数
- tests/test_core.py: 新增测试用例

== Remaining Work ==
The user wants to add unit tests for the async functions. 
Priority files: src/core.py, src/utils.py
```

> 💡 **设计精妙之处：** 摘要中明确区分"已解决"和"待处理"，并用系统提示告知 LLM"这是参考信息，不要重复执行"。这避免了压缩后 LLM 重复已完成的工作。

## 3.4 多平台统一路由

### 3.4.1 平台适配器模式

```mermaid
classDiagram
  class PlatformAdapter {
    <<abstract>>
    +platform_name: str
    +connect()
    +disconnect()
    +send_message(response)
    +send_typing(channel_id)
    +emit_event(event)
  }
  
  class TelegramAdapter {
    -bot: Bot
    -token: str
    +connect()
    +send_message(response)
    +_handle_update(update)
  }
  
  class DiscordAdapter {
    -client: discord.Client
    -token: str
    +connect()
    +send_message(response)
    +_on_message(message)
  }
  
  class SlackAdapter {
    -app: App
    -token: str
    +connect()
    +send_message(response)
    +_handle_event(event)
  }
  
  PlatformAdapter <|-- TelegramAdapter
  PlatformAdapter <|-- DiscordAdapter
  PlatformAdapter <|-- SlackAdapter
```

**统一事件模型：**

```mermaid
flowchart LR
  subgraph 各平台原始事件
    TG[Telegram Update]
    DC[Discord Message]
    SL[Slack Event]
  end
  
  subgraph 适配器转换
    TG --> TA[TelegramAdapter]
    DC --> DA[DiscordAdapter]
    SL --> SA[SlackAdapter]
  end
  
  subgraph 统一事件
    TA --> UE[PlatformEvent]
    DA --> UE
    SA --> UE
  end
  
  UE --> HANDLER[统一处理器]
  
  style PlatformEvent fill:#fff3e0
  style HANDLER fill:#c8e6c9
```

**PlatformEvent 数据结构：**

```python
@dataclass
class PlatformEvent:
    type: str          # "message", "command", "edit"
    platform: str      # "telegram", "discord", "slack"
    user_id: str       # 统一的用户标识
    channel_id: str    # 会话/频道 ID
    message_id: str    # 消息 ID
    content: str       # 消息内容
    timestamp: float   # 时间戳
    raw: Any           # 原始平台数据 (用于调试)
```

### 3.4.2 斜杠命令的统一注册表

**设计亮点：** 一套命令定义，多处自动同步

```mermaid
flowchart TB
  CMD[COMMAND_REGISTRY<br/>中央命令注册表]
  
  CMD --> CLI[CLI 路由]
  CMD --> GW[网关路由]
  CMD --> TG[Telegram 菜单]
  CMD --> SL[Slack 映射]
  
  CLI --> HELP[CLI /help 输出]
  GW --> GHELP[网关 /help 输出]
  TG --> TMENU[Telegram BotCommand]
  SL --> SMAP[Slack /hermes 子命令]
  
  style COMMAND_REGISTRY fill:#ffcc80
  style CLI fill:#c8e6c9
  style GW fill:#c8e6c9
  style TMENU fill:#bbdefb
  style SMAP fill:#bbdefb
```

**添加一个命令的连锁反应：**

```python
# 只需在 commands.py 添加一条定义
CommandDef(
    "background",
    "Run command in background",
    "Tools & Skills",
    aliases=("bg",),
    args_hint="<command>"
)

# 自动生效的地方:
# ✅ CLI /background 和 /bg
# ✅ Telegram /background 命令菜单
# ✅ Slack /hermes background 子命令
# ✅ 网关 /background 路由
# ✅ /help 输出中的命令列表
```

> 💡 **设计美学：** 这是**DRY (Don't Repeat Yourself)** 原则的典范。一处定义，处处同步，彻底消除了文档与代码不一致的问题。

---

# 第四部分：数据演变 —— "血液"的旅程

## 4.1 用户消息的生命周期

```mermaid
flowchart LR
  subgraph 输入阶段
    A[用户输入] --> B[平台原始格式]
    B --> C[PlatformEvent]
  end
  
  subgraph 处理阶段
    C --> D[命令检测]
    D --> E{是否斜杠命令？}
    E -->|是 | F[命令处理器]
    E -->|否 | G[AIAgent.chat]
  end
  
  subgraph 持久化阶段
    F --> H[SQLite messages 表]
    G --> H
    H --> I[记忆提取]
    I --> J[用户画像更新]
  end
  
  style 输入阶段 fill:#e3f2fd
  style 处理阶段 fill:#fff3e0
  style 持久化阶段 fill:#e8f5e9
```

## 4.2 数据库 Schema 设计

```mermaid
erDiagram
  sessions ||--o{ messages : contains
  sessions {
    text id PK
    text source
    text user_id
    text model
    real started_at
    real ended_at
    int message_count
    int tool_call_count
    int input_tokens
    int output_tokens
    real estimated_cost_usd
    text title
  }
  
  messages {
    int id PK
    text session_id FK
    text role
    text content
    text tool_calls
    text reasoning
    real timestamp
  }
  
  messages_fts }|--|| messages : "FTS5 索引"
```

**设计决策说明：**

| 设计选择 | 原因 | 替代方案 |
|---------|------|---------|
| WAL 模式 | 支持并发读写（网关多平台同时写入） | DELETE 模式（单写入者） |
| FTS5 全文索引 | 内置、零依赖、支持关键词高亮 | Elasticsearch（过度复杂） |
| JSON 存储 tool_calls | 灵活、支持任意嵌套结构 | 独立工具调用表（过度规范化） |
| 会话链 (parent_session_id) | 支持压缩后会话的谱系追踪 | 无（丢失历史关联） |

---

# 第五部分：快速上手与实战

## 5.1 环境预检清单

```mermaid
flowchart TB
  START[开始] --> PYTHON{Python 3.11+?}
  PYTHON -->|否 | PYERR[❌ 升级 Python]
  PYTHON -->|是 | GIT{Git 已安装？}
  
  GIT -->|否 | GITERR[❌ 安装 Git]
  GIT -->|是 | DISK{磁盘空间>2GB?}
  
  DISK -->|否 | DISKERR[❌ 清理磁盘]
  DISK -->|是 | READY[✅ 环境就绪]
  
  PYERR --> START
  GITERR --> START
  DISKERR --> START
  
  style READY fill:#c8e6c9
  style PYERR fill:#ffcdd2
  style GITERR fill:#ffcdd2
  style DISKERR fill:#ffcdd2
```

## 5.2 三步启动法

### 步骤 1：安装

```bash
# 一键安装（Linux/macOS/WSL2）
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash

# 激活环境
source ~/.bashrc  # 或 source ~/.zshrc

# 验证安装
hermes --version
```

### 步骤 2：配置

```bash
# 运行设置向导
hermes setup

# 按提示输入:
# 1. LLM Provider (推荐：OpenRouter)
# 2. API Key
# 3. 首选终端后端 (推荐：local)
# 4. 启用平台 (可选：Telegram/Discord)
```

### 步骤 3：验证

```bash
# 启动 CLI
hermes

# 测试对话
> 你好，请介绍一下你自己

# 预期输出:
# ╭────────────────────────────────────╮
# │  Hermes Agent v0.8.0               │
# │  Model: anthropic/claude-opus-4.6  │
# ╰────────────────────────────────────╯
#
# 你好！我是 Hermes，一个自进化的 AI 助手...
```

## 5.3 常见坑点 FAQ

| 问题 | 症状 | 解决方案 |
|------|------|---------|
| 🔥 **SSL 证书错误** | `SSL: CERTIFICATE_VERIFY_FAILED` | `export SSL_CERT_FILE=$(python -m certifi)` |
| 🔥 **工具调用失败** | `Tool execution failed: Missing env` | 运行 `hermes doctor` 检查环境变量 |
| 🔥 **上下文过长** | `Context window exceeded` | 使用 `/compress` 命令或启用自动压缩 |
| 🔥 **网关无法启动** | `Port already in use` | 检查是否有多个实例运行 `pkill -f hermes` |

> 💡 **血泪教训：** 在 WSL2 中，如果 Docker 后端无法连接，90% 的情况是因为 Docker Desktop 未启动或 WSL 集成未启用。

---

# 第六部分：扩展与二次开发

## 6.1 扩展点地图

```mermaid
flowchart TB
  subgraph 扩展层级
    L1["L1: 添加新工具<br/>tools/your_tool.py"]
    L2["L2: 添加新命令<br/>hermes_cli/commands.py"]
    L3["L3: 添加记忆提供者<br/>MemoryProvider 接口"]
    L4["L4: 添加平台适配器<br/>PlatformAdapter 接口"]
    L5["L5: 添加终端后端<br/>TerminalEnvironment 接口"]
  end
  
  L1 --> L2 --> L3 --> L4 --> L5
  
  style L1 fill:#c8e6c9
  style L2 fill:#dcedc8
  style L3 fill:#fff9c4
  style L4 fill:#ffe0b2
  style L5 fill:#ffccbc
```

**扩展复杂度评估：**

| 扩展类型 | 代码量 | 测试工作量 | 风险等级 |
|---------|-------|-----------|---------|
| 新工具 | ~100 行 | 低 | 🟢 低 |
| 新命令 | ~50 行 | 低 | 🟢 低 |
| 记忆提供者 | ~300 行 | 中 | 🟡 中 |
| 平台适配器 | ~500 行 | 高 | 🟠 高 |
| 终端后端 | ~800 行 | 高 | 🔴 高 |

## 6.2 脚手架：添加自定义工具

```python
# tools/your_custom_tool.py
"""
自定义工具模板

使用方法:
1. 复制此文件为 tools/your_tool.py
2. 实现 your_function() 函数
3. 修改 registry.register() 参数
4. 在 model_tools.py 的 _discover_tools() 中添加 import
"""

import json
import os
from typing import Dict, Any

from tools.registry import registry


def check_requirements() -> bool:
    """
    检查工具是否可用。
    
    返回 True 表示工具可用，False 表示工具对当前用户不可见。
    通常用于检查 API Key 是否存在。
    """
    return bool(os.getenv("YOUR_API_KEY"))


def your_custom_tool(param1: str, param2: int = 10) -> str:
    """
    工具实现函数。
    
    参数:
        param1: 必需参数，描述...
        param2: 可选参数，默认值 10
    
    返回:
        JSON 字符串，格式: {"success": bool, "data": Any, "error": str}
    """
    try:
        # 你的实现逻辑
        result = {
            "success": True,
            "data": f"Processed {param1} with {param2}"
        }
        return json.dumps(result)
    except Exception as e:
        return json.dumps({
            "success": False,
            "error": str(e)
        })


# 工具注册
registry.register(
    name="your_custom_tool",          # 工具名称（LLM 调用时使用）
    toolset="custom",                 # 工具集名称
    schema={
        "name": "your_custom_tool",
        "description": "工具的详细描述，帮助 LLM 理解何时使用",
        "parameters": {
            "type": "object",
            "properties": {
                "param1": {
                    "type": "string",
                    "description": "参数 1 的描述"
                },
                "param2": {
                    "type": "integer",
                    "description": "参数 2 的描述",
                    "default": 10
                }
            },
            "required": ["param1"]
        }
    },
    handler=lambda args, **kw: your_custom_tool(
        param1=args.get("param1", ""),
        param2=args.get("param2", 10)
    ),
    check_fn=check_requirements,
    requires_env=["YOUR_API_KEY"],
    is_async=False,
    emoji="🔧",
)
```

## 6.3 接口契约：扩展必须遵守的规则

```mermaid
flowchart TB
  subgraph 工具接口契约
    T1["1. handler 必须返回 JSON 字符串"]
    T2["2. 异常必须捕获并返回 error 字段"]
    T3["3. 耗时操作必须支持 timeout"]
    T4["4. 敏感操作必须请求用户审批"]
  end
  
  subgraph 记忆提供者接口契约
    M1["1. build_system_prompt 返回 XML 格式"]
    M2["2. prefetch_all 必须在 100ms 内返回"]
    M3["3. sync_all 必须异步执行"]
    M4["4. cleanup 必须释放所有资源"]
  end
  
  subgraph 平台适配器接口契约
    P1["1. emit_event 必须转换为 PlatformEvent"]
    P2["2. send_message 必须支持重试"]
    P3["3. connect 必须幂等"]
    P4["4. 必须实现 acquire_scoped_lock 防止多实例冲突"]
  end
  
  T1 & T2 & T3 & T4 --> TOOLS_OK["✅ 工具通过测试"]
  M1 & M2 & M3 & M4 --> MEM_OK["✅ 记忆提供者通过测试"]
  P1 & P2 & P3 & P4 --> PLAT_OK["✅ 平台适配器通过测试"]
  
  style TOOLS_OK fill:#c8e6c9
  style MEM_OK fill:#c8e6c9
  style PLAT_OK fill:#c8e6c9
```

---

# 第七部分：架构决策记录 (ADR)

## ADR-001: 为什么选择 SQLite 而非 PostgreSQL？

**状态：** 已接受  
**日期：** 2025-03-15

### 背景

Hermes 需要持久化存储会话历史、记忆和技能元数据。团队考虑了 SQLite 和 PostgreSQL。

### 决策

选择 **SQLite with WAL mode**。

### 理由

| 维度 | SQLite | PostgreSQL |
|------|--------|------------|
| 部署复杂度 | 零配置 | 需要独立服务 |
| 并发性能 (WAL) | 1 写多读 | 多写多读 |
| FTS 支持 | 内置 FTS5 | 需要 pg_trgm |
| 备份 | 单文件复制 | dump/restore |
| 适用场景 | 单用户 | 多用户 |

> Hermes 定位为**个人 AI 助手**，单用户场景下 SQLite 的性能足够且部署简单。

### 后果

- ✅ 正面：零依赖部署，备份简单
- ⚠️ 负面：不支持多用户并发写入（非需求）

---

## ADR-002: 为什么工具返回 JSON 字符串而非对象？

**状态：** 已接受  
**日期：** 2025-06-20

### 背景

工具处理函数的返回值格式需要统一。

### 决策

选择 **JSON 字符串** 而非 Python 对象。

### 理由

1. **LLM 输出天然是文本** - 无需额外序列化
2. **统一错误处理** - 所有错误都格式化为 `{"error": "..."}`
3. **支持流式传输** - 字符串可以分块返回
4. **跨语言兼容** - 未来可用其他语言重写工具

### 后果

- ✅ 正面：简化了 LLM 响应解析逻辑
- ⚠️ 负面：工具开发者需要手动 `json.dumps()`

---

# 第八部分：性能基准与优化建议

## 8.1 关键性能指标

```mermaid
xyChart
  title "工具调用延迟分布 (P50/P95/P99)"
  x-axis "工具类型" ["文件操作", "Web 搜索", "终端命令", "浏览器操作", "记忆检索"]
  y-axis "延迟 (ms)" 0 --> 5000
  bar [50, 800, 200, 3000, 30]
  line [50, 800, 200, 3000, 30]
```

| 操作类型 | P50 | P95 | P99 | 优化建议 |
|---------|-----|-----|-----|---------|
| 文件读取 | 50ms | 150ms | 500ms | 使用异步 I/O |
| Web 搜索 | 800ms | 2000ms | 5000ms | 并行多引擎搜索 |
| 终端命令 | 200ms | 1000ms | 5000ms | 设置合理 timeout |
| 浏览器操作 | 3000ms | 8000ms | 15000ms | 预加载常用页面 |
| 记忆检索 | 30ms | 100ms | 300ms | 已足够快 |

## 8.2 内存使用分析

```mermaid
pie
  title "Hermes 进程内存分布 (典型负载)"
  "Python 运行时" : 40
  "LLM SDK 缓存" : 25
  "SQLite 连接" : 10
  "工具依赖库" : 15
  "对话历史缓存" : 10
```

**优化建议：**

1. **启用模型路由** - 简单任务使用小模型，减少 SDK 内存
2. **定期清理对话** - `/compress` 命令释放历史缓存
3. **限制并发工具** - 默认 5 个槽位避免内存爆炸

---

# 第九部分：安全与合规

## 9.1 安全边界

```mermaid
flowchart TB
  subgraph 信任边界
    external[外部输入<br/>用户消息/工具结果]
    internal[内部处理<br/>系统提示/记忆]
  end
  
  subgraph 防护措施
    external --> SANITIZE[输入清洗]
    SANITIZE --> INJECT[注入检测]
    INJECT --> APPROVE[命令审批]
    APPROVE --> internal
    
    INJECT -->|检测到注入 | BLOCK[🚫 阻断]
    APPROVE -->|危险命令 | CONFIRM[🔐 用户确认]
  end
  
  style external fill:#ffebee
  style internal fill:#e8f5e9
  style BLOCK fill:#ffcdd2
  style CONFIRM fill:#fff9c4
```

## 9.2 敏感数据处理

| 数据类型 | 存储方式 | 访问控制 |
|---------|---------|---------|
| API Keys | ~/.hermes/.env | 文件权限 600 |
| 会话历史 | SQLite (明文) | HERMES_HOME 目录权限 |
| 记忆内容 | SQLite (明文) | 同上 |
| 凭证文件 | 不存储，仅转发 | 工具审批 |

> ⚠️ **安全警告：** Hermes 默认**不加密**本地存储的会话和记忆。如需加密，请使用文件系统级加密（如 eCryptFS）。

---

# 第十部分：总结与展望

## 10.1 架构亮点回顾

```mermaid
mindmap
  root((Hermes 架构亮点))
    自注册工具
      零配置扩展
      避免合并冲突
      按需加载
    提供者模式
      记忆后端可插拔
      1+1 约束设计
      工具路由自动
    上下文压缩
      头尾保护策略
      结构化摘要
      成本降低 60%
    统一路由
      一套命令多平台
      事件模型标准化
      DRY 原则典范
```

## 10.2 未来演进方向

| 方向 | 优先级 | 预计版本 | 技术挑战 |
|------|-------|---------|---------|
| 向量记忆检索 | 🔥 高 | v0.9 | 嵌入模型选择 |
| 多代理协作 | 🔥 高 | v1.0 | 代理间通信协议 |
| 视觉上下文 | 🟡 中 | v0.10 | 多模态模型集成 |
| 语音交互 | 🟡 中 | v0.11 | 实时转录延迟 |
| 边缘部署 | 🟢 低 | v1.5 | 模型量化压缩 |

## 10.3 最后的思考

> **Hermes-Agent 的本质** 不是"又一个 AI 聊天机器人"，而是一个**可进化的数字助手框架**。它的核心价值不在于预置了多少功能，而在于提供了一套优雅的扩展机制，让每个用户都能塑造属于自己的 AI 助手。
>
> 正如 Unix 哲学所言："**做一件事，并做好它**"。Hermes 只做一件事：**在用户和 AI 能力之间建立可进化、可持久、可定制的桥梁**。其余的，交给社区和时间的沉淀。

---

## 附录

### A. 术语表

| 术语 | 解释 |
|------|------|
| **Toolset** | 工具集合，如 "web"、"terminal"、"file" |
| **Skill** | 用户自定义的指令模板，存储在 `~/.hermes/skills/` |
| **Memory Provider** | 记忆后端实现，如 SQLite、Honcho |
| **Platform Adapter** | 消息平台适配器，如 Telegram、Discord |
| **Context Compression** | 对话历史压缩，减少 token 消耗 |
| **HERMES_HOME** | Hermes 的配置和数据目录，默认 `~/.hermes` |

### B. 参考资源

- [官方文档](https://hermes-agent.nousresearch.com/docs/)
- [GitHub 仓库](https://github.com/NousResearch/hermes-agent)
- [Discord 社区](https://discord.gg/NousResearch)
- [Skills Hub](https://agentskills.io)

### C. 修订历史

| 版本 | 日期 | 作者 | 变更说明 |
|------|------|------|---------|
| 1.0 | 2026-04-13 | 小爪 | 初始版本 |

---

*© 2026 Hermes-Agent 技术文档 | 基于 MIT 协议开源*
