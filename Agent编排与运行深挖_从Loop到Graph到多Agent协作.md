# AI Agent 编排与运行：从 while 循环到工业级 Harness

> **摘要**：你每天用的 AI 编程助手（CodeBuddy、Claude Code、Cursor），核心不过是一个 while 循环套着一个大模型 API。本文从这个最小原理出发，逐层拆解 Loop Engineering、Graph Engineering、Harness 三层架构，再到多 Agent 协作和 Java 生态实现（LangChain4j / Spring AI），最后补充 function call、多模态、生图原理和产品架构选型。读完后你能完整解释：从你敲下一个问题到 AI 回答的那 30 秒里，到底发生了什么。
>
> **适合读者**：有 Java / Spring Boot 后端经验，对 AI Agent 开发好奇但未深入实践的工程师。
>
> **技术版本**：LangChain4j 1.13 / Spring AI 2.0 GA / LangGraph（2026 年 7 月）

---

## 一、先分清三个词：Harness、Loop、Graph

2026 年 Agent 领域讨论最多的三个词，很多文章把它们并列比较，但它们其实在不同的层级：

```
┌─────────────────────────────────────────────────┐
│  Harness（运行外壳）                              │
│  管：上下文窗口、工具执行、权限、日志、中断、重试     │
│                                                 │
│   ┌─────────────────────────────────────────┐   │
│   │  编排层：决定"下一步干什么"的两种方式        │   │
│   │                                         │   │
│   │   方式A: Loop        方式B: Graph        │   │
│   │   while循环+模型自决   预画节点图+程序控制    │   │
│   └─────────────────────────────────────────┘   │
│                     ▼                           │
│              LLM API（messages进，回复出）          │
└─────────────────────────────────────────────────┘
```

**一句话总结**：Harness 回答"Agent 在哪跑、谁替它干脏活"；Loop 和 Graph 回答"下一步谁说了算"——Loop 是模型说了算，Graph 是你画的图说了算。三者是包含和选择的关系，不是三选一。

---

## 二、Loop Engineering：40 行代码就是一个完整 Agent

Claude Code、CodeBuddy、OpenAI Agents SDK、Codex CLI，核心都是同一个 while 循环。

### 底层原理

大模型 API 响应里有个关键字段 `finish_reason`：值为 `stop` 表示"我说完了"；值为 `tool_calls` 表示"我要调工具"。**整个 Loop Engineering 就建立在这一个字段上**：

```java
List<Message> messages = new ArrayList<>();
messages.add(system("你是天气助手，可用工具：getWeather(city)"));
messages.add(user("深圳明天适合跑步吗？"));

while (true) {                                  // ← 这个 while 就是 Agent 的"命"
    ChatResponse resp = llmApi.chat(messages, toolSchemas);

    if (resp.finishReason() == STOP) {           // 模型说完了 → 循环结束
        return resp.text();
    }
    if (resp.finishReason() == TOOL_CALLS) {     // 模型要工具 → 替它执行
        messages.add(resp.assistantMessage());   // 先把"它想调啥"记入历史
        for (ToolCall tc : resp.toolCalls()) {
            String result = myTools.execute(tc.name(), tc.args());
            messages.add(toolResult(tc.id(), result));
        }
        // 不 return，回到 while 顶部，带着新历史再问一次模型
    }
}
```

### 具体数据走一遍

用户问"深圳明天适合跑步吗？"，messages 数组的两次变化：

**第 1 圈发出去：**
```json
[ {"role":"system", "content":"你是天气助手..."},
  {"role":"user",   "content":"深圳明天适合跑步吗？"} ]
```
模型返回：`finish_reason=tool_calls` → `{"name":"getWeather","args":{"city":"深圳"}}`

代码执行 `getWeather("深圳")` → 返回 `{"temp":31,"rain":true,"humidity":88}`

**第 2 圈发出去（长大了！）：**
```json
[ {"role":"system",    "content":"你是天气助手..."},
  {"role":"user",      "content":"深圳明天适合跑步吗？"},
  {"role":"assistant", "tool_calls":[{"id":"call_a1","name":"getWeather","args":"{\"city\":\"深圳\"}"}]},
  {"role":"tool",      "tool_call_id":"call_a1", "content":"{\"temp\":31, \"rain\":true, \"humidity\":88}"} ]
```
模型返回：`finish_reason=stop` → "明天31度有雨湿度88%，不建议户外跑步..."

