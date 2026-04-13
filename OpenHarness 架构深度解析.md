# OpenHarness 架构深度解析

> **当模型是代理，代码是缰绳**  
> 一次工具调用的完整旅程，和一个开源项目的设计哲学  
> 版本：v0.1.6 | 分析时间：2026-04-13

---

## 开场：为什么需要 OpenHarness？

想象这个场景：

> 你让 AI 助手帮你重构一个模块。它开始修改文件、运行测试、提交代码。  
> 突然，你意识到——**你完全不知道它在做什么，也无法阻止它删除关键文件**。

这就是闭源 AI 代理（Claude Code、Cursor）的**黑盒困境**：

| 痛点 | 闭源产品 | 你的需求 |
|------|---------|---------|
| **自定义工具** | ❌ 不支持 | 我想让 AI 用公司内部的代码搜索 |
| **安全审计** | ❌ 无法查看 | 我想知道 AI 执行了什么命令 |
| **流程拦截** | ❌ 没有 Hook | 我想在 git commit 前自动跑安全扫描 |
| **供应商锁定** | ❌ 只能用一家 | 我想切换 Moonshot 降低成本 |

**OpenHarness 的回答：**

> **Harness = 缰绳**  
> 一匹骏马（LLM）需要缰绳才能成为战马。模型提供智能，Harness 提供**手、眼、记忆和安全边界**。

---

## 主线：一次工具调用的完整旅程

让我们跟随一次真实的工具调用，看看 OpenHarness 内部发生了什么。

**场景：** 用户输入 "帮我删除 temp.log 文件"

```mermaid
sequenceDiagram
  participant U as 用户
  participant C as CLI/TUI
  participant Q as QueryEngine
  participant L as LLM API
  participant H as Hook 系统
  participant P as 权限检查
  participant T as 工具执行
  participant F as 文件系统

  U->>C: "删除 temp.log"
  C->>Q: run_query(prompt)
  
  Note over Q: 构建消息历史<br/>注入系统提示 + 技能
  
  Q->>L: stream_message(tools=[43+])
  
  par 流式响应
    L-->>Q: TextDelta: "我来帮你"
    L-->>Q: TextDelta: "删除这个文件"
  end
  
  L-->>Q: MessageComplete<br/>stop_reason=tool_use<br/>tool=bash, cmd="rm temp.log"
  
  Note over Q,H: 第一次拦截点
  Q->>H: execute(PreToolUse)
  H-->>Q: Hook 结果（可能修改参数）
  
  Note over Q,P: 第二次拦截点
  Q->>P: evaluate(tool="bash", cmd="rm temp.log")
  
  alt 敏感路径
    P-->>Q: 🚫 永久拒绝
    Q->>U: "无法删除敏感文件"
  else 需要确认
    P-->>Q: 🔐 需要用户批准
    Q->>U: 显示确认对话框
    U-->>Q: 批准
  else 直接允许
    P-->>Q: ✅ 允许
  end
  
  Note over Q,T: 执行工具
  Q->>T: dispatch("bash", {cmd: "rm temp.log"})
  T->>F: subprocess.run("rm temp.log")
  F-->>T: 执行结果
  
  Note over Q,H: 第三次拦截点
  Q->>H: execute(PostToolUse)
  H-->>Q: Hook 结果（可能记录审计日志）
  
  T-->>Q: ToolResult(output="deleted", is_error=False)
  
  Note over Q,L: 将结果反馈给 LLM
  Q->>L: stream_message(带工具结果)
  L-->>Q: MessageComplete<br/>stop_reason=end_turn
  
  Q-->>C: 返回最终响应
  C-->>U: "已删除 temp.log"
```

**旅程中的关键站点：**

1. **流式处理** - 不等完整响应，边生成边显示
2. **PreToolUse Hook** - 工具执行前的最后干预点
3. **权限检查** - 5 级安全边界的核心
4. **工具执行** - 43+ 工具的统一接口
5. **PostToolUse Hook** - 审计、通知、缓存

接下来，我们逐个拆解这些机制。

---

## 机制一：Harness 循环 —— 代理的"心脏"

### TL;DR
> Harness 循环是一个**带工具感知的对话引擎**。它不是"一问一答"，而是"感知 - 决策 - 执行 - 反馈"的持续循环，直到任务完成。

