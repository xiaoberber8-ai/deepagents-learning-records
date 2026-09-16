# Deep Agents 第 2 章（上）：第一个工具调用 Agent

> 课程：Datawhale《Deep Agents 实战》第 2 章 [快速开始](https://datawhalechina.github.io/deepagents-in-action/chapters/ch02-quickstart/#%E5%AE%8C%E6%95%B4%E4%BB%A3%E7%A0%81)
> 完成日期：2026-09-16
> 运行环境：WSL2 (Ubuntu) + conda base + `research_deepagent/.venv`（uv 创建）
> 关键版本：deepagents 0.7.13 / langchain 1.4.0 / langchain-openai 1.6.2 / langgraph 1.2.11 / Python 3.12.4

---

## 一、核心结论

短短十几行代码即可跑通一个带工具调用的 Agent，本质是三件事的组合：

1. **`ChatOpenAI` + `base_url`**：用 OpenAI 兼容接口接入国内平台（硅基流动），换 URL + Key + 模型名即可切换平台
2. **`create_deep_agent(model, tools, system_prompt)`**：把模型、工具、角色绑定成一个 LangGraph 编译图
3. **`agent.invoke({"messages": [...]})`**：点火。内部是"模型 ⇄ 工具"的循环，跑完返回整个状态，`result["messages"][-1].content` 是最终回答

最小可用代码（`easyagents/weatheragent.py`，实测跑通）：

```python
import os
from langchain_openai import ChatOpenAI
from deepagents import create_deep_agent

model = ChatOpenAI(
    model=os.environ.get("MODEL_NAME", "Qwen/Qwen2.5-7B-Instruct"),
    api_key=os.environ["SILICONFLOW_API_KEY"],
    base_url="https://api.siliconflow.cn/v1",
)

def get_weather(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"

agent = create_deep_agent(
    model=model,
    tools=[get_weather],
    system_prompt="You are a helpful assistant.",
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "上海明天天气怎么样？"}]}
)
print(result["messages"][-1].content)
```

## 二、tools 不是模型自带的

模型是冻结的文本函数，"调用工具"是四方协作的假象：

| 步骤 | 执行者 | 做什么 |
|------|--------|--------|
| ① 注册 | `create_deep_agent()` | 把函数名 + docstring + 类型注解翻译成 JSON Schema 发给模型 |
| ② 决策 | 模型 | 只输出 `tool_calls`（"点菜单"），不执行任何代码 |
| ③ 执行 | LangGraph 框架 | 在你的电脑上真的调用 Python 函数 |
| ④ 回填 | 框架 | 工具结果包装成 ToolMessage 喂回模型，模型再推理出最终回答 |

**关键认知：**

- 模型看不见函数体（return 语句），只看到函数签名和 docstring。**docstring 不是注释，是模型的说明书**，写差了模型就不会调用或乱调用
- `get_weather` 返回写死的晴天是课程刻意的 mock 设计：把函数体换成真实 API 调用，①②④ 一步都不用改——这就是工具调用的架构优势
- 模型知识在训练时冻结，拿不到实时信息（明天的天气、最新新闻）；**工具是模型连接真实世界的唯一通道**，这不是锦上添花而是刚需
- 除了传入的 tools，Deep Agents 还提供文件系统、子 Agent 等 Harness 能力；v0.7 不再默认启用任务规划，需要 `write_todos` 时要显式传 `middleware=[TodoListMiddleware()]`

## 三、实际操作记录

### 0. 环境准备（在项目 venv 内）

```bash
cd ~/self_study/deepagents/projects/research_deepagent
source .venv/bin/activate          # 每次新终端先激活
uv pip install deepagents langchain-openai
# → "Checked 2 packages in 76ms" 表示已装好且满足要求，正常

# 环境变量（注意：等号两边不能有空格）
export SILICONFLOW_API_KEY="sk-..."
export MODEL_NAME="Qwen/Qwen2.5-7B-Instruct"
```

提示符显示 `(research-deepagent) (base)` 不用管：`base` 是 conda 开机自动激活的残留标签，实际生效的是 venv（`which python` 指向 `.venv/bin/python` 即正确）。

### 1. 跑通 weatheragent.py

```bash
python easyagents/weatheragent.py
# → "上海明天天气晴朗，阳光明媚！"（模型基于 mock 返回值组织回答）
```

### 2. 跑通 searchagents.py（真实工具 + 任务规划）

在 weatheragent 基础上升级三处：

- **真实工具**：`internet_search` 包装 Tavily SDK，docstring 带 `Args:` 逐参数说明，参数类型用 `Literal["general","news","finance"]` 枚举约束——工具定义的标准范式
- **中文角色指令**：`research_instructions` 把 Agent 定位成研究员并告知可用工具
- **任务规划**：`middleware=[TodoListMiddleware()]` 显式注入 `write_todos`

```bash
python easyagents/searchagents.py
# → 输出一段关于 LangGraph 的介绍（内容来自真实搜索）
```

### 3. 观察中间消息（验证四步分工）

把最后的 print 换成遍历打印，能看到完整链路：

```python
for i, msg in enumerate(result["messages"]):
    print(f"[{i}] {type(msg).__name__} | {str(getattr(msg, 'content', ''))[:80]}")
    if getattr(msg, "tool_calls", None):
        print("    工具:", [tc["name"] for tc in msg.tool_calls])
```

## 四、试错记录（三个真实踩坑）

### 坑 1：文件没保存，磁盘上是 0 字节

现象：`python easyagents/searchagents.py` 秒退、零输出、零报错、exit=0。

原因：IDE 里写的代码停留在编辑器缓冲区，**磁盘文件还是空的**。Python 执行空文件就是这个表现。两次脚本都栽过（weatheragent 起初还缺 `.py` 后缀）。

教训：**写完代码先 Ctrl+S**；`wc -c 文件名` 可验证文件非空。

### 坑 2：代理掐断连接（Connection reset by peer）

现象：`OpenAIConnectionError: Connection error`，traceback 显示请求走了 `http_proxy` 通道；`api.smith.langchain.com`（LangSmith）也被重置。

原因：终端里挂着代理环境变量，代理未正常工作。硅基流动是国内直连服务，不需要代理。

解决：

```bash
unset http_proxy https_proxy all_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY
python easyagents/searchagents.py   # 立即跑通
```

LangSmith 的警告不影响运行；学习阶段可在 `.env` 里把 `LANGSMITH_TRACING` 设为 `false`。

### 坑 3：export 等号后加空格

现象：`export MODEL_NAME= "Qwen/..."` 报 `not a valid identifier`。

原因：bash 把它解析成两条——设置 `MODEL_NAME` 为空值 + 导出一个叫 `Qwen/...` 的非法变量名。**shell 变量赋值等号两边永远不要空格**。

## 五、agent.invoke 的执行流程

`invoke` 不是一次调用，而是 LangGraph 驱动的循环：

```
START → [model 节点] ⇄ [tools 节点] → END
```

单次 invoke 的时间线：

1. **初始化状态**：输入放进 state 的 messages 列表
2. **model 节点——中间件洋葱圈**：请求依次穿过 filesystem（注入虚拟文件系统工具）→ subagents（注入 task 工具）→ summarization（上下文过长时压缩）→ todo（注入 write_todos + 说明）→ prompt_caching，最内层才是真正的模型 HTTP 调用
3. **路由判断**：返回的 AIMessage 有 `tool_calls` → 框架执行工具，结果追加进 messages，**回到第 2 步**；无 tool_calls → 走到 END
4. **返回最终 state**：`result["messages"]` 含全程消息，`[-1]` 是最终回答

保护机制：递归上限（防无限循环，超限抛 `GraphRecursionError`）、summarization（长任务自动压缩历史）。

**为什么跑 searchagents 看不到 todo 内容？** 三个原因叠加：
- `TodoListMiddleware` 只是"多塞一个工具"，用不用由模型决定；简单问题模型不会主动列计划
- 只打印 `[-1]`，中间的 write_todos 调用看不到
- 7B 小模型对"何时该规划"的判断弱。想触发：在 system_prompt 里明确要求"开始前必须先用 write_todos 列出计划"

## 六、7B 模型的幻觉实录

searchagents 的输出中，模型声称"LangGraph 是由阿里云开发的"——**错误**，LangGraph 是 LangChain 公司（LangChain Inc.）开发。这印证了小模型"知识冻结 + 可靠性差"的双重短板，也说明：

- 即使接了真实搜索工具，小模型仍可能把幻觉混进最终回答（搜索结果只是"事实来源"之一，改写环节仍靠模型）
- 课程推荐复杂任务换 `zai-org/GLM-5.2` 等更强模型，正是为了压低这类幻觉

## 七、平台切换：base_url 模式的正确用法

OpenAI 兼容协议下，换平台 = 换三样东西：

| 平台 | base_url | 模型名示例 |
|------|----------|-----------|
| 硅基流动 | `https://api.siliconflow.cn/v1` | `Qwen/Qwen2.5-7B-Instruct` |
| DeepSeek | `https://api.deepseek.com/v1` | `deepseek-chat` |
| 智谱 | `https://open.bigmodel.cn/api/paas/v4` | `glm-4.6` |

但**只改 `.env` 没用**——脚本里 `base_url` 是硬编码的，且脚本没有 `load_dotenv()`，`.env` 不会自动加载。正确姿势是把三个变量全部交给环境变量：

```python
model = ChatOpenAI(
    model=os.environ["MODEL_NAME"],
    api_key=os.environ["LLM_API_KEY"],
    base_url=os.environ["LLM_BASE_URL"],
)
```

`.env` 写一套"当前使用"的配置，切平台只改三行、代码零改动；每次运行前 `set -a; source .env; set +a` 导出（或在脚本开头加 `load_dotenv()`）。各平台 key 不通用，切平台时 key 和 URL 要一起换。

## 八、概念辨析

- **tools**：普通 Python 函数 + 让模型看得见的元数据（函数名、docstring、类型注解）；执行永远发生在本地框架侧
- **docstring**：模型决定"何时调用、如何填参"的唯一依据，工具开发中是功能性组件而非注释
- **mock 工具**：返回写死值的占位工具，用于隔离"链路学习"与"数据源接入"两个关注点
- **`invoke` / `stream` / `ainvoke`**：同步跑完 / 逐步吐出状态 / 异步版，分别适合脚本调试、实时展示、Web 并发
- **Middleware（中间件）**：在模型调用前对请求做加工的洋葱圈管道；Deep Agents 的 Harness 能力（文件系统、子 Agent、todo）都是这样注入的

---

> 上一篇：[AgentSeek 环境搭建](../agentseek-环境搭建/README.md)
> 下一篇预告：`@tool` 装饰器与复杂参数 Schema、虚拟文件系统