循环结束，共 2 圈。

### 关键认知

模型全程**没有**联网查天气。它只是看到工具清单后说了句"去查深圳天气"。真正查的是你的 Java 方法。**所谓 Agent 的"自主性"，本质是把"下一步干什么"的决定权从 if-else 交给了模型的每一次输出。**

### 业内实例

| 产品 | 核心循环之外加了什么 |
|------|---------------------|
| **Claude Code / CodeBuddy** | 上下文压缩、工具权限审批、子 Agent |
| **OpenAI Agents SDK** | `Runner.run(agent, input)` 封装 + max_turns + handoff + guardrails |
| **LangChain4j AiServices** | 把整个循环藏进一个 Java 动态代理 |

> **一句话**：Loop Engineering 就是一个 while(true)，模型说"调工具"就替它调，说"完事了"就退出。所有花活都是给这个 while 加保险。

---

## 三、Graph Engineering：把流程画成图，模型只在节点里干活

代表框架：LangGraph（Python/JS）、LangGraph4j（Java）、Spring AI Alibaba Graph。适合"错一步就出事故"的业务场景。

### 核心思想

Loop 的问题：模型说了算 → 它可能一直转不停、走冤枉路、在第 7 圈忘了第 1 圈干过啥。银行审批敢这么跑吗？不敢。于是换个思路——**把流程做成显式的状态机**：

```java
// 1. 定义共享状态
class LoanState { String applyNo; BigDecimal amount; int creditScore; String decision; }

// 2. 节点 = 普通函数
graph.addNode("查征信",   state -> { state.creditScore = creditApi.query(state.applyNo); });
graph.addNode("规则初筛", state -> { /* 纯 if-else，不花一分钱模型费 */ });
graph.addNode("LLM分析",  state -> { state.evidences = llm.analyze(state); });
graph.addNode("人工复核", state -> { throw new Interrupt(); });  // 挂起等真人

// 3. 边 = 转移规则（你的代码判断，不是模型即兴发挥）
graph.addEdge("查征信", "规则初筛");
graph.addConditionalEdge("规则初筛", state ->
    state.creditScore < 550 ? "直接拒绝" :
    state.amount.compareTo(new BigDecimal(500000)) > 0 ? "人工复核" : "LLM分析");
```

### 底层原理三件套

| 机制 | 做什么 |
|------|--------|
| **超步执行（Super-step）** | 引擎按"回合"跑：执行激活节点 → 合并状态 → 计算下一回合激活谁。本质是个调度器 |
| **Checkpointer** | 每回合结束全量序列化存库。带来崩溃恢复和时间旅行（回到第 5 步改参数重跑） |
| **Interrupt** | 节点抛中断 → 引擎存档挂起 → 等人工操作 → 从断点续跑。Human-in-the-loop 的本质就是"存档 + 等待 + 读档" |

> **一句话**：Graph Engineering 就是不放心让模型自己决定路线，提前把地图画死，模型只负责在指定的格子里填内容。

---

## 四、Anthropic 的五种工作流模式 + 选择铁律

Anthropic《Building Effective Agents》把业内实践收敛成了标准词汇表：

| 模式 | 形状 | 典型场景 |
|------|------|----------|
| **Prompt Chaining** | A → B → C 串行 | 写初稿 → 润色 → 翻译 |
| **Routing** | 先分类再分发 | 客服：退款问题给 A，技术问题给 B |
| **Parallelization** | 同时跑多路汇总 | 同一份合同让 3 个视角同时审 |
| **Orchestrator-Workers** | 指挥官动态拆任务派给工人 | CodeBuddy 派子 Agent 搜代码 |
| **Evaluator-Optimizer** | 生成 → 打分 → 不合格重来 | 生成 SQL → 试执行 → 报错改 |
| **Autonomous Agent** | 裸 Loop，模型全权决定 | 修一个不知道在哪的 bug |