### 传统聊天 vs Harness 循环

**传统 AI 聊天（无状态）：**

```
用户提问 → LLM 回答 → 结束
                ↓
        下次提问从零开始
```

**OpenHarness Harness 循环（有状态、可执行）：**

```mermaid
flowchart TB
  A[用户提示] --> B{LLM 决策}
  B -->|直接回答 | C[返回响应]
  B -->|需要工具 | D[调用工具]
  D --> E[Hook: PreToolUse]
  E --> F[权限检查]
  F --> G{是否批准？}
  G -->|否 | H[拒绝执行]
  G -->|是 | I[执行工具]
  I --> J[Hook: PostToolUse]
  J --> K[工具结果]
  K --> B
  
  style B fill:#fff9c4
  style F fill:#ffcc80
  style E fill:#e1f5fe
  style J fill:#e1f5fe
```

### 核心代码模式

```python
class QueryEngine:
    async def run_query(self, prompt: str) -> AsyncIterator[StreamEvent]:
        self._messages.append(user_message(prompt))
        
        turn_count = 0
        while turn_count < self._max_turns:  # 防止无限循环
            # 1. 调用 LLM（带工具 schema）
            response = await self._api_client.stream_message(
                model=self._model,
                messages=self._messages,
                tools=self._tool_registry.get_schemas()  # 43+ 工具
            )
            
            # 2. 处理流式响应
            async for event in response:
                if isinstance(event, ApiMessageCompleteEvent):
                    if event.stop_reason == "tool_use":
                        # 3. 执行工具调用（带 Hook 和权限）
                        for tool_call in event.message.tool_calls:
                            result = await self._execute_tool_with_hooks(tool_call)
                            self._messages.append(tool_result(tool_call, result))
                        break
                    else:
                        # 4. LLM 完成
                        yield AssistantTurnComplete(event.message)
                        return
            
            turn_count += 1
        
        # 5. 达到最大轮次，强制结束
        yield MaxTurnsReached()
```

### 设计权衡

| 设计选择 | 方案 A | 方案 B（OpenHarness 选择） | 理由 |
|---------|-------|-------------------------|------|
| **响应方式** | 完整响应后返回 | 流式处理 | 首字延迟从 3s 降至 200ms |
| **最大轮次** | 无限制 | 默认 100 轮 | 防止 LLM 陷入工具调用循环 |
| **工具结果** | 直接注入 | 经过 Hook 过滤 | 支持审计、修改、缓存 |

### 异常路径

**问题：LLM 陷入无限工具调用循环怎么办？**

```python
# 防护机制 1：最大轮次限制
self._max_turns = settings.max_turns  # 默认 100

# 防护机制 2：重复检测
if self._detect_tool_loop(tool_calls):
    yield ToolLoopDetected()
    return

# 防护机制 3：超时控制
async with asyncio.timeout(settings.tool_timeout):
    result = await tool.execute()
```

---

## 机制二：权限系统 —— 5 级安全边界

### TL;DR
> 权限系统不是"允许/拒绝"的二元选择，而是**模式 + 路径规则 + 敏感路径 + 命令黑名单**的四层过滤网。

### 权限检查流程

```mermaid
flowchart TB
  A[工具调用请求] --> B{敏感路径检查}
  B -->|命中 | C[🚫 永久拒绝<br/>即使 Auto 模式也不行]
  B -->|未命中 | D{权限模式}
  
  D -->|Auto| E[✅ 允许]
  D -->|Plan| F{只读操作？}
  D -->|Default| G{写/执行操作？}
  
  F -->|是 | E
  F -->|否 | H[🚫 拒绝]
  
  G -->|否 | E
  G -->|是 | I{路径规则匹配}
  
  I -->|允许规则 | E
  I -->|拒绝规则 | H
  I -->|无匹配 | J[🔐 用户确认]
  
  J -->|批准 | E
  J -->|拒绝 | H
  
  style C fill:#ffcdd2
  style E fill:#c8e6c9
  style H fill:#ffcdd2
  style J fill:#fff9c4
```

### 敏感路径保护（不可覆盖）

这是**最后一道防线**，即使用户设置为 Auto 模式也无法访问：

