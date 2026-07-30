# Agent 工作流设计：多轮推理、工具编排与状态管理

> Function Calling 让你调了一个工具，Agent 让你调了一组工具。区别不在"多调了几次"，而在谁来管理调用顺序——是你写死 if-else，还是模型自己迭代决策。

---

## 1. Agent 到底是什么？

一句话：**Agent = LLM + 工具 + 循环**。

```python
# Agent 的最小实现：一个 while 循环
def agent(query: str, tools: list, max_steps: int = 10) -> str:
    messages = [{"role": "user", "content": query}]
    for step in range(max_steps):
        response = llm.chat(messages, tools=tools)
        if response.tool_calls is None:
            return response.content           # 终止
        for tc in response.tool_calls:
            result = execute(tc)
            messages.extend([response, tool_result(tc.id, result)])
    return "未收敛"
```

复杂不在于这个循环，而在于循环内的三个工程问题：
1. **规划**：LLM 怎么拆解复杂任务为多个工具调用？
2. **编排**：什么时候串行、什么时候并行、什么时候选另一条路重试？
3. **状态**：多轮之间消息怎么管理？上下文爆了怎么办？

---

## 2. 规划模式：ReAct vs Plan-Execute

### 2.1 ReAct（推理 + 行动交替）

每轮先思考再行动，适合"走一步看一步"的任务：

```
Step 1: Thought: 用户问"A公司最近三个月营收"，我需要先找到A公司的股票代码
        Action: search_company(name="A公司")
        Observation: {股票代码: "000001", 名称: "甲公司"}

Step 2: Thought: 拿到了代码 000001，可以用它查财务数据
        Action: get_financials(stock_code="000001", period="2024Q1-2024Q3")
        Observation: {营收: [12.3亿, 14.1亿, 15.6亿]}
```

```python
def react_agent(query: str, tools: list, max_steps: int = 10):
    messages = [{
        "role": "system",
        "content": """按以下格式响应：
        Thought: 当前需要做什么
        Action: 工具名
        Action Input: 工具参数 JSON
        得到结果后，如果信息足够就输出 Final Answer，否则继续 Thought"""
    }, {"role": "user", "content": query}]

    for _ in range(max_steps):
        response = llm.chat(messages)
        if "Final Answer" in response:
            return response
        # 解析 Thought → Action → Action Input → 执行 → 追加 Observation
        action, args = parse_action(response)
        result = execute(action, args)
        messages.append(f"Observation: {result}")
```

ReAct 适合路径不确定的工具调用——每一步依赖上一步的结果决定下一步。

### 2.2 Plan-Execute（先规划再执行）

```python
def plan_execute_agent(query: str, tools: list):
    # Step 1: 生成完整计划
    plan = llm.chat([{
        "role": "system",
        "content": "将用户任务拆解为步骤列表，每步标注依赖。输出 JSON: [{\"step\":1,\"desc\":\"...\",\"tool\":\"...\",\"depends_on\":[]}]"
    }, {"role": "user", "content": query}])

    steps = json.loads(plan)

    # Step 2: 按依赖拓扑排序执行
    results = {}
    for step in sorted(steps, key=lambda s: max([0] + [s2["step"] for s2 in steps if s2["step"] in s["depends_on"]])):
        args = {k: results.get(v, v) for k, v in step["inputs"].items()}  # 依赖结果注入
        results[step["step"]] = execute(step["tool"], args)

    # Step 3: LLM 汇总
    return llm.chat([{"role": "user", "content": f"基于以下结果回答: {results}\n问题: {query}"}])
```

| 模式 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| ReAct | 路径不确定、需要中间结果决策 | 灵活、能自适应调整 | 轮次多、延迟高 |
| Plan-Execute | 步骤明确、可并行 | 快、可做批量优化 | 遇到意外无法变通 |

生产环境建议两套都备：默认走 Plan-Execute（快），plan 执行到一半发现结果不符合预期时兜底切 ReAct（灵）。

---

## 3. 工具编排：什么时候并行、什么时候串行

上篇 Function Calling 讲了并行调用。Agent 场景更复杂——你不知道哪些调用有依赖。

### 依赖感知编排

```python
def orchestrate(tool_calls: list) -> list:
    """
    根据参数依赖关系判断串行/并行
    规则：参数值中引用了其他函数输出 → 必须等那个函数先执行
    """
    results = {}
    executed = set()
    pending = list(tool_calls)

    while pending:
        # 找出一批没有依赖的（可并行执行）
        batch = [tc for tc in pending
                 if all(ref not in [t["name"] for t in pending]
                        for ref in extract_dependencies(tc["args"]))]

        if not batch:
            # 死锁检测：取第一个串行执行破局
            batch = [pending[0]]

        # 并行执行这一批
        batch_results = parallel_execute(batch, results)
        results.update(batch_results)
        executed.update(tc["name"] for tc in batch)
        pending = [tc for tc in pending if tc["name"] not in executed]

    return results
```

