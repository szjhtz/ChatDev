# ChatDev 2.0 (DevAll) - Code Wiki

## 1. 项目概述

### 1.1 项目简介
ChatDev 2.0 (DevAll) 是一个零代码多智能体平台，用于"开发一切"。它允许用户通过简单的配置快速构建和执行自定义的多智能体系统，无需编写代码。用户可以定义智能体、工作流和任务，以编排复杂场景，如数据可视化、3D 生成和深度研究。

### 1.2 项目历史与版本
- **2026年1月7日**: ChatDev 2.0 (DevAll) 正式发布
- **经典 ChatDev 1.0 (Legacy)**: 已移至 `chatdev1.0` 分支维护

### 1.3 核心特性
- 零代码多智能体编排
- 可视化工作流设计
- Python SDK 支持
- Web UI 控制台
- 多种执行策略（DAG、循环、多数投票）
- 可扩展的节点、提供程序和工具

## 2. 项目架构

### 2.1 整体架构图
```
DevAll Platform
├── Frontend (Vue 3 + Vite)
│   ├── Workflow Canvas (VueFlow)
│   ├── Web Console
│   └── API Integration
├── Backend (FastAPI)
│   ├── API Routes
│   ├── Services Layer
│   └── WebSocket Server
├── Runtime Engine
│   ├── Node Executors
│   ├── Edge Conditions/Processors
│   ├── Memory Management
│   └── Tool/Function Management
└── Configuration System
    ├── YAML Workflow Definitions
    ├── Schema Registry
    └── Config Loader
```

### 2.2 目录结构详解

```
/workspace/
├── .agents/              # 技能定义和示例
├── assets/               # 项目资源和演示图片
├── check/                # YAML 配置验证工具
├── docs/                 # 用户文档（中英文）
├── entity/               # 核心数据模型和配置
│   ├── configs/         # 配置模型（节点、边、图）
│   ├── graph_config.py  # 图配置类
│   ├── messages.py      # 消息模型
│   └── enums.py         # 枚举定义
├── frontend/             # Vue 3 前端应用
│   ├── src/
│   │   ├── components/  # Vue 组件
│   │   ├── pages/       # 页面组件
│   │   └── utils/       # 工具函数
│   └── package.json
├── functions/            # 自定义 Python 工具函数
│   ├── edge/            # 边条件函数
│   ├── edge_processor/  # 边处理器函数
│   └── function_calling/ # 函数调用工具
├── mcp_example/          # MCP 服务器示例
├── runtime/              # 运行时引擎
│   ├── bootstrap/       # 启动引导
│   ├── edge/            # 边条件和处理器
│   ├── node/            # 节点执行器
│   │   ├── agent/       # Agent 节点功能
│   │   │   ├── memory/  # 内存管理
│   │   │   ├── providers/ # LLM 提供程序
│   │   │   ├── skills/  # 技能管理
│   │   │   ├── thinking/ # 思考模块
│   │   │   └── tool/    # 工具管理
│   │   └── executor/    # 节点执行器
│   └── sdk.py           # Python SDK
├── schema_registry/      # 配置模式注册表
├── server/               # FastAPI 后端
│   ├── routes/          # API 路由
│   ├── services/        # 业务服务
│   └── app.py           # FastAPI 应用
├── tests/                # 测试文件
├── tools/                # 开发工具
├── utils/                # 工具函数库
├── workflow/             # 工作流引擎
│   ├── executor/        # 执行器
│   ├── hooks/           # 钩子
│   ├── runtime/         # 运行时上下文
│   └── graph.py         # 图执行器
├── yaml_instance/        # 工作流 YAML 示例
├── yaml_template/        # 工作流 YAML 模板
├── Dockerfile            # Docker 配置
├── Makefile              # 构建脚本
├── README.md             # 项目说明
├── pyproject.toml        # Python 项目配置
├── requirements.txt      # Python 依赖
└── run.py                # CLI 入口
```

## 3. 核心模块详解

### 3.1 Entity 模块 - 配置与数据模型

**位置**: `/workspace/entity/`