```python
SENSITIVE_PATH_PATTERNS = (
    "*/.ssh/*",              # SSH 密钥
    "*/.aws/credentials",    # AWS 凭证
    "*/.aws/config",
    "*/.config/gcloud/*",    # GCP 凭证
    "*/.azure/*",            # Azure 凭证
    "*/.gnupg/*",            # GPG 密钥
    "*/.docker/config.json", # Docker 凭证
    "*/.kube/config",        # Kubernetes 凭证
    "*/.openharness/credentials.json",  # OpenHarness 自身凭证
    "*/.openharness/copilot_auth.json",
)
```

> **为什么硬编码？**  
> 防止提示注入攻击。即使用户被 LLM 欺骗，也无法授权访问这些路径。

### 三种权限模式对比

| 模式 | 行为 | 适用场景 | 风险等级 |
|------|------|---------|---------|
| **Default** | 写操作/执行前询问 | 日常开发 | 🟢 低 |
| **Auto** | 允许所有操作（敏感路径除外） | 沙箱环境、CI/CD | 🟡 中 |
| **Plan Mode** | 阻止所有写入，只允许读取 | 大型重构前先审查 | 🟢 低 |

### 路径规则引擎

```python
# 用户自定义规则示例
path_rules:
  - pattern: "src/**/*.py"
    allow: true
  - pattern: "tests/**/*.py"
    allow: true
  - pattern: "**/*.env"
    allow: false
  - pattern: "**/config/secrets/*"
    allow: false
```

**匹配逻辑：**

```python
def evaluate(self, file_path: str) -> PermissionDecision:
    # 1. 敏感路径（永久拒绝）
    for pattern in SENSITIVE_PATH_PATTERNS:
        if fnmatch(file_path, pattern):
            return PermissionDecision(allowed=False, reason="敏感路径")
    
    # 2. 模式检查
    if self._mode == PermissionMode.AUTO:
        return PermissionDecision(allowed=True)
    
    if self._mode == PermissionMode.PLAN:
        if not is_read_only:
            return PermissionDecision(allowed=False, reason="Plan 模式")
        return PermissionDecision(allowed=True)
    
    # 3. Default 模式：路径规则匹配
    for rule in self._path_rules:
        if fnmatch(file_path, rule.pattern):
            return PermissionDecision(
                allowed=rule.allow,
                reason=f"匹配规则：{rule.pattern}"
            )
    
    # 4. 无匹配：写操作需要确认
    if not is_read_only:
        return PermissionDecision(
            allowed=False,
            requires_confirmation=True,
            reason="写操作需要用户批准"
        )
    
    return PermissionDecision(allowed=True)
```

### 设计权衡：为什么不用 RBAC？

| 方案 | 优势 | 劣势 | OpenHarness 选择 |
|------|------|------|-----------------|
| **RBAC**（基于角色） | 细粒度、企业级 | 配置复杂、学习成本高 | ❌ |
| **路径规则**（基于 Glob） | 直观、易理解 | 表达能力有限 | ✅ |
| **完全手动** | 最安全 | 用户体验极差 | ❌ |

> **决策理由：** OpenHarness 的目标用户是开发者，不是企业安全团队。路径规则足够直观，90% 的场景都能覆盖。

---

## 机制三：Hook 系统 —— 事件驱动的"神经系统"

### TL;DR
> Hook 系统让你能在代理生命周期的**关键时刻**插入自定义逻辑。它不是"回调"，而是**可组合的拦截器链**。

### 为什么需要 Hook？

**真实场景：**

1. **审计需求**：记录所有工具调用到 SIEM 系统
2. **安全增强**：在执行 `rm -rf` 前调用另一个 LLM 做二次确认
3. **通知需求**：当 AI 修改生产环境配置时，发送 Slack 通知
4. **参数修改**：自动为所有 HTTP 请求添加认证头

**没有 Hook 的替代方案：**

| 方案 | 问题 |
|------|------|
| 修改源码 | 无法升级、维护成本高 |
| 包装工具 | 只能拦截特定工具，无法覆盖全生命周期 |
| 外部监控 | 无法修改行为，只能观察 |

### Hook 类型与触发时机