**核心直觉**：没有参数依赖的调用，一律并行。有依赖的自然串行。不需要人工标注哪个函数依赖哪个——从参数引用关系自动推导。

---

## 4. 状态管理：context 是 Agent 的瓶颈

### 4.1 上下文窗口管理

一个 5 轮 ReAct Agent，messages 可能的膨胀：

```
轮 1: system + user (500 token) → assistant + tool_call (300) → tool_result (2000)
轮 2: [所有历史] + assistant (300) + tool_result (1500)
轮 3: [所有历史] + assistant (100) → Final Answer
总计: 约 6000 token，每多一轮加 1500-2500
```

超过 context 窗口的后果是灾难性的——LLM 丢失早期消息，行为退化。

**三条实用策略**：

```python
def manage_context(messages: list, max_tokens: int = 8000) -> list:
    total = sum(estimate_tokens(m) for m in messages)

    if total <= max_tokens:
        return messages

    # 策略 1：压缩 tool_result（只保留摘要）
    for msg in messages:
        if msg["role"] == "tool" and estimate_tokens(msg) > 1000:
            msg["content"] = summarize(msg["content"])

    # 策略 2：裁剪中间轮（保留首尾）
    if estimate_tokens_sum(messages) > max_tokens:
        system = messages[0]
        recent = messages[-6:]  # 保留最后 3 轮（user + assistant + tool 各 2 条）
        summary = llm.chat([{
            "role": "user",
            "content": f"用一段话概括以下对话的关键发现: {serialize(messages[1:-6])}"
        }])
        messages = [system, {"role": "user", "content": f"历史摘要: {summary}"}] + recent

    return messages
```

### 4.2 状态持久化

Agent 任务可能跨多次 HTTP 请求（中间需要人工确认）。状态必须持久化到外部存储：

```python
import pickle, redis

class AgentState:
    def __init__(self, session_id: str):
        self.session_id = session_id
        self.messages: list = []
        self.plan: list = []          # Plan-Execute 的计划步骤
        self.completed_steps: set = set()
        self.intermediate_results: dict = {}

    def save(self):
        redis.setex(f"agent:{self.session_id}", 3600,
                     pickle.dumps(self))

    @classmethod
    def load(cls, session_id: str):
        data = redis.get(f"agent:{session_id}")
        return pickle.loads(data) if data else cls(session_id)
```

---

## 5. 生产级 Agent 的完整骨架

```python
class ProductionAgent:
    def __init__(self, llm, tools, max_steps=10, max_context=8000):
        self.llm = llm
        self.tools = tools
        self.max_steps = max_steps
        self.max_context = max_context

    def run(self, query: str, session_id: str = None) -> str:
        state = AgentState.load(session_id) or AgentState(session_id or uuid4())
        state.messages.append({"role": "user", "content": query})

        for _ in range(self.max_steps):
            state.messages = manage_context(state.messages, self.max_context)   # 上下文裁剪
            response = self.llm.chat(state.messages, tools=self.tools)

            if response.tool_calls is None:
                state.save()
                return response.content

            batch = orchestrate(response.tool_calls)    # 依赖感知编排
            results = parallel_execute(batch)
            state.intermediate_results.update(results)
            state.messages.extend([{"role": "tool", "content": r} for r in results])
            state.save()

        return "达到最大步数，任务未完成"
```

---

## 6. 关键决策清单

- [ ] 是否同时实现了 ReAct 和 Plan-Execute 两套规划模式？Plan 优先走不通切 ReAct？
- [ ] 工具编排是否根据参数依赖自动判定串行/并行（而非人工标注）？
- [ ] 是否实现了上下文窗口管理（tool_result 截断 + 中间轮摘要）？
- [ ] Agent 状态是否持久化到外部存储（支持跨请求会话）？
- [ ] 是否设置了 max_steps 安全阀？
- [ ] 每个工具调用是否有超时兜底？

---

*下一篇预告：信贷风控方向——「特征工程实战：从原始数据到模型入模的完整链路」*
---

**文档元信息：**
- **发布日期**：2026-07-26（根据文件名推断）
- **作者**：CuriousLinYu
- **来源**：https://github.com/CuriousLinYu/blog （GitHub 仓库）