前五种是**工作流**（路径你定，可预测），最后一种才是**完全放权**。

**选择铁律：能用工作流就不用自主 Agent。** 确定性每让渡一分给模型，你就要多付一分的测试、监控、兜底成本。实战几乎都是混合体：主干用工作流钉死，只在"确实无法预知路径"的节点里放一个小 Loop。

---

## 五、Harness：循环之外那 80% 的脏活

为什么裸循环 40 行，Claude Code 却是几十万行？差的就是 Harness。

### 三大核心能力

**1. 上下文管家**

模型窗口 200K token，循环转 50 圈历史轻松爆掉。Harness 做压缩（compaction）：把旧对话摘要成一段话替换原文。

```java
void maybeCompress(List<Message> messages) {
    if (tokenCount(messages) > CONTEXT_LIMIT * 0.8) {
        List<Message> old = messages.subList(0, messages.size() - KEEP_LAST_N);
        String summary = llm.call("请用 200 字总结以下对话：" + old);
        messages.clear();
        messages.add(systemMsg);
        messages.add(user("【历史摘要】" + summary));
        messages.addAll(keepLastN);
    }
}
```

**2. 工具执行沙箱 + 权限**

模型说"执行 rm -rf"，Harness 不能照办。要有白名单、审批弹窗、超时熔断。

```java
for (ToolUse block : llmResponse.toolUses) {
    if (isDangerous(block)) {
        boolean ok = askUser("要执行 " + block.name + " 吗？");
        block.result = ok ? execute(block) : "用户拒绝";
    } else {
        block.result = execute(block);
    }
    messages.add(toolResult(block.result));
}
```

**3. 子 Agent（开新循环干分支任务）**

```java
String spawnSubAgent(String task, ToolSet narrowTools) {
    List<Message> child = [ childSystemPrompt ];
    while (true) {
        LlmResp r = llm.call(child);
        if (r.finishReason == "end_turn") break;
        for (ToolUse b : r.toolUses)
            child.add(execute(b, narrowTools));  // 只能用收窄后的工具
    }
    return lastText(child);  // 整段对话塌缩成一句话
}
```

> **一句话**：模型是发动机，Loop/Graph 是变速箱，Harness 是整台车的其余部分——底盘、刹车、油表、安全气囊。2026 年的共识：产品差距主要拉开在 Harness。

---

## 六、多 Agent 协作：底层只有一个机制

所谓"Agent A 把任务交给 Agent B"，代码层面就是：**B 被包装成 A 工具清单里的一项**。没有新魔法，全是循环的套娃。

| 模式 | 谁指挥 | 一句话机制 |
|------|--------|------------|
| **Supervisor 主从** | 一个主管 Agent | 主管把每个下属注册为工具，动态点名 |
| **Handoff 交接** | 接力，无主管 | 特殊工具调用直接移交对话控制权 |
| **群聊** | 发言选择器 | 所有 Agent 共享聊天室，轮流发言 |
| **层级/剧组** | 按角色分工 | 预设角色，按流程调度 |

**为什么拆多 Agent？** 两个硬理由：
1. **上下文隔离**——搜代码的子 Agent 翻了 80 个文件的垃圾信息，不该污染主 Agent 宝贵的窗口
2. **提示词专精**——"你是 SQL 专家"比"你什么都会"效果好得多

---

## 七、Java 生态实现拆解

### 7.1 LangChain4j：AiServices = 动态代理 + 反射 + 内置 Loop

```java
interface WeatherAssistant {
    @SystemMessage("你是天气助手")
    String chat(String userMessage);
}

class WeatherTools {
    @Tool("查询城市实时天气")
    String getWeather(@P("城市名") String city) { return weatherApi.query(city); }
}

WeatherAssistant assistant = AiServices.builder(WeatherAssistant.class)
    .chatModel(model)
    .tools(new WeatherTools())
    .chatMemory(MessageWindowChatMemory.withMaxMessages(20))
    .build();

assistant.chat("深圳明天适合跑步吗？");  // 一行调用，循环在里面转
```

`build()` 背后干了三件事：