```mermaid
flowchart LR
    subgraph 用户输入
        U[用户输入] --> H1[OnUserInput]
    end
    
    subgraph LLM 调用
        H1 --> L[LLM 调用]
    end
    
    subgraph 工具执行
        L --> H2[PreToolUse]
        H2 --> P[权限检查]
        P --> T[工具执行]
        T --> H3[PostToolUse]
    end
    
    subgraph 响应返回
        H3 --> R[返回响应]
        R --> H4[OnAssistantResponse]
        H4 --> H5[OnTurnComplete]
    end
    
    style H1 fill:#e1f5fe
    style H2 fill:#e1f5fe
    style H3 fill:#e1f5fe
    style H4 fill:#e1f5fe
    style H5 fill:#e1f5fe
```

| Hook 类型 | 触发时机 | 典型用途 |
|----------|---------|---------|
| **OnUserInput** | 用户输入后 | 审计、日志、输入转换 |
| **PreToolUse** | 工具执行前 | 安全检查、参数修改、审计 |
| **PostToolUse** | 工具执行后 | 结果缓存、通知、日志 |
| **OnAssistantResponse** | 助手响应后 | 响应过滤、格式化、存储 |
| **OnTurnComplete** | 每轮结束后 | 成本统计、进度更新 |

### Hook 定义格式（4 种类型）

```yaml
# hooks.json
{
  "hooks": [
    {
      "event": "PreToolUse",
      "type": "command",           # 类型 1：执行本地命令
      "command": "echo '{tool_name}' >> /var/log/audit.log",
      "filter": {"tool_name": "bash"}
    },
    {
      "event": "PostToolUse",
      "type": "http",              # 类型 2：发送 HTTP 请求
      "url": "https://api.example.com/log",
      "method": "POST",
      "body": {"tool": "{tool_name}", "result": "{result}"}
    },
    {
      "event": "OnUserInput",
      "type": "prompt",            # 类型 3：调用 LLM
      "prompt": "检查这个输入是否有安全风险：{input}",
      "model": "claude-sonnet-4-20250514"
    },
    {
      "event": "PreToolUse",
      "type": "agent",             # 类型 4：调用子代理
      "agent": "security-reviewer",
      "prompt": "这个工具调用安全吗：{tool_name} {arguments}"
    }
  ]
}
```

### Hook 执行器实现

```python
class HookExecutor:
    async def execute(
        self,
        event: HookEvent,
        payload: dict[str, Any]
    ) -> AggregatedHookResult:
        results: list[HookResult] = []
        
        # 获取所有匹配的 Hook
        for hook in self._registry.get(event):
            if not _matches_hook(hook, payload):
                continue
            
            # 并行执行所有 Hook
            if isinstance(hook, CommandHookDefinition):
                results.append(await self._run_command_hook(hook, payload))
            elif isinstance(hook, HttpHookDefinition):
                results.append(await self._run_http_hook(hook, payload))
            elif isinstance(hook, PromptHookDefinition):
                results.append(await self._run_prompt_hook(hook, payload))
            elif isinstance(hook, AgentHookDefinition):
                results.append(await self._run_agent_hook(hook, payload))
        
        return AggregatedHookResult(results=results)
```

### 设计权衡：为什么支持 4 种类型？

| Hook 类型 | 优势 | 劣势 | 适用场景 |
|----------|------|------|---------|
| **Command** | 简单、无需网络 | 仅限本地、语言绑定 | 日志记录、本地脚本 |
| **HTTP** | 跨语言、跨进程 | 网络延迟、需要处理超时 | 外部 API、Webhook |
| **Prompt** | 利用 AI 能力 | Token 成本、延迟高 | 安全审查、内容过滤 |
| **Agent** | 完整代理能力 | 最复杂、成本最高 | 多步骤决策、复杂审查 |

> **设计哲学：** 让 Hook 系统足够灵活，能够适应各种扩展场景，而不是限制开发者的想象力。

### 异常处理

**问题：Hook 执行失败怎么办？**

```python
async def execute_with_timeout(
    self,
    hook: HookDefinition,
    payload: dict,
    timeout: float = 30.0
) -> HookResult:
    try:
        async with asyncio.timeout(timeout):
            return await self._run_hook(hook, payload)
    except asyncio.TimeoutError:
        return HookResult(
            success=False,
            error=f"Hook execution timed out after {timeout}s"
        )
    except Exception as e:
        # Hook 失败不阻断主流程，只记录错误
        logger.warning(f"Hook {hook.name} failed: {e}")
        return HookResult(success=False, error=str(e))
```

