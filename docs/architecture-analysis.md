# OpenManus 架构分析

本文档详细分析 OpenManus 项目的主流程架构、Agent 系统和 Planning 机制。

## 目录

- [执行模式概述](#执行模式概述)
- [主流程详解 (main.py)](#主流程详解-mainpy)
- [规划流程详解 (run_flow.py)](#规划流程详解-run_flowpy)
- [Agent 系统](#agent-系统)
- [工具系统](#工具系统)
- [关键代码位置索引](#关键代码位置索引)

---

## 执行模式概述

OpenManus 提供两种执行模式：

| 模式 | 入口文件 | 任务分析 | 任务规划 | 执行方式 | 适用场景 |
|------|----------|----------|----------|----------|----------|
| **直接执行** | `main.py` | 无 | 无 | LLM 逐步临时决策 | 简单任务 |
| **规划执行** | `run_flow.py` | 有 | 有 | 按预定计划执行 | 复杂多步骤任务 |

---

## 主流程详解 (main.py)

### 特点

- **没有**预先分析任务需求
- **没有**生成执行计划
- 每一步都是 LLM 根据当前上下文"临时思考"决定下一步

### 执行流程

```
用户输入 → Agent.run() → 循环执行 step()
                              │
                              ├─ think(): LLM 临时决定用什么工具
                              └─ act(): 执行工具
```

### 详细步骤

```
1. main.py
   └─ Manus.create() → 创建 Agent 实例
   └─ agent.run(prompt) → 开始执行

2. BaseAgent.run(request)
   └─ 将用户请求加入 memory
   └─ while current_step < max_steps:
       └─ step() → 单步执行

3. ToolCallAgent.step() [继承自 ReActAgent]
   ├─ think()  ← 思考阶段
   │   └─ LLM.ask_tool(messages, tools)
   │       └─ LLM 返回: 文本响应 + tool_calls 列表
   │
   └─ act()    ← 行动阶段
       └─ 遍历 tool_calls，执行每个工具
       └─ 将结果加入 memory

4. 循环直到:
   - LLM 调用 terminate 工具
   - 达到 max_steps 限制
   - 检测到 stuck 状态
```

---

## 规划流程详解 (run_flow.py)

### 特点

- 先用 LLM + PlanningTool 生成完整计划
- 计划包含多个步骤和状态管理
- 按计划顺序执行
- **调用的是 Agent，不是直接调用 tool_call**

### 执行流程

```
用户输入
    │
    ▼
PlanningFlow.execute()
    │
    ├─ 1. _create_initial_plan()
    │      └─ LLM + PlanningTool 生成计划
    │         例如: ["[MANUS] 搜索资料", "[DATA_ANALYSIS] 分析数据"]
    │
    ├─ 2. 循环执行每个步骤:
    │      ├─ _get_current_step_info() → 获取步骤，提取 [AGENT_TYPE]
    │      ├─ get_executor(step_type) → 根据类型选择 Agent
    │      └─ _execute_step(executor, step_info)
    │             └─ executor.run(step_prompt)  ← Agent 执行
    │                    │
    │                    └─ Agent 内部: think() → act() → tool_call
    │
    └─ 3. _finalize_plan() → 生成总结
```

### 执行层次

```
PlanningFlow
    │
    └─ agent.run(step_prompt)    ← Flow 调用 Agent
           │
           └─ agent 内部使用 tool_call  ← Agent 内部调用工具
```

### Agent 创建与注册

Agent 在 `run_flow.py` 中手动创建并传入：

```python
# run_flow.py:12-16
agents = {
    "manus": Manus(),
}
if config.run_flow_config.use_data_analysis_agent:
    agents["data_analysis"] = DataAnalysis()

# 创建 PlanningFlow
flow = FlowFactory.create_flow(flow_type=FlowType.PLANNING, agents=agents)
```

### Agent 选择机制

`PlanningFlow` 通过步骤文本中的标记来选择 Agent：

```python
# app/flow/planning.py:77-92
def get_executor(self, step_type: Optional[str] = None) -> BaseAgent:
    # 如果步骤类型匹配某个 agent key，使用那个 agent
    if step_type and step_type in self.agents:
        return self.agents[step_type]
    # 否则使用第一个可用的 executor
    for key in self.executor_keys:
        if key in self.agents:
            return self.agents[key]
    # 回退到 primary agent
    return self.primary_agent
```

步骤文本示例：
```
"[MANUS] 搜索相关文档"       → 使用 Manus agent
"[DATA_ANALYSIS] 分析数据"   → 使用 DataAnalysis agent
```

从步骤文本中提取标记（`planning.py:244-247`）：
```python
type_match = re.search(r"\[([A-Z_]+)\]", step)
if type_match:
    step_info["type"] = type_match.group(1).lower()
```

### 步骤状态管理

```python
class PlanStepStatus(str, Enum):
    NOT_STARTED = "not_started"   # 未开始
    IN_PROGRESS = "in_progress"   # 执行中
    COMPLETED = "completed"       # 已完成
    BLOCKED = "blocked"           # 被阻塞
```

---

## Agent 系统

### Agent 继承层次

```
BaseAgent（基础类）
  └─ ReActAgent（思考-行动反应模式）
      └─ ToolCallAgent（工具调用能力）
          ├─ Manus（通用多功能 Agent）
          ├─ DataAnalysis（数据分析 Agent）
          ├─ BrowserAgent（浏览器自动化 Agent）
          ├─ SWEAgent（软件工程 Agent）
          ├─ MCPAgent（MCP 协议 Agent）
          └─ SandboxAgent（沙箱环境 Agent）
```

### 可用 Agent 列表

| Agent | 文件 | 描述 | 可用工具 |
|-------|------|------|----------|
| **Manus** | `app/agent/manus.py` | 通用多功能 Agent | PythonExecute, BrowserUseTool, StrReplaceEditor, AskHuman, Terminate + MCP 工具 |
| **DataAnalysis** | `app/agent/data_analysis.py` | 数据分析 Agent | NormalPythonExecute, VisualizationPrepare, DataVisualization, Terminate |
| **BrowserAgent** | `app/agent/browser.py` | 浏览器自动化 Agent | BrowserUseTool, Terminate |
| **SWEAgent** | `app/agent/swe.py` | 软件工程 Agent | Bash, StrReplaceEditor, Terminate |
| **MCPAgent** | `app/agent/mcp.py` | MCP 协议 Agent | 动态加载 MCP 工具 |
| **SandboxAgent** | `app/agent/sandbox_agent.py` | 沙箱环境 Agent | 沙箱工具集 |

### 核心类详解

#### BaseAgent (`app/agent/base.py`)

核心属性：
- `name`, `description` - Agent 标识
- `system_prompt`, `next_step_prompt` - 提示词
- `llm` - 语言模型实例
- `memory` - 对话记忆
- `state` - AgentState: IDLE/RUNNING/FINISHED/ERROR
- `max_steps`, `current_step` - 执行步数控制

核心方法：
- `async run(request)` - 主执行循环
- `async step()` - 单步执行（由子类实现）
- `update_memory(role, content)` - 更新对话历史
- `is_stuck()` - 检测重复应答

#### ReActAgent (`app/agent/react.py`)

实现 ReAct（Reasoning + Acting）模式：
- `async think()` - 思考阶段（由子类实现）
- `async act()` - 行动阶段（由子类实现）
- `async step()` - step = think + act

#### ToolCallAgent (`app/agent/toolcall.py`)

关键属性：
```python
tool_choices: ToolChoice = AUTO  # auto/none/required
available_tools: ToolCollection  # 工具集合
special_tool_names: List[str]    # 特殊工具（如 terminate）
```

核心功能：
- 通过 LLM 进行工具调用
- 管理 `available_tools`（ToolCollection）
- 实现 `think()` 方法：调用 LLM 获取工具建议
- 实现 `act()` 方法：执行推荐的工具调用

### 当前 run_flow.py 的局限性

目前 `run_flow.py` **只注册了 2 个 Agent**:
- `manus` (默认)
- `data_analysis` (可选，需配置开启)

其他 Agent (BrowserAgent, SWEAgent 等) 虽然存在，但**没有在 run_flow.py 中使用**。如果需要使用它们，需要手动添加到 `agents` 字典中。

---

## 工具系统

### 工具架构

```
BaseTool（基类）
  └─ 具体工具实现
     ├─ PythonExecute - Python 代码执行
     ├─ BrowserUseTool - 浏览器自动化
     ├─ StrReplaceEditor - 文件编辑
     ├─ AskHuman - 人工询问
     ├─ Terminate - 任务终止
     ├─ FileOperators - 文件操作
     ├─ WebSearch - 网络搜索
     ├─ PlanningTool - 计划管理
     ├─ MCPClientTool - MCP 工具包装
     ├─ Bash - Shell 命令执行
     └─ DataVisualization - 数据可视化
```

### BaseTool 接口 (`app/tool/base.py`)

```python
class BaseTool(ABC, BaseModel):
    name: str                    # 工具名称
    description: str             # 工具描述
    parameters: dict             # JSON schema 格式的参数定义

    async def execute(self, **kwargs) -> ToolResult:
        """实际执行的方法"""
        ...

    def to_param(self) -> Dict:
        """转换为 OpenAI 兼容的 function call 格式"""
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": self.parameters,
            }
        }
```

### ToolCollection (`app/tool/tool_collection.py`)

```python
class ToolCollection:
    def __init__(self, *tools: BaseTool):
        self.tools = tools
        self.tool_map = {tool.name: tool}

    def to_params(self) -> List[Dict]:
        """生成所有工具的 schema"""
        return [tool.to_param() for tool in self.tools]

    async def execute(self, *, name: str, tool_input: Dict) -> ToolResult:
        """执行指定工具"""
        tool = self.tool_map.get(name)
        return await tool(**tool_input)

    def add_tools(self, *tools: BaseTool):
        """动态添加工具（如 MCP 工具）"""
        ...
```

---

## 关键代码位置索引

| 组件 | 文件路径 | 说明 |
|------|----------|------|
| 直接执行入口 | `main.py` | 无规划模式 |
| 规划执行入口 | `run_flow.py` | 有规划模式 |
| MCP 执行入口 | `run_mcp.py` | MCP 模式 |
| Agent 基类 | `app/agent/base.py` | run() 主循环 |
| ReAct Agent | `app/agent/react.py` | think/act 模式 |
| 工具调用 Agent | `app/agent/toolcall.py` | think()/act() 实现 |
| Manus Agent | `app/agent/manus.py` | 通用 Agent |
| 数据分析 Agent | `app/agent/data_analysis.py` | 数据分析专用 |
| 浏览器 Agent | `app/agent/browser.py` | 浏览器自动化 |
| SWE Agent | `app/agent/swe.py` | 软件工程 |
| 规划流程 | `app/flow/planning.py` | PlanningFlow 类 |
| 流程工厂 | `app/flow/flow_factory.py` | FlowFactory |
| 规划工具 | `app/tool/planning.py` | PlanningTool |
| 工具基类 | `app/tool/base.py` | BaseTool |
| 工具集合 | `app/tool/tool_collection.py` | ToolCollection |
| LLM 接口 | `app/llm.py` | LLM 调用封装 |
| 数据结构 | `app/schema.py` | Message, Memory 等 |
| 配置管理 | `app/config.py` | 配置加载 |

---

## 目录结构概览

```
OpenManus/
├── main.py                    # 单 Agent 入口
├── run_flow.py               # 多 Agent 规划入口
├── run_mcp.py                # MCP 模式入口
│
├── app/
│   ├── agent/
│   │   ├── base.py          # BaseAgent 基础类
│   │   ├── react.py         # ReActAgent 思考-行动模式
│   │   ├── toolcall.py      # ToolCallAgent 工具调用
│   │   ├── manus.py         # Manus 通用 Agent
│   │   ├── data_analysis.py # 数据分析 Agent
│   │   ├── browser.py       # 浏览器 Agent
│   │   ├── swe.py           # 软件工程 Agent
│   │   └── mcp.py           # MCP Protocol Agent
│   │
│   ├── flow/
│   │   ├── base.py          # BaseFlow 基类
│   │   ├── planning.py      # 规划流程实现
│   │   └── flow_factory.py  # 流程工厂
│   │
│   ├── tool/
│   │   ├── base.py          # BaseTool 基类
│   │   ├── tool_collection.py  # 工具集合
│   │   ├── planning.py      # 计划工具
│   │   ├── python_execute.py   # Python 执行
│   │   ├── file_operators.py   # 文件操作
│   │   ├── browser_use_tool.py # 浏览器自动化
│   │   ├── str_replace_editor.py # 文件编辑
│   │   └── ...
│   │
│   ├── schema.py            # 数据结构定义
│   ├── llm.py              # LLM 接口层
│   ├── config.py           # 配置管理
│   └── prompt/              # 提示词模板
│
├── config/
│   └── config.toml         # 配置文件
│
└── docs/
    └── architecture-analysis.md  # 本文档
```