| 魔法表象 | 拆穿后的实现 |
|----------|-------------|
| 接口没有实现类却能调用 | **JDK 动态代理** `Proxy.newProxyInstance()`，和 MyBatis Mapper 完全同一个套路 |
| 模型怎么知道有 getWeather 可用 | **反射扫描 @Tool**：读方法名、参数名、描述，生成 JSON Schema |
| 工具自动被调用、自动多轮 | **内置 Loop**：检测到 ToolExecutionRequest 就反射调用方法，直到模型返回纯文本 |

1.13 版新增的 `langchain4j-agentic` 模块直接把五种工作流模式做成 builder：

```java
// 链式
var flow = AgenticServices.sequenceBuilder().subAgents(writer, editor).build();

// 评审循环
var loop = AgenticServices.loopBuilder()
    .subAgents(writer, scorer)
    .maxIterations(3)
    .exitCondition(scope -> scope.readState("score", 0.0) >= 0.8)
    .build();

// 主从
var team = AgenticServices.supervisorBuilder().subAgents(writer, editor, researcher).build();
```

### 7.2 Spring AI 2.0：ChatClient + Advisor 链 + ToolCallingManager

```python
class WeatherTools:
    @Tool(description="查询城市实时天气")
    def get_weather(city: str) -> dict:
        return weather_api.query(city)

answer = chat_client.prompt()
    .user("深圳明天适合跑步吗？")
    .tools([WeatherTools()])
    .advisors(MessageChatMemoryAdvisor.builder(memory).build())
    .call().content()
```

三个核心机制：

| 机制 | 类比 | 干什么 |
|------|------|--------|
| **Advisor 链** | Servlet Filter / Spring AOP | 请求发模型前、响应回来后依次穿过。记忆、RAG、日志、安全审查全是可插拔的"环绕通知" |
| **ToolCallingManager** | MVC HandlerAdapter | 检测 tool_calls → 反射执行 → 结果拼回 → 自动再请求，直到 stop |
| **ChatModel 抽象** | JDBC | 统一抹平各家 API 差异，换模型 = 换 starter |

**选型一句话**：纯练手/非 Spring 项目用 LangChain4j 更快见效；公司项目已是 Spring Boot 就直接 Spring AI。两者概念完全同构，学会一个另一个半天上手。

---

## 八、function call / RAG / MCP / 多模态，到底在谁那？

| 能力 | 在模型内吗 | 真相 |
|------|-----------|------|
| **function call** | 部分 | 工具定义是 harness 写的文本；模型被微调成按格式吐 JSON；**执行永远是 harness** |
| **RAG** | 否 | 100% harness 侧（检索、切块、拼进 prompt） |
| **MCP** | 否 | 100% harness 侧（harness 与外部工具 server 的标准协议） |
| **多模态·看图** | **是** | 模型内有视觉编码器，图片字节切成视觉 token 和文字 token 一起进 Transformer |
| **多模态·生图/视频** | 否 | LLM 不生成像素，它通过 tool_use 让 harness 去调扩散模型 |

### function call 完整链路

```
1. harness 把工具定义写成 JSON Schema，塞进 API 请求
   tools = [{ name:"search_content", input_schema:{...} }]

2. 模型返回 tool_use 块（结构化 JSON 文本）
   { tool_use: { name:"search_content", input:{ pattern:"getUserMenu" } } }

3. harness 解析 JSON → 执行（如 ripgrep）→ 结果塞回 messages

4. 再调一次模型，模型基于结果产出最终回答
```

**模型全程没碰文件系统，它只吐了那段 JSON 文本。**

---

## 九、多 Agent 并行 = harness 同时发 N 个 HTTP 请求

```java
ExecutorService pool = ...;
List<Future<String>> futures = new ArrayList<>();
for (SubTask t : split(task)) {
    futures.add(pool.submit(() -> spawnSubAgent(t, tools)));
}
List<String> parts = futures.stream().map(Future::get).toList();
String merged = reduce(parts);
messages.add(user("并行结果：" + merged));
```

模型本身不并行。每次 LLM 调用是一次独立的 HTTP 请求。所谓"并行"发生在 harness 编排层。