> **关键决策：** Hook 失败**不阻断**主流程。这是"增强"系统，不是"阻断"系统。如果 Hook 关键到需要阻断，应该用权限系统。

---

## 机制四：工具注册表 —— 插件化架构

### TL;DR
> 所有工具都继承 `BaseTool`，通过统一的 `ToolRegistry` 管理。这使得添加新工具就像插 U 盘一样简单。

### 工具基类设计

```python
class BaseTool(ABC):
    """所有工具的抽象基类。"""
    
    name: str
    description: str
    input_model: Type[BaseModel]  # Pydantic 模型
    
    @abstractmethod
    async def execute(
        self,
        arguments: BaseModel,
        context: ToolExecutionContext
    ) -> ToolResult:
        pass
    
    def to_schema(self) -> dict:
        """转换为 OpenAI 工具格式（供 LLM 调用）。"""
        return {
            "name": self.name,
            "description": self.description,
            "parameters": self.input_model.model_json_schema()
        }
```

### 统一返回格式

```python
@dataclass
class ToolResult:
    output: str           # 人类可读的输出（显示给用户）
    content: Any = None   # 机器可读的内容（给 LLM）
    is_error: bool = False
    error_message: str | None = None
```

**为什么分离 `output` 和 `content`？**

| 字段 | 用途 | 示例 |
|------|------|------|
| `output` | 显示给用户 | "找到 3 个匹配的文件" |
| `content` | 给 LLM 处理 | `["file1.py", "file2.py", "file3.py"]` |

> **设计洞察：** 用户想看简洁的总结，LLM 需要结构化的数据。一份结果，两种呈现。

### 43+ 工具分类

| 类别 | 工具数量 | 典型工具 |
|------|---------|---------|
| **文件 I/O** | 5 | `bash`, `read_file`, `write_file`, `edit_file`, `glob` |
| **搜索** | 4 | `grep`, `web_search`, `web_fetch`, `tool_search` |
| **MCP** | 3 | `mcp_tool`, `list_mcp_resources`, `read_mcp_resource` |
| **任务管理** | 6 | `task_create`, `task_get`, `task_list`, `task_update`, `task_stop`, `task_output` |
| **多代理** | 4 | `agent`, `send_message`, `team_create`, `team_delete` |
| **定时任务** | 4 | `cron_create`, `cron_list`, `cron_delete`, `cron_toggle` |
| **元工具** | 5 | `skill`, `config`, `brief`, `sleep`, `ask_user` |
| **其他** | 12 | `notebook_edit`, `lsp`, `enter_plan_mode`, `worktree`... |

### 添加自定义工具（5 步）

```python
# 1. 定义输入模型
class MyToolInput(BaseModel):
    query: str = Field(description="搜索查询")
    limit: int = Field(default=10, description="最大结果数")

# 2. 继承 BaseTool
class MyCustomTool(BaseTool):
    name = "my_custom_tool"
    description = "在我的数据库中搜索信息"
    input_model = MyToolInput
    
    async def execute(
        self,
        arguments: MyToolInput,
        context: ToolExecutionContext
    ) -> ToolResult:
        try:
            results = await search_database(arguments.query, limit=arguments.limit)
            return ToolResult(
                output=f"找到 {len(results)} 条结果",
                content=results,
                is_error=False
            )
        except Exception as e:
            return ToolResult(
                output=f"搜索失败：{str(e)}",
                is_error=True
            )

# 3. 注册工具
registry.register(MyCustomTool())
```

### 设计权衡：为什么用 Pydantic？

| 方案 | 优势 | 劣势 | OpenHarness 选择 |
|------|------|------|-----------------|
| **dict + 手动验证** | 灵活、无依赖 | 容易出错、代码冗长 | ❌ |
| **Pydantic** | 自动验证、类型提示、JSON Schema | 需要学习 | ✅ |
| **attrs** | 轻量 | 缺少 JSON Schema 生成 | ❌ |

> **决策理由：** LLM 需要工具的 JSON Schema 来理解如何调用。Pydantic 的 `model_json_schema()` 方法完美匹配这个需求。

---

## 机制五：技能系统 —— 按需加载的知识

### TL;DR
> 技能是**按需加载的 Markdown 文件**，不是始终占用的系统提示。这使得 OpenHarness 可以支持 40+ 技能而不浪费 Token。