**职责**: 定义核心数据结构、配置模型和枚举。

#### 3.1.1 BaseConfig - 配置基类
**文件**: [entity/configs/base.py](file:///workspace/entity/configs/base.py)

`BaseConfig` 是所有配置类的基类，提供：
- 验证机制 (`validate()`)
- 模式导出 (`collect_schema()`)
- 字段规范 (`FIELD_SPECS`)
- 约束条件 (`CONSTRAINTS`)
- 子配置路由 (`CHILD_ROUTES`)

关键方法：
```python
@dataclass
class BaseConfig:
    path: str
    FIELD_SPECS: ClassVar[Dict[str, ConfigFieldSpec]]
    CONSTRAINTS: ClassVar[Sequence[RuntimeConstraint]]
    
    def validate(self) -> None
    @classmethod
    def field_specs(cls) -> Dict[str, ConfigFieldSpec]
    @classmethod
    def collect_schema(cls) -> SchemaNode
```

#### 3.1.2 GraphConfig - 图配置
**文件**: [entity/graph_config.py](file:///workspace/entity/graph_config.py)

`GraphConfig` 包装了解析后的图定义和运行时元数据。

```python
@dataclass
class GraphConfig:
    definition: GraphDefinition
    name: str
    output_root: Path
    log_level: LogLevel
    metadata: Dict[str, Any]
    source_path: Optional[str]
    vars: Dict[str, Any]
    
    @classmethod
    def from_definition(cls, definition: GraphDefinition, ...) -> "GraphConfig"
    def get_node_definitions(self) -> List[Node]
    def get_edge_definitions(self) -> List[EdgeConfig]
```

#### 3.1.3 配置子模块
- **node/**: 节点配置（Agent、Human、Python、Skills 等）
- **edge/**: 边配置（条件、处理器、动态边）

### 3.2 Runtime 模块 - 运行时引擎

**位置**: `/workspace/runtime/`

**职责**: 提供节点执行、内存管理、工具调用等核心功能。

#### 3.2.1 Python SDK
**文件**: [runtime/sdk.py](file:///workspace/runtime/sdk.py)

提供从 Python 代码执行工作流的便捷接口。

关键类：
```python
@dataclass
class WorkflowMetaInfo:
    session_name: str
    yaml_file: str
    log_id: Optional[str]
    outputs: Optional[Dict[str, Any]]
    token_usage: Optional[Dict[str, Any]]
    output_dir: Path

@dataclass
class WorkflowRunResult:
    final_message: Optional[Message]
    meta_info: WorkflowMetaInfo

def run_workflow(
    yaml_file: Union[str, Path],
    *,
    task_prompt: str,
    attachments: Optional[Sequence[Union[str, Path]]] = None,
    session_name: Optional[str] = None,
    ...
) -> WorkflowRunResult
```

使用示例：
```python
from runtime.sdk import run_workflow

result = run_workflow(
    yaml_file="yaml_instance/demo.yaml",
    task_prompt="Summarize the attached document.",
    attachments=["/path/to/document.pdf"]
)

if result.final_message:
    print(f"Output: {result.final_message.text_content()}")
```

#### 3.2.2 节点执行器
**文件**: [runtime/node/executor/factory.py](file:///workspace/runtime/node/executor/factory.py)

`NodeExecutorFactory` 为不同类型的节点创建执行器。

```python
class NodeExecutorFactory:
    @staticmethod
    def create_executors(context: ExecutionContext, subgraphs: dict = None) -> Dict[str, NodeExecutor]
    @staticmethod
    def create_executor(node_type: str, context: ExecutionContext, ...) -> NodeExecutor
```

支持的节点类型：
- `agent`: AI 智能体节点
- `human`: 人工交互节点
- `python`: Python 代码执行节点
- `subgraph`: 子图节点
- `literal`: 字面量节点
- `passthrough`: 直通节点
- `loop_counter`: 循环计数器
- `loop_timer`: 循环计时器

#### 3.2.3 Agent 节点功能
- **Memory**: 内存管理（简单内存、嵌入内存、Mem0 内存等）
- **Providers**: LLM 提供程序（OpenAI、Gemini 等）
- **Skills**: 技能管理
- **Thinking**: 思考模块（自我反思等）
- **Tool**: 工具调用管理

### 3.3 Workflow 模块 - 工作流引擎

**位置**: `/workspace/workflow/`

**职责**: 管理图的执行、拓扑构建、执行策略等。

#### 3.3.1 GraphExecutor - 图执行器
**文件**: [workflow/graph.py](file:///workspace/workflow/graph.py)

`GraphExecutor` 是核心执行引擎，负责：
- 初始化内存和思考管理器
- 构建节点执行器
- 管理执行流程
- 处理边条件和处理器
- 收集结果

关键方法：
```python
class GraphExecutor:
    @classmethod
    def execute_graph(cls, graph: GraphContext, task_prompt: Any, ...) -> "GraphExecutor"
    
    def run(self, task_prompt: Any) -> Dict[str, Any]
    def _execute_node(self, node: Node) -> None
    def _process_result(self, node: Node, input_payload: List[Message]) -> List[Message]
    def get_final_output_message(self) -> Message | None
```

#### 3.3.2 执行策略
支持三种执行策略：
1. **DAG 执行策略** (`DagExecutionStrategy`): 用于无环图
2. **循环执行策略** (`CycleExecutionStrategy`): 用于有环图
3. **多数投票策略** (`MajorityVoteStrategy`): 用于并行投票

#### 3.3.3 动态边执行
支持动态配置的边，可进行并行/树状执行：
- `DynamicEdgeExecutor` 处理动态配置
- 支持输入分割和结果聚合

### 3.4 Server 模块 - 后端 API

**位置**: `/workspace/server/`

**职责**: 提供 REST API 和 WebSocket 服务。

#### 3.4.1 FastAPI 应用
**文件**: [server/app.py](file:///workspace/server/app.py)

```python
from fastapi import FastAPI
from server.bootstrap import init_app

app = FastAPI(title="DevAll Workflow Server", version="1.0.0")
init_app(app)
```

#### 3.4.2 API 路由
- `/health`: 健康检查
- `/workflows`: 工作流管理
- `/execute`: 执行工作流
- `/sessions`: 会话管理
- `/websocket`: WebSocket 连接
- `/tools`: 工具管理
- `/uploads`: 文件上传

#### 3.4.3 服务层
- `WorkflowRunService`: 工作流运行服务
- `SessionExecution`: 会话执行
- `WebSocketExecutor`: WebSocket 执行器
- `ArtifactDispatcher`: 制品分发

### 3.5 Frontend 模块 - Web 控制台

**位置**: `/workspace/frontend/`

**职责**: 提供可视化工作流设计和执行界面。

#### 3.5.1 技术栈
- **框架**: Vue 3 + Vite
- **工作流画布**: VueFlow
- **路由**: Vue Router
- **国际化**: Vue I18n
- **Markdown**: markdown-it

#### 3.5.2 核心组件
- `WorkflowView.vue`: 工作流设计视图
- `WorkflowWorkbench.vue`: 工作流工作台
- `LaunchView.vue`: 工作流启动视图
- `WorkflowNode.vue`: 节点组件
- `WorkflowEdge.vue`: 边组件

#### 3.5.3 页面
- `HomeView`: 首页
- `WorkflowList`: 工作流列表
- `TutorialView`: 教程
- `BatchRunView`: 批量运行

## 4. 核心类与函数

### 4.1 配置系统核心类

| 类名 | 文件路径 | 职责 |
|------|----------|------|
| `BaseConfig` | [entity/configs/base.py](file:///workspace/entity/configs/base.py) | 所有配置类的基类 |
| `GraphDefinition` | entity/configs/graph.py | 图定义配置 |
| `Node` | entity/configs/node/node.py | 节点配置 |
| `EdgeConfig` | entity/configs/edge/edge.py | 边配置 |
| `GraphConfig` | [entity/graph_config.py](file:///workspace/entity/graph_config.py) | 图配置包装类 |
| `Message` | entity/messages.py | 消息数据模型 |

### 4.2 运行时核心类

| 类名 | 文件路径 | 职责 |
|------|----------|------|
| `GraphExecutor` | [workflow/graph.py](file:///workspace/workflow/graph.py) | 图执行器核心 |
| `NodeExecutor` | runtime/node/executor/base.py | 节点执行器基类 |
| `NodeExecutorFactory` | [runtime/node/executor/factory.py](file:///workspace/runtime/node/executor/factory.py) | 节点执行器工厂 |
| `MemoryBase` | runtime/node/agent/memory/memory_base.py | 内存基类 |
| `MemoryManager` | runtime/node/agent/memory/__init__.py | 内存管理器 |
| `ThinkingManagerBase` | runtime/node/agent/thinking/registry.py | 思考管理器基类 |

### 4.3 核心函数

| 函数名 | 文件路径 | 功能 |
|--------|----------|------|
| `run_workflow()` | [runtime/sdk.py](file:///workspace/runtime/sdk.py) | 从 Python 执行工作流 |
| `load_config()` | check/check.py | 加载并验证 YAML 配置 |
| `ensure_schema_registry_populated()` | runtime/bootstrap/schema.py | 初始化模式注册表 |
| `init_app()` | server/bootstrap.py | 初始化 FastAPI 应用 |

## 5. 依赖关系

### 5.1 Python 依赖 (pyproject.toml)

**核心依赖**:
- `pyyaml`: YAML 解析
- `openai`: OpenAI API
- `fastapi==0.124.0`: Web 框架
- `pydantic==2.12.5`: 数据验证
- `uvicorn`: ASGI 服务器
- `websockets`: WebSocket 支持
- `mcp` / `fastmcp`: MCP 协议

**AI/ML 相关**:
- `faiss-cpu`: 向量数据库
- `google-genai>=1.52.0`: Google AI
- `mem0ai>=1.0.9`: Mem0 内存

**数据处理**:
- `pandas>=2.3.3`: 数据分析
- `numpy>=2.3.5`: 数值计算
- `matplotlib`: 绘图
- `seaborn>=0.13.2`: 统计绘图
- `cartopy`: 地图绘制
- `openpyxl>=3.1.2`: Excel 处理

**工具库**:
- `requests`: HTTP 请求
- `beautifulsoup4`: HTML 解析
- `ddgs`: DuckDuckGo 搜索
- `click>=8.1.8,<8.3`: CLI
- `tenacity`: 重试
- `filelock>=3.20.1`: 文件锁
- `markdown>=3.10`: Markdown
- `xhtml2pdf>=0.2.17`: PDF 生成
- `pygame>=2.6.1`: 游戏开发
- `chardet>=5.2.0`: 字符编码检测

**测试**:
- `pytest`: 测试框架

### 5.2 JavaScript 依赖 (frontend/package.json)

**核心依赖**:
- `vue@^3.5.22`: Vue 3 框架
- `@vue-flow/core@^1.47.0`: 工作流画布
- `@vue-flow/background@^1.3.2`: 背景网格
- `@vue-flow/controls@^1.1.3`: 控制组件
- `@vue-flow/minimap@^1.5.4`: 小地图
- `vue-router@^4.6.0`: 路由
- `vue-i18n@^11.3.0`: 国际化
- `js-yaml@^4.1.0`: YAML 解析
- `markdown-it@^14.1.0`: Markdown
- `markdown-it-anchor@^9.2.0`: Markdown 锚点

**开发依赖**:
- `vite@^7.1.7`: 构建工具
- `@vitejs/plugin-vue@^6.0.1`: Vue 插件
- `eslint@^9.39.1`: 代码检查
- `eslint-plugin-vue@^10.5.1`: Vue ESLint
- `cross-env@^10.1.0`: 环境变量

### 5.3 模块依赖关系图

```
server/ (FastAPI)
  ↓ uses
workflow/ (工作流引擎)
  ↓ uses
runtime/ (运行时)
  ↓ uses
entity/ (配置模型)
  ↓ uses
utils/ (工具函数)

frontend/ (Vue)
  ↓ API calls
server/ (FastAPI)
```

## 6. 项目运行方式

### 6.1 环境要求

- **操作系统**: macOS / Linux / WSL / Windows
- **Python**: 3.12+
- **Node.js**: 18+
- **包管理器**: [uv](https://docs.astral.sh/uv/)

### 6.2 安装步骤

#### 6.2.1 后端依赖
```bash
uv sync
```

#### 6.2.2 前端依赖
```bash
cd frontend && npm install
```

#### 6.2.3 环境配置
```bash
cp .env.example .env
# 编辑 .env 文件，设置 API_KEY 和 BASE_URL
```

### 6.3 运行应用

#### 6.3.1 使用 Makefile（推荐）
```bash
# 同时启动后端和前端
make dev

# 访问 Web 控制台: http://localhost:5173
```

#### 6.3.2 手动命令

**启动后端**:
```bash
uv run python server_main.py --port 6400 --reload
```

**启动前端**:
```bash
cd frontend
VITE_API_BASE_URL=http://localhost:6400 npm run dev
```

#### 6.3.3 使用 Docker
```bash
# 构建并运行
docker compose up --build

# 访问
# 后端: http://localhost:6400
# 前端: http://localhost:5173
```

### 6.4 实用命令

```bash
# 显示帮助
make help

# 同步 YAML 工作流到前端
make sync

# 验证所有 YAML 工作流
make validate-yamls

# 运行后端测试
make backend-tests

# 停止服务
make stop
```

### 6.5 CLI 使用

```bash
# 运行工作流
python run.py --path yaml_instance/demo.yaml --name test_project

# 检查模式
python run.py --inspect-schema
```

## 7. 工作流配置

### 7.1 YAML 结构

工作流配置使用 YAML 文件，基本结构：

```yaml
graph:
  nodes:
    - id: node1
      type: agent
      config:
        provider: openai
        model: gpt-4
        system_prompt: "You are a helpful assistant."
  edges:
    - source: node1
      target: node2
      condition: "true"
  memory:
    - name: global_memory
      type: simple
  initial_instruction: ""
  organization: "DefaultOrg"
vars:
  API_KEY: "${API_KEY}"
```

### 7.2 节点类型

1. **Agent**: AI 智能体节点
2. **Human**: 人工交互节点
3. **Python**: Python 代码执行
4. **Subgraph**: 子图引用
5. **Literal**: 静态内容
6. **Passthrough**: 消息传递
7. **LoopCounter**: 循环计数
8. **LoopTimer**: 循环计时

### 7.3 示例工作流

项目在 `yaml_instance/` 目录下提供了丰富的示例：

- `demo_*.yaml`: 功能演示
- `ChatDev_v1.yaml`: ChatDev 1.0 复刻
- `GameDev_with_manager.yaml`: 游戏开发
- `deep_research_v1.yaml`: 深度研究
- `data_visualization_basic.yaml`: 数据可视化

## 8. 扩展开发

### 8.1 添加自定义节点

1. 在 `runtime/node/executor/` 中创建执行器
2. 在 `entity/configs/node/` 中创建配置类
3. 在 `runtime/node/registry.py` 中注册

### 8.2 添加自定义工具

1. 在 `functions/function_calling/` 中添加 Python 函数
2. 在节点配置中引用

### 8.3 添加自定义内存存储

1. 继承 `MemoryBase`
2. 实现 `load()`, `save()`, `get()`, `set()` 方法
3. 在 `runtime/node/agent/memory/registry.py` 中注册

## 9. 参考文档

- [用户指南 (英文)](docs/user_guide/en/index.md)
- [用户指南 (中文)](docs/user_guide/zh/index.md)
- [工作流编写](docs/user_guide/en/workflow_authoring.md)
- [内存模块](docs/user_guide/en/modules/memory.md)
- [工具模块](docs/user_guide/en/modules/tooling/index.md)

## 10. 贡献指南

欢迎社区贡献！可以通过以下方式：
- 提交 Issue 报告问题
- 提交 Pull Request 修复 bug 或添加新功能
- 分享高质量的工作流模板

## 11. 许可证

Apache-2.0 License