---

## 十、模型是个无状态函数

**这是全文最重要的一句话**：

```
模型 = f(messages, params) → (text, tool_use)
```

无状态、无记忆、不持久化。每次调用都从零开始。

| 你以为的 | 实际 |
|----------|------|
| 模型"记得"我们之前的对话 | harness 把历史存 DB，每次调用重新拼进 messages |
| 上下文窗口 = 大脑容量 | 单次调用能塞进 messages 的 token 上限 |
| skill / rules 是模型自带的 | harness 在调模型前拼进 system prompt |

```java
String sys = BASE_PROMPT
    + rules.load()
    + skills.match(query)
    + memory.load(session);
List<Message> req = [ system(sys), ...history, user(input) ];
llm.call(req);  // 模型拿到这整坨；它自己不存任何东西
```

---

## 十一、生图原理：不是在白纸上画，是擦去噪声

| 阶段 | 做什么 | 谁干的 |
|------|--------|--------|
| 文本编码 | 把 prompt 编码成 embedding 作为"条件" | CLIP / LLM |
| 迭代去噪 | 从纯随机噪声图开始，20~50 步逐步"雕"成目标图像 | 扩散模型（U-Net / DiT） |
| 解码成像素 | 从 latent 解码回 RGB 像素 | VAE Decoder |

**噪声**就是一张每个像素从高斯分布随机采样的图（电视雪花）。训练时学"预测每步加了多少噪声"，推理时反过来逐步减去。不能一步到位，因为模型只被训练过"减一点点噪声"这个动作。

现代模型（SDXL、Sora）用 Diffusion Transformer 替代 U-Net，但生成机制是**扩散去噪**，不是自回归逐 token。

---

## 十二、产品架构选型：不是"个人 vs 团队"，是摩擦矩阵

| 产品 | 架构 | 计费 | 适合谁 |
|------|------|------|--------|
| **Claude Code** | 纯本地 CLI | 按 token 付费 | 要灵活切模型、隐私、随处跑 |
| **CodeBuddy** | 混合（云端编排 + 本机壳） | 订阅/免费额度，便宜 | 国内个人+团队、要便宜+开箱 |
| **Cursor** | 混合（云端） | 订购制 | 要顺滑 IDE 体验、团队协作 |

**真正的因果链**：要不要后端 → 状态存哪 → 能协作还是能随处跑。功用差异是架构的物理必然，不是产品任性。

| 倾向 Claude Code | 倾向 CodeBuddy / Cursor |
|-----------------|------------------------|
| 愿为质量付高价或自己选便宜模型 | 要低价/免费额度 |
| 能折腾配 key、命令行 | 要下载即用 |
| 不愿代码出本机 | 接受云端处理换方便 |
| 单人/随处跑 | 团队共享 |

---

## 总结

你在 AI 编程助手里敲下一个问题后发生的一切：

1. **Harness** 把你的问题、系统提示词、工具清单打包成 messages 数组，发起第一次请求
2. 模型返回 `finish_reason=tool_calls`，Harness 替它执行、结果塞回、再次请求——**Loop** 转了一圈又一圈
3. 上下文管家数着 token，权限层判断"读文件不危险，不弹窗"
4. 如果是信贷审批系统，同样的活会被画成带 checkpoint 的状态图——那就是 **Graph**
5. 落到 Java，LangChain4j 用动态代理 + 反射 + 内置循环帮你藏好，Spring AI 用 Advisor 链 + ToolCallingManager 帮你藏好——**全是你写了多年的老朋友：Proxy、反射、拦截器、状态机**
6. 上下文摘要、权限审批、派子 Agent——**这些"智能行为"没有一个是模型干的，全是 Harness 写的代码。模型只负责动嘴，动手的永远是程序。**

---

---

**来源信息：**
- **文件路径**: https://raw.githubusercontent.com/CuriousLinYu/blog/main/Agent编排与运行深挖_从Loop到Graph到多Agent协作.md
- **作者**: CuriousLinYu (林宇)
- **发布日期**: 2026年7月30日
- **仓库地址**: https://github.com/CuriousLinYu/blog