### 技能 vs 系统提示词

| 特性 | 系统提示词 | 技能 (Skill) |
|------|-----------|-------------|
| 加载时机 | 会话开始 | 按需加载 |
| Token 消耗 | 始终占用 | 仅使用时占用 |
| 适用场景 | 核心身份定义 | 领域专业知识 |
| 文件格式 | 字符串 | Markdown (.md) |

**经济账：**

假设系统提示词 2000 tokens，40 个技能每个 1000 tokens：

| 方案 | Token 消耗 | 成本（每 1000 次对话） |
|------|-----------|---------------------|
| 全部注入 | 42,000 tokens | $0.42（Claude Sonnet） |
| 按需加载 | 2,000 + 平均 3,000 | $0.05 |
| **节省** | **-93%** | **88% 成本下降** |

### 技能文件格式

```markdown
---
name: commit
description: Create clean, well-structured git commits
author: OpenHarness Team
version: 1.0.0
---

# Commit Skill

## 何时使用
当用户要求创建 git 提交时使用此技能。

## 工作流程
1. 运行 `git diff --cached` 查看暂存区变更
2. 分析变更内容，识别修改的文件和功能
3. 按照约定式提交格式编写提交信息

## 提交信息规范
- 主题行不超过 50 字符
- 使用祈使语气（"Add" 而非 "Added"）
- 首字母小写
```

### 技能加载流程

```mermaid
flowchart TB
    A[启动时] --> B[加载内置技能]
    B --> C[加载用户技能<br/>~/.openharness/skills/]
    C --> D[加载插件技能<br/>plugins/*/skills/]
    D --> E[注册到 SkillRegistry]
    
    E --> F{LLM 请求技能}
    F -->|匹配 | G[加载技能内容]
    G --> H[注入到系统提示]
    H --> I[LLM 使用技能]
    
    F -->|不匹配 | J[不加载]
    
    style B fill:#e3f2fd
    style C fill:#e3f2fd
    style D fill:#e3f2fd
    style G fill:#fff9c4
    style H fill:#c8e6c9
```

### 技能加载代码

```python
def load_skills_from_dirs(
    directories: Iterable[str | Path],
    *,
    source: str = "user"
) -> list[SkillDefinition]:
    skills: list[SkillDefinition] = []
    
    for directory in directories:
        root = Path(directory).expanduser().resolve()
        
        for child in sorted(root.iterdir()):
            if child.is_dir():
                skill_path = child / "SKILL.md"
                if skill_path.exists():
                    content = skill_path.read_text(encoding="utf-8")
                    name, description = _parse_skill_markdown(child.name, content)
                    
                    skills.append(SkillDefinition(
                        name=name,
                        description=description,
                        content=content,
                        source=source,
                        path=str(skill_path)
                    ))
    
    return skills
```

### 设计权衡：为什么用 Markdown？

| 方案 | 优势 | 劣势 | OpenHarness 选择 |
|------|------|------|-----------------|
| **纯文本** | 最简单 | 缺少结构、难以解析 | ❌ |
| **JSON/YAML** | 结构化 | 对人类不友好 | ❌ |
| **Markdown** | 人类可读、LLM 友好、支持结构 | 需要解析 | ✅ |

> **洞察：** 技能文件既给 LLM 看，也给人看。Markdown 是唯一同时满足两者的格式。

---

## 架构决策记录（ADR）

### ADR-001：为什么选择 Textual 而非 Rich？

**背景：** OpenHarness 需要一个交互式终端界面。

**候选方案：**

| 维度 | Rich | Textual（选择） |
|------|------|----------------|
| 交互性 | 有限 | 完整应用框架 |
| 组件系统 | 无 | 有（按钮、输入框等） |
| 事件驱动 | 无 | 有 |
| 学习曲线 | 低 | 中 |

**决策理由：**

> OpenHarness 需要**命令选择器**、**权限确认对话框**、**模式切换器**等交互组件。Rich 只能做静态输出，Textual 是完整的应用框架。

**后果：**

- ✅ 正面：支持复杂的交互 UI
- ⚠️ 负面：学习曲线较陡

---

### ADR-002：为什么工具返回 ToolResult 而非原始数据？

**背景：** 工具执行函数的返回值格式需要统一。

**候选方案：**

| 方案 | 优势 | 劣势 |
|------|------|------|
| 原始数据（dict/str） | 简单 | 错误处理不统一 |
| ToolResult 数据类 | 统一错误处理、分离展示与内容 | 需要包装返回值 |

**决策理由：**

```python
@dataclass
class ToolResult:
    output: str           # 人类可读
    content: Any = None   # 机器可读
    is_error: bool = False
    error_message: str | None = None
```

> 所有错误都格式化为 `is_error=True`，TUI 和 API 可以统一处理。

---

### ADR-003：为什么 Hook 失败不阻断主流程？

**背景：** Hook 执行可能失败（超时、网络错误、代码 bug）。

**候选方案：**

| 方案 | 优势 | 劣势 |
|------|------|------|
| 阻断主流程 | Hook 关键性强 | 单点故障风险 |
| 不阻断（选择） | 容错性强 | Hook 可能失效 |

**决策理由：**

> Hook 是"增强"系统，不是"阻断"系统。如果 Hook 关键到需要阻断，应该用权限系统。

---

## 性能与优化

### 关键性能指标

| 操作类型 | P50 | P95 | P99 | 优化建议 |
|---------|-----|-----|-----|---------|
| 文件读取 | 20ms | 100ms | 500ms | 使用异步 I/O |
| Bash 命令 | 100ms | 500ms | 2000ms | 设置合理 timeout |
| Web 搜索 | 800ms | 2000ms | 5000ms | 并行多引擎搜索 |
| MCP 调用 | 500ms | 1500ms | 3000ms | 启用自动重连 |
| 技能加载 | 50ms | 200ms | 500ms | 已足够快 |

### 内存使用分析

```
典型负载下的内存分布：

Python 运行时      ████████████████████████████  35%
Textual TUI       ████████████████              20%
LLM SDK 缓存      ██████████████████████        25%
工具依赖库        ████████                      10%
对话历史缓存      ████████                      10%
```

**优化建议：**

1. **启用自动压缩** - `auto_compact_threshold_tokens` 减少历史缓存
2. **限制并发工具** - 默认并发数避免内存爆炸
3. **使用流式处理** - 避免完整响应缓冲

---

## 扩展点地图

```
扩展复杂度从低到高：

Level 1: 添加新技能     (~50 行 Markdown)   🟢 低风险
         ↓
Level 2: 添加 Hook      (~20 行 YAML)       🟢 低风险
         ↓
Level 3: 添加新工具     (~150 行 Python)    🟡 中风险
         ↓
Level 4: 添加插件       (~200 行)           🟡 中风险
         ↓
Level 5: 添加供应商     (~300 行)           🟠 高风险
```

---

## 总结：OpenHarness 的本质

> **OpenHarness 不是"Claude Code 的克隆"**，而是一个**可审计、可扩展、可定制的 AI 代理基础设施**。

它的核心价值在于证明了：

1. **生产级 AI 代理的架构可以被开源复现**
2. **通过 Hook 系统、增强的权限控制和多供应商支持实现差异化创新**
3. **模型提供智能，代码提供边界**

正如项目标语：

> **"The model is the agent. The code is the harness."**

OpenHarness 要做的是那根既结实又灵活的缰绳，让每一匹 AI 骏马都能成为用户的战马。

---

## 附录：与 Hermes-Agent 对比

| 特性 | OpenHarness | Hermes-Agent |
|------|-------------|--------------|
| **定位** | Claude Code 开源实现 | 自进化 AI 代理 |
| **核心创新** | Harness 模式、Hook 系统 | 技能自生成、记忆闭环 |
| **工具数量** | 43+ | 40+ |
| **权限控制** | 5 级（模式 + 路径 + 命令） | 3 级（审批 + 黑名单） |
| **扩展系统** | Hook + 技能 + 插件 | 技能 + 记忆提供者 |
| **多平台** | CLI + TUI + ohmo 网关 | CLI + 5 个消息平台 |
| **测试覆盖** | 114 单元 + 6 套 E2E | ~3000 测试 |

**选择建议：**

- 需要 **Claude Code 兼容** 和 **Hook 系统** → OpenHarness
- 需要 **多消息平台** 和 **记忆闭环** → Hermes-Agent

---

*© 2026 OpenHarness 技术文档 | 基于 MIT 协议开源*
