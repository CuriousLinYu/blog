# OpenSpec 源码架构与 Agent Harness + LLM 协作机制分析

> 分析对象：当前工作区 \`OpenSpec\`
>
> 分析日期：2026-07-29
>
> 核心结论先行：\*\*OpenSpec 不是另一个会写代码的 AI，也不是 LLM SDK；它是位于 Agent Harness 和项目文件之间的“规格驱动控制平面”——用确定性的 CLI、制品依赖图、模板、校验器和归档机制，约束并记录本来容易漂移的 LLM 工作过程。\*\*

说白了就是：\*\*CodeBuddy / Claude Code 是“能看文件、改代码、跑命令的手”，LLM 是“负责理解和判断的脑”，OpenSpec 是“施工图、工序表、验收单和档案柜”。\*\*

---

## 1. 先抓住主要矛盾

用户容易产生的第一个误解是：安装 OpenSpec 后，是不是 OpenSpec 自己调用大模型、理解需求并修改代码？

答案是否定的。

当前源码中：

- 没有 OpenAI、Anthropic、LangChain、Ollama 等 LLM SDK 依赖；
- 没有模型 API Key、messages/completions 调用；
- 没有一个常驻 Agent Runtime；
- 甚至不存在 \`openspec apply\` 这个 CLI 子命令。

真正发生的是：

1. OpenSpec 把工作流写成 Skill / Slash Command Markdown；
2 \`openspec init\` 将这些文件安装进 \`.codebuddy/\`、\`.claude/\` 等 Harness 能识别的目录；
3. CodeBuddy 或 Claude Code 加载工作流提示词；
4. LLM 根据提示词调用 \`openspec status --json\`、\`openspec instructions ... --json\`；
5. OpenSpec CLI 返回确定性的状态、路径、模板和约束；
6. LLM 再通过 Harness 的读写文件、代码搜索、终端等工具完成语义工作和代码实现。

源码证据：

- CLI 依赖中没有 LLM SDK：\`package.json:69-106\`
- CodeBuddy 被登记为支持的 AI 工具：\`src/core/config.ts:22-58\`
- CodeBuddy 命令适配器：\`src/core/command-generation/adapters/codebuddy.ts:1-38\`
- Claude Code 命令适配器：\`src/core/command-generation/adapters/claude.ts:1-42\`
- \`apply\` 工作流明确要求 Agent “Make the code changes required”：\`src/core/templates/workflows/apply-change.ts:92-105\`
- CLI 只注册了 \`instructions\`，没有注册 \`apply\`：\`src/cli/index.ts:581-604\`

---

## 2. 总体架构

\`\`\`mermaid
flowchart TB
    U\[用户<br/>描述需求或调用 /opsx:\*\]

    subgraph H\[Agent Harness：CodeBuddy / Claude Code / Cursor / Codex 等\]
        CMD\[加载 Slash Command / Skill\]
        LOOP\[Agent Tool Loop<br/>选择工具、读取结果、继续推理\]
        TOOLS\[代码搜索 / 文件读写 / Shell / 测试 / Git\]
    end

    LLM\[LLM<br/>语义理解、方案推理、文档生成、代码生成\]

    subgraph O\[OpenSpec\]
        ADAPTER\[Harness 适配层<br/>生成各工具专属命令与 Skill\]
        CLI\[CLI 命令入口<br/>status / instructions / validate / archive ...\]
        GRAPH\[Artifact Graph<br/>制品依赖和 ready/blocked/done\]
        SCHEMA\[Schema + Template<br/>制品类型、顺序、模板、规则\]
        VALIDATOR\[Parser + Validator<br/>规格格式和 delta 一致性校验\]
        ARCHIVE\[Archive / Spec Apply<br/>主规格合并与变更归档\]
        ROOT\[Root / Store<br/>规格根目录和跨仓库存储\]
    end
    FS\[(文件系统 / Git 仓库<br/>proposal.md · specs · design.md · tasks.md · 源码)\]

    U --> CMD
    CMD --> LOOP
    LOOP <--> LLM
    LOOP --> TOOLS
    TOOLS --> CLI
    TOOLS <--> FS

    ADAPTER --> CMD
    CLI --> GRAPH
    CLI --> SCHEMA
    CLI --> VALIDATOR
    CLI --> ARCHIVE
    CLI --> ROOT
    GRAPH <--> FS
    SCHEMA --> GRAPH
    VALIDATOR <--> FS
    ARCHIVE <--> FS
    ROOT <--> FS
\`\`\`

### 2.1 三个角色不能混淆

| 角色 | 对外提供什么 | 依赖谁 | 不负责什么 |
|---|---|---|---|
| OpenSpec | 规格目录、制品图、模板、结构化状态、校验、归档、工具适配文件 | 文件系统；由 Harness 调 CLI | 不理解自然语言，不搜索业务代码，不自行实现任务 |
| Agent Harness | 命令入口、上下文装载、工具调用循环、权限控制、文件与终端能力 | LLM；本地工具；OpenSpec CLI | 本身不定义 OpenSpec 的规格语义 |
| LLM | 理解需求、填充制品、技术判断、改代码、分析验证结果 | Harness 提供的上下文和工具 | 不能天然保证流程稳定、状态持久、输出格式一致 |

因此，\*\*Harness + LLM 本来就能写代码；OpenSpec 的作用不是“赋予写代码能力”，而是把这项能力装入一个可重复、可追踪、可交接的工程协议中。\*\*

---

## 3. OpenSpec 源码分层

## 3.1 可执行入口层

真实启动链：

\`\`\`text
npm 全局命令 openspec
  → package.json 的 bin 映射
  → bin/openspec.js
  → dist/cli/index.js
  → Commander program.parse(argv)
  → 对应命令处理器
\`\`\`

证据：

- npm bin 映射：\`package.json:24-32\`
- 启动脚本加载构建产物：\`bin/openspec.js:1-5\`
- Commander 程序和全局 hook：\`src/cli/index.ts:94-147\`
- CLI 最终解析参数：\`src/cli/index.ts:660-668\`

\`src/cli/index.ts\` 是装配层，负责注册：

- \`init\`、\`update\`
- \`list\`、\`show\`、\`view\`
- \`validate\`
- \`archive\`
- \`status\`
- \`instructions\`
- \`new change\`
- \`schema\`、\`store\`、\`config\` 等命令组

证据：\`src/cli/index.ts:155-658\`

## 3.2 Harness 适配层

OpenSpec 不要求所有 AI 工具采用相同目录和命令命名。它定义了统一的工作流内容，再用 adapter 转换成各 Harness 能识别的文件。

核心模块：

| 模块 | 作用 |
|---|---|
| \`src/core/config.ts\` | 支持工具清单、工具 ID、技能目录和自动探测路径 |
| \`src/core/command-generation/registry.ts\` | 注册各 Harness 的命令适配器 |
| \`src/core/command-generation/generator.ts\` | 用统一内容生成工具专属文件 |
| \`src/core/command-generation/invocation.ts\` | 处理 \`/opsx:apply\`、\`/opsx-apply\`、\`@opsx-apply\` 等调用差异 |
| \`src/core/shared/skill-generation.ts\` | 汇总全部 Workflow，生成 Agent Skill 内容 |
| \`src/core/init.ts\` | 首次安装 Skill 和命令文件 |
| \`src/core/update.ts\` | OpenSpec 升级后重生成这些文件 |

适配器注册证据：\`src/core/command-generation/registry.ts:41-74\`

调用形式并非硬编码一张人工维护表，而是根据 adapter 输出路径推导 namespaced / flat，再叠加工具自己的前缀：\`src/core/command-generation/invocation.ts:1-99\`

这解决了一个实际问题：

- Claude Code / CodeBuddy：目录是 \`commands/opsx/<id>.md\`，调用形式通常为 \`/opsx:<id>\`；
- Cursor 等工具：文件名是 \`opsx-<id>.md\`，调用形式通常为 \`/opsx-<id>\`；
- Amazon Q 使用 \`@\` 前缀。

## 3.3 Schema 与制品图层

默认 \`spec-driven\` Schema 定义四类制品：

\`\`\`mermaid
flowchart LR
    P\[proposal.md<br/>为什么改、改什么\] --> S\[specs/\*\*/\*.md<br/>系统必须表现成什么样\]
    P --> D\[design.md<br/>技术上怎么实现\]
    S --> T\[tasks.md<br/>实施步骤\]
    D --> T
    T --> A\[允许进入 apply\]
\`\`\`

源码事实：

- \`proposal\` 无依赖：\`schemas/spec-driven/schema.yaml:4-36\`
- \`specs\` 依赖 \`proposal\`：\`schemas/spec-driven/schema.yaml:38-127\`
- \`design\` 依赖 \`proposal\`：\`schemas/spec-driven/schema.yaml:129-162\`
- \`tasks\` 同时依赖 \`specs\` 和 \`design\`：\`schemas/spec-driven/schema.yaml:164-201\`
- apply 要求 \`tasks\`，并以 \`tasks.md\` 跟踪进度：\`schemas/spec-driven/schema.yaml:203-207\`

\`ArtifactGraph\` 将 YAML 变成可计算图，提供：

- 拓扑构建顺序；
- 下一批 ready 制品；
- blocked 制品及缺失依赖；
- 工作流是否完成；
- apply 所需制品是否完成。

证据：\`src/core/artifact-graph/graph.ts:12-214\`

这部分是 OpenSpec 最像“工作流引擎”的地方，但它是\*\*轻量文件状态机\*\*，不是 BPM 审批引擎。

## 3.4 Instructions：把确定性状态编译成 LLM 可消费的任务包

\`openspec instructions <artifact> --change <name> --json\` 返回的不是一句普通提示，而是一个结构化任务包，包含：

- \`schemaName\`
- \`changeDir\`
- \`planningHome\`
- \`resolvedOutputPath\`
- \`existingOutputPaths\`
- \`instruction\`
- 项目级 \`context\`
- 当前制品的 \`rules\`
- 依赖制品及路径
- 完成当前制品后会解锁什么
- 模板正文

证据：

- 指令对象字段定义：\`src/core/artifact-graph/instruction-loader.ts:73-108\`
- 模板加载：\`src/core/artifact-graph/instruction-loader.ts:192-234\`
- 指令生成和字段组装：\`src/core/artifact-graph/instruction-loader.ts:301-393\`

这相当于把：

\`\`\`text
“现在该写什么？”
“写到哪里？”
“必须先读什么？”
“格式是什么？”
“项目有什么额外规则？”
\`\`\`

编译成机器可读 JSON，交给 LLM 处理语义内容。

## 3.5 状态检测层

\`openspec status --change ...\` 的核心依据是：每个 artifact 的 \`generates\` 路径是否已经有匹配文件。

- 普通路径：检查该文件是否存在；
- Glob：用 \`fast-glob\` 检查至少一个匹配文件；
- 至少存在一个输出文件，就认为 artifact output exists。

证据：\`src/core/artifact-graph/outputs.ts:13-41\`

状态再结合依赖图被标为：

- \`blocked\`
- \`ready\`
- \`done\`
- 特殊配置下的 \`skipped\`

证据：\`src/core/artifact-graph/state.ts:7-28\`、\`src/commands/workflow/status.ts:106-224\`

这里必须明确：\*\*\`done\` 主要表示文件存在，不表示文件内容正确。\*\*

测试甚至专门证明了乱序创建 \`design.md\` 时，\`design\` 仍会被计为完成，只是下一步仍要求 \`proposal\`：\`test/core/artifact-graph/workflow.integration.test.ts:94-113\`

## 3.6 Parser 与 Validator 层

OpenSpec 的 Validator 校验的是“规格制品”，不是业务程序运行结果。

它能检查：

- 主规格能否解析成 Requirement / Scenario；
- 需求描述是否包含 SHALL / MUST；
- ADDED / MODIFIED 是否至少有一个 Scenario；
- 同一个需求是否同时出现在 ADDED、MODIFIED、REMOVED；
- RENAMED 与其他 delta 操作是否冲突；
- delta section 存在但没有合法 requirement block；
- 主规格是否错误保留 delta 操作标题。

入口：

- \`validate change\`：\`src/commands/validate.ts:195-203\`
- \`validate spec\`：\`src/commands/validate.ts:207-212\`

主要规则证据：

- 解析 Spec / Change 并做 Zod 校验：\`src/core/validation/validator.ts:28-105\`
- ADDED / MODIFIED 场景和规范词检查：\`src/core/validation/validator.ts:216-269\`
- delta 互斥冲突检查：\`src/core/validation/validator.ts:299-334\`
- 无有效 delta 条目检查：\`src/core/validation/validator.ts:343-362\`
- 主规格 Requirement 的 SHALL / MUST 检查：\`src/core/validation/validator.ts:465-475\`

它不能证明：

- 代码真的实现了 Requirement；
- 测试真的覆盖 Scenario；
- 性能、安全、并发行为符合设计；
- \`tasks.md\` 勾选是真实完成而不是误勾。

## 3.7 Archive 与主规格演进层

OpenSpec 同时保存两种规格：

\`\`\`text
openspec/specs/<capability>/spec.md
    长期有效的“当前系统规格”

openspec/changes/<change>/specs/<capability>/spec.md
    本次变更的 delta：ADDED / MODIFIED / REMOVED / RENAMED
\`\`\`

归档时，delta 被同步进主规格，然后 change 目录移动到带日期的 archive 下。

CLI 注册：\`src/cli/index.ts:403-420\`

CLI Archive 实现入口：\`src/core/archive.ts:152-184\`

确定性 delta 合并算法位于 \`src/core/specs-apply.ts\`，负责：

- 新增 Requirement；
- 修改 Requirement；
- 删除 Requirement；
- 重命名 Requirement；
- 保留未被 delta 触及的原内容。

这使主规格不是每次复制一份完整文档，而是像数据库迁移或 Git patch 一样随 change 演进。

但当前系统还提供另一个 Agent-driven \`sync\` 工作流：由 LLM 读取 delta 和 main spec，做“智能局部合并”。模板明确写明这是 agent-driven operation：\`src/core/templates/workflows/sync-specs.ts:10-18\`

因此要区分：

- \`openspec archive\` CLI：有程序化校验、合并和移动；
- \`/opsx:sync\`：LLM 根据规则做语义合并；
- \`/opsx:archive\`：是 Harness 工作流，会组织 status、sync 判断和目录移动。

## 3.8 Root / Store 层

OpenSpec 不强制规格一定与业务代码同仓库。

根目录选择优先级大致是：

1. 显式 \`--store <id>\`；
2. 当前目录向上的最近 OpenSpec root；
3. 项目 \`openspec/config.yaml\` 声明的 store pointer；
4. 全局 \`defaultStore\`；
5. 没有注册 store 时允许隐式当前根目录。

证据：\`src/core/root-selection.ts:1-22\`、\`src/core/root-selection.ts:392-455\`

Store registry 负责：

- store ID；
- 本地 Git 后端路径；
- 可选 remote / branch；
- 全局 registry；
- store 自身身份元数据；
- 文件锁和原子写，避免并发损坏 registry。

证据：\`src/core/store/foundation.ts:28-65\`、\`src/core/store/foundation.ts:289-349\`

端到端测试覆盖了两种模式：

- 业务仓库引用只读产品规格 Store，但设计仍落本仓库：\`test/cli-e2e/capstone-journeys.test.ts:40-103\`
- 代码仓库只有 store pointer，整个变更生命周期都落到外部 Store：\`test/cli-e2e/capstone-journeys.test.ts:105-180\`

---

## 4. OpenSpec 如何配合 CodeBuddy

## 4.1 安装阶段

OpenSpec 明确将 CodeBuddy 登记为工具 ID \`codebuddy\`，技能根目录为 \`.codebuddy\`：\`src/core/config.ts:27-34\`

初始化时：

1. 根据 profile 选择 Workflow；
2. 获取 Skill templates 和 Command templates；
3. 对每个 Harness 选择适配器；
4. 写入 Harness 专属目录。

证据：

- 选择模板：\`src/core/init.ts:702-705\`
- 生成 Skill：\`src/core/init.ts:707-736\`
- 生成命令：\`src/core/init.ts:743-752\`
- \`openspec update\` 重生成：\`src/core/update.ts:264-309\`

CodeBuddy adapter 生成：

\`\`\`text
.codebuddy/commands/opsx/explore.md
.codebuddy/commands/opsx/propose.md
.codebuddy/commands/opsx/apply.md
...
\`\`\`

证据：\`src/core/command-generation/adapters/codebuddy.ts:13-37\`

适配器测试锁定了 CodeBuddy 路径和 frontmatter：\`test/core/command-generation/adapters.test.ts:297-316\`

同时，Skill 生成器会生成类似：

\`\`\`text
.codebuddy/skills/openspec-propose/SKILL.md
.codebuddy/skills/openspec-apply-change/SKILL.md
.codebuddy/skills/openspec-verify-change/SKILL.md
...
\`\`\`

Skill frontmatter 带有：

\`\`\`yaml
allowed-tools: Bash(openspec:\*)
compatibility: Requires openspec CLI.
\`\`\`

证据：\`src/core/shared/skill-generation.ts:125-155\`、\`src/core/shared/allowed-tools.ts:1-11\`

注意：\`allowed-tools\` 只是支持该字段的 Harness 可识别的预授权提示；源码明确说明它不限制其他工具，不识别该字段的工具会忽略：\`src/core/shared/allowed-tools.ts:1-9\`

## 4.2 CodeBuddy 运行阶段

以 \`/opsx:apply add-auth\` 为例：

\`\`\`mermaid
sequenceDiagram
    participant U as 用户
    participant CB as CodeBuddy Harness
    participant M as LLM
    participant OS as OpenSpec CLI
    participant FS as 项目文件系统

    U->>CB: /opsx:apply add-auth
    CB->>CB: 加载 .codebuddy/commands/opsx/apply.md
    CB->>M: 注入工作流、用户参数、项目上下文
    M->>OS: openspec status --change add-auth --json
    OS->>FS: 读取 .openspec.yaml / schema / artifact 文件
    FS-->>OS: 文件状态
    OS-->>M: schema、依赖图、artifact 状态和路径
    M->>OS: openspec instructions apply --change add-auth --json
    OS-->>M: contextFiles、tasks、progress、state、instruction
    M->>FS: 读取 proposal/specs/design/tasks 和业务源码
    M->>FS: 修改业务源码、测试、tasks.md checkbox
    M->>CB: 汇报进度和验证结果
    CB-->>U: 本轮实现结果
\`\`\`

这里：

- CodeBuddy 负责加载命令、把 LLM 和工具接起来；
- LLM 负责决定读哪些代码、如何实现；
- OpenSpec 负责告诉 LLM 当前 change 的确定性状态和应读制品；
- 文件系统负责持久保存规格和实现。

## 4.3 CodeBuddy 集成的真实边界

当前仓库适配器注释写的是 \*\*CodeBuddy Code CLI\*\*，而不是对某个特定 IDE UI 的内部 API 集成：\`src/core/config.ts:33\`

因此它的集成方式是\*\*文件协议集成\*\*，不是：

- 调 CodeBuddy 私有网络 API；
- 嵌入 CodeBuddy 运行时；
- 向 CodeBuddy 注册 MCP Server；
- 替换 CodeBuddy 自己的 Agent Loop。

只要 CodeBuddy 当前产品版本会扫描 \`.codebuddy/commands\` / \`.codebuddy/skills\`，这些工作流就能被发现；反之，OpenSpec 自己不会启动 CodeBuddy。

---

## 5. OpenSpec 如何配合 Claude Code 及其他 Harness

Claude adapter 输出：

\`\`\`text
.claude/commands/opsx/<id>.md
\`\`\`

并在命令 frontmatter 中写入 \`allowed-tools: Bash(openspec:\*)\`：\`src/core/command-generation/adapters/claude.ts:14-41\`

测试锁定了：

- \`/opsx:<id>\` 对应目录结构；
- YAML frontmatter；
- \`allowed-tools\`；
- 工作流正文被原样注入。

证据：\`test/core/command-generation/adapters.test.ts:50-82\`

对其他 Harness，核心工作流内容不变，只转换：

1. 配置目录；
2. 文件名；
3. frontmatter 格式；
4. 命令前缀和 namespaced / flat 调用形式；
5. Skill 中对其他命令的交叉引用。

这就是 OpenSpec 的可移植性来源：\*\*业务规格协议保持一致，Harness 方言由 adapter 解决。\*\*

---

## 6. Harness + LLM 到底怎样工作

抽象成一个最小 Agent Loop：

\`\`\`text
while 任务未完成:
    Harness 收集：用户消息 + 当前命令 Skill + 项目规则 + 工具结果
    LLM 推理：下一步应该调用什么工具
    Harness 执行：搜索 / 读取 / 写文件 / 运行 openspec / 跑测试
    Harness 把结果返回给 LLM
    LLM 更新判断
\`\`\`

OpenSpec 插入这个循环的两个位置：

### 6.1 循环开始前：提供稳定 Workflow Prompt

例如 \`apply-change.ts\` 规定：

- 先选 change；
- 再查 status；
- 再取 apply instructions；
- 必须读取 \`contextFiles\`；
- 逐项实现；
- 完成一个任务立即勾选；
- 不清楚或遇到设计问题要暂停。

证据：\`src/core/templates/workflows/apply-change.ts:20-113\`

### 6.2 循环中：提供确定性 State API

LLM 不需要凭聊天记忆猜：

- 当前 change 是哪个 schema；
- proposal 是否存在；
- 哪个 artifact 被 blocked；
- tasks 在哪里；
- 完成了几项；
- main spec 位于本仓库还是 Store。

它通过 CLI JSON 每次重新查询。

这正是 OpenSpec 的核心价值：\*\*将“容易幻觉的状态”从 LLM 上下文中剥离，放进可重复计算的文件和 CLI。\*\*

---

## 7. OpenSpec 源码到底起什么作用

## 7.1 作用一：把一次聊天变成持久化工程状态

没有 OpenSpec 时，需求、设计和任务往往只存在于聊天历史。换会话、换模型、换开发者后，上下文容易丢失。

OpenSpec 将其固化为：

\`\`\`text
proposal.md  → 变更动机和范围
specs/       → 可测试的系统行为
design.md    → 技术决策和权衡
tasks.md     → 可跟踪实施清单
archive/     → 已完成变更历史
main specs   → 当前系统应有行为
\`\`\`

这是一种项目级长期记忆，而非模型会话记忆。

## 7.2 作用二：把“下一步做什么”变成可计算依赖图

ArtifactGraph 让顺序不是靠 LLM 自觉：

- 没 proposal 时，specs/design blocked；
- 没 specs/design 时，tasks blocked；
- 没 tasks 时，apply blocked。

这是确定性流程控制。

但它只控制“文件依赖是否满足”，不控制“文档内容是否真的高质量”。

## 7.3 作用三：统一 LLM 的输入结构

不同会话、不同模型都能通过 \`instructions --json\` 拿到同样的：

- 模板；
- 输出位置；
- 依赖文件；
- 项目 context；
- artifact rules；
- operation guidance。

项目配置结构证据：\`src/core/project-config.ts:32-76\`

项目 context、artifact rules 和 apply/archive guidance 被刻意分开，避免把建议误当硬状态：\`src/core/project-config.ts:89-108\`

## 7.4 作用四：提供跨 Harness 的工作流分发

同一套工作流可以安装到 CodeBuddy、Claude Code、Cursor、Codex 等工具，而不必为每个工具手工维护一份提示词。

当前支持工具清单：\`src/core/config.ts:22-58\`

## 7.5 作用五：提供规格格式校验

OpenSpec 能比普通 Markdown 更早发现：

- Requirement 没有可验证语义；
- Scenario 缺失；
- delta 操作冲突；
- 文档结构错误。

这降低的是“规格自身坏掉”的风险，不是“代码实现错误”的全部风险。

## 7.6 作用六：维持“当前规格”与“变更历史”

主规格表示现在，delta 表示这次变化，archive 表示历史。

因此 OpenSpec 不只是生成四个文档的模板工具，它还提供了一个类似：

\`\`\`text
当前状态 + 变更补丁 + 历史记录
\`\`\`

的规格版本模型。

## 7.7 作用七：让规格与代码仓库解耦

Store 允许：

- 产品规格集中一个仓库；
- 多个代码仓库只读引用产品规格；
- 低层设计和实现任务保留在各自仓库；
- 或让某代码仓库将全部 planning state 外置。

这对多仓库、多服务、产品规格统一治理有实际价值。

## 7.8 作用八：提供基础可观测性，而不是 AI 推理

\`view\` 命令按文件和任务 checkbox 汇总：

- Draft Changes；
- Active Changes；
- Completed Changes；
- Specs 和 Requirement 数量；
- 任务进度。

证据：\`src/core/view.ts:8-80\`、\`src/core/view.ts:82-155\`

Telemetry 只记录匿名命令和版本，用于产品使用统计，不参与业务工作流判断：\`README.md:243-251\`

---

## 8. 如果没有 OpenSpec，会缺失什么

先给准确结论：\*\*没有 OpenSpec，不会失去 CodeBuddy / Claude Code 的代码生成能力；失去的是围绕这项能力的一套规格化、持久化、可计算、跨工具的工程控制机制。\*\*

| 能力 | 没有 OpenSpec 会怎样 | Harness / 人工能否替代 | 替代成本 |
|---|---|---|---|
| 自然语言理解、写代码 | 仍然存在 | LLM 原生就能做 | 无 |
| 代码搜索、文件编辑、跑测试 | 仍然存在 | Harness 原生就能做 | 无 |
| 统一的 proposal/spec/design/tasks | 不再自动形成统一结构 | 可以手写模板和规则 | 中等，容易团队漂移 |
| 制品依赖图 | 不再有统一 ready/blocked 状态 | 可由 Agent 自觉按步骤执行 | 高，长任务容易跳步 |
| 持久化变更状态 | 更依赖聊天历史和零散文档 | 可用 Issue、TAPD、Markdown 替代 | 中高，需要自行打通 |
| 结构化 \`status --json\` | Agent 需要自己扫描和推断 | 可以写脚本替代 | 中等 |
| 结构化 \`instructions --json\` | 模板、路径、规则需要人工拼上下文 | 可以维护自定义 Skills | 高，多个 Harness 重复维护 |
| 跨 Harness 适配 | 每个工具单独维护命令文件 | 可以分别维护 | 高，容易版本漂移 |
| 规格语法与 delta 冲突校验 | 错误更晚暴露 | 可写 lint / CI | 中高 |
| 主规格 + delta + archive | 当前状态和变更历史容易混在一起 | Git + 文档流程可替代 | 高，需要团队纪律和脚本 |
| 多仓库 Store / Reference | 产品规格复用需另建机制 | Git submodule、文档平台可替代 | 高，语义和 Agent 接入需自建 |
| Agent 驱动 verify 工作流 | 不再有统一核对清单 | Code Review / 自定义 Skill 可替代 | 中等 |

### 8.1 最明显的缺失：上下文无法稳定跨会话

一次会话里 LLM 可能记得用户为什么要改；三天后换一个 Agent，它只看到当前代码，很难知道：

- 原始目标；
- 哪些范围明确不改；
- 哪些技术决策经过权衡；
- 还有哪些任务没做；
- 当前代码是目标状态还是中间状态。

OpenSpec 用文件补上这部分。

### 8.2 第二个缺失：模型容易“直接冲进代码”

Harness 的默认强项是快速搜索和修改代码，却未必先完成：

\`\`\`text
需求边界 → 可测试行为 → 技术设计 → 实施清单
\`\`\`

OpenSpec 的依赖图和 Workflow Prompt 将这条链显式化。

### 8.3 第三个缺失：不同 Harness 的工作方式会分叉

没有 adapter 层时：

- Claude Code 一套 \`.claude/commands\`；
- CodeBuddy 一套 \`.codebuddy/commands\`；
- Cursor 又是另一种命名；
- 每次流程修改要同步多份。

OpenSpec 将工作流源模板集中维护，再生成各工具文件。

### 8.4 第四个缺失：规格不会自动演进成“当前真相”

普通设计文档经常只记录“当时想怎么改”，完成后没人把它同步回系统当前规格。

OpenSpec 的 delta + sync/archive 模型试图保证：

\`\`\`text
变更前主规格 + 本次 delta = 变更后主规格
\`\`\`

### 8.5 不要夸大缺失

如果团队已经具备以下全部能力：

- 严格 Issue / RFC 模板；
- ADR；
- 测试用例管理；
- 自定义 Agent Skills；
- CI 规格 lint；
- 变更归档；
- 跨仓库规格服务；

那么没有 OpenSpec 也能工作。OpenSpec 的价值是\*\*把这些机制以 AI Agent 可消费的本地协议整合起来\*\*，而不是宣称传统工程流程都不存在。

---

## 9. Critical Problems（关键问题与边界）

## 9.1 \`done\` 是文件存在，不是语义完成

这是当前架构最大的认知风险。

只要 \`tasks.md\` 存在，artifact graph 就可能认为 tasks artifact 已完成；文件为空、内容错误、依赖文档缺失，并不会自动被 status 识别为语义失败。

源码证据：\`src/core/artifact-graph/outputs.ts:13-41\`

模板自身也承认 status 是 file-existence only：\`src/core/templates/workflows/propose.ts:81-88\`

影响：

- \`status\` 适合作为流程导航；
- 不应被当作质量验收结果；
- apply 前还需要 \`validate\` 和人工/Agent 复核内容。

## 9.2 很多 Guardrail 是 Prompt Contract，不是程序强制

例如 apply 模板要求 Agent：

- 读取所有 context files；
- 一个任务完成后立即勾选；
- 不清楚时暂停；
- 遇到设计问题更新 artifacts。

但 OpenSpec CLI 无法证明 Agent 真的执行了这些动作。

模板明确说明部分 context/guidance 是 “prompt-level behavior contracts, not enforceable checks”：\`src/core/templates/workflows/apply-change.ts:59-72\`

因此实际可靠性仍受以下因素影响：

- Harness 是否完整加载 Skill；
- LLM 是否遵循长提示；
- 工具权限是否足够；
- 用户是否中途打断；
- tasks checkbox 是否被如实更新。

## 9.3 Verify 主要是 Agent 启发式检查，不是形式化证明

\`verify\` 要求 LLM 用关键词搜索代码、推断 Requirement 是否实现、检查测试是否覆盖 Scenario。

模板自己写明：使用 keyword search、reasonable inference，不要求 perfect certainty：\`src/core/templates/workflows/verify-change.ts:154-160\`

所以 verify 的价值是统一审查框架，而不是证明程序正确。

## 9.4 CLI Archive 与 Agent Archive 是两条路径

当前既有 \`openspec archive\` 的 TypeScript 实现，又有 \`/opsx:archive\` 的 Agent 工作流模板。

二者的检查、同步和最终移动责任并非完全同一套代码。

风险：

- 行为随入口不同而略有差异；
- 文档必须明确用户走的是 CLI archive 还是 Agent archive；
- 不能把 Agent 模板里的步骤误认为 CLI 内部已经程序化执行。

## 9.5 Agent-driven 智能合并更灵活，也更依赖模型

\`/opsx:sync\` 能理解“只增加一个 Scenario，不覆盖其他 Scenario”这类局部语义，优于简单文本替换。

代价是：

- 合并结果受 LLM 判断影响；
- 需要合并后重新比较主规格；
- Prompt 明确要求幂等，但幂等不是类型系统强制的。

证据：\`src/core/templates/workflows/sync-specs.ts:90-125\`、\`src/core/templates/workflows/sync-specs.ts:188-224\`

## 9.6 Adapter 是文件协议兼容，不是深度 Runtime 集成

CodeBuddy / Claude Code 若改变：

- 扫描目录；
- frontmatter 字段；
- 命令命名规则；
- Skill 标准；

OpenSpec adapter 也必须跟着升级。

这就是为什么 adapter 测试和 \`openspec update\` 很重要。

---

## 10. Worth Learning（值得学习的设计）

## 10.1 把 LLM 不擅长的状态管理交给确定性程序

LLM 擅长语义，不擅长长期、精确、可重复的状态记忆。

OpenSpec 让：

- 文件系统保存事实；
- Schema 保存流程；
- Graph 计算状态；
- CLI 输出 JSON；
- LLM 只处理语义。

这是合理的人机分工。

## 10.2 统一内容，适配不同 Harness 方言

工作流模板只有一套，adapter 只关心路径和 frontmatter。这个架构避免了为 20 多个 AI 工具复制 20 多份业务逻辑。

## 10.3 用 delta 表达意图，而不是反复复制全量规格

ADDED / MODIFIED / REMOVED / RENAMED 让一次 change 的意图清晰，也便于 archive 后形成主规格。

## 10.4 Schema 可覆盖且有优先级

Schema 查找优先级：

\`\`\`text
项目本地 schema > 用户级 schema > 包内置 schema
\`\`\`

证据：\`src/core/artifact-graph/resolver.ts:77-118\`

这允许团队定制自己的制品图，而不是被固定的 proposal/spec/design/tasks 锁死。

## 10.5 Store 将“产品规格”和“代码实现”分层

被引用 Store 可作为只读上游约束，低层设计和代码仍落各自仓库。这比简单复制产品文档更适合多服务协作。

---

## 11. Practical Takeaways（实际使用建议）

## 11.1 在 CodeBuddy 中把 OpenSpec 当“规划控制平面”

推荐职责划分：

\`\`\`text
OpenSpec：需求、规格、设计、任务、状态、归档
CodeBuddy：代码检索、编辑、命令执行、测试、Git、交互
LLM：语义分析、技术决策、实现和核对
CI：编译、测试、lint、安全扫描、真实质量门禁
\`\`\`

不要把 \`status=done\` 当测试通过，也不要用 \`validate\` 代替业务测试。

## 11.2 关键任务使用完整闭环

\`\`\`mermaid
flowchart LR
    E\[explore<br/>澄清问题\] --> P\[propose/new<br/>建立 change\]
    P --> A1\[proposal<br/>范围与原因\]
    A1 --> A2\[specs<br/>可测试行为\]
    A1 --> A3\[design<br/>技术决策\]
    A2 --> A4\[tasks<br/>实施清单\]
    A3 --> A4
    A4 --> I\[apply<br/>Harness + LLM 改代码\]
    I --> V\[validate + tests + verify\]
    V --> S\[sync<br/>delta 合并主规格\]
    S --> R\[archive<br/>保存历史\]
\`\`\`

## 11.3 给 Schema、Validate、Test 三层不同定位

- \*\*Schema / Graph\*\*：保证工作顺序和文件形状；
- \*\*OpenSpec Validate\*\*：保证规格语法与 delta 一致性；
- \*\*项目 Test / CI\*\*：保证实现行为；
- \*\*Agent Verify / Code Review\*\*：核对规格、设计、任务、代码之间是否一致。

四层不能互相替代。

## 11.4 团队定制优先放在配置和 Schema，不要直接复制模板

优先使用：

- \`openspec/config.yaml\` 的 \`context\`；
- artifact \`rules\`；
- apply/archive operation guidance；
- 项目本地 Schema；
- Workflow profile。

这样 \`openspec update\` 后仍能保留团队约束。

## 11.5 将验证结果落到 CI

为了弥补 Prompt Guardrail 不可强制的问题，建议 CI 至少执行：

\`\`\`text
openspec validate --all --strict
项目编译
单元测试 / 集成测试
lint / typecheck
\`\`\`

如果团队要求“未完成任务不能合并”，还需额外编写确定性检查，不能只依赖 \`/opsx:verify\` 的 LLM 判断。

---

## 12. 最终判断

### OpenSpec 是什么

\*\*术语版：\*\*OpenSpec 是一个面向 AI Coding Agent 的、文件系统驱动的规格生命周期与工作流协议实现。它由 Harness adapter、可配置 Artifact DAG、结构化 instructions/status API、Markdown parser/validator、delta spec 合并、archive 和 Store 组成。

\*\*说白了就是：\*\*它不替你写代码，它负责让“谁来写、先看什么、按什么规格写、做到哪了、最后怎么验收和留档”不再全靠聊天记忆和模型自觉。

### 它最重要的源码价值

不是某一个 Markdown 模板，而是这五件事组合在一起：

1. \*\*持久规格状态\*\*：change artifacts 和 main specs；
2. \*\*确定性流程状态\*\*：Schema + ArtifactGraph；
3. \*\*LLM 任务接口\*\*：\`status --json\` + \`instructions --json\`；
4. \*\*Harness 可移植性\*\*：CodeBuddy / Claude 等 adapter；
5. \*\*生命周期闭环\*\*：validate、sync、archive、Store。

### 没有它会怎样

Harness + LLM 仍然能完成单次编码任务，但需要团队自行补齐：

- 规格模板；
- 依赖顺序；
- 状态追踪；
- 跨会话记忆；
- 规格校验；
- 主规格演进；
- 变更归档；
- 多 Harness 分发；
- 跨仓库规格复用。

所以 OpenSpec 的真实价值不是“让 AI 从不能写代码变成能写代码”，而是：

> \*\*让原本能写代码但容易忘、容易跳步、容易漂移的 Agent，进入一条有施工图、有工序、有状态、有验收、有档案的工程流水线。\*\*

---

## 13. 证据完整度与反向验证

### 已验证

- 检查了 CLI 入口、命令注册和实际 handler。
- 检查了 CodeBuddy 与 Claude 的专用 adapter 及其测试。
- 检查了 init/update 的实际文件生成路径。
- 检查了默认 Schema、ArtifactGraph、状态检测和集成测试。
- 检查了 instructions、project config、validator、archive、Store 和 view。
- 搜索了源码与依赖，未发现 LLM Provider SDK 或模型 API 调用。
- 反向搜索了 \`.command('apply')\`，没有命中，确认 apply 是 Harness workflow 而不是 CLI 子命令。

### 已排除

- 排除了“OpenSpec 自己调用 LLM”的理解。
- 排除了“\`status=done\` 等于内容正确”的理解。
- 排除了“\`validate\` 会验证业务代码实现”的理解。
- 排除了“CodeBuddy 集成是私有 API / MCP 集成”的理解。
- 排除了“没有 OpenSpec，CodeBuddy / Claude Code 就不能写代码”的理解。

### 尚存边界

- 本次尝试执行三组 Vitest 时，本地环境没有可直接调用的 \`vitest\` 二进制，因此没有把“测试当前实际运行通过”作为结论；文中测试引用属于对测试源码断言的静态核实。
- CodeBuddy adapter 的源码目标名称是 \`CodeBuddy Code (CLI)\`；CodeBuddy IDE 对 \`.codebuddy/commands\` / \`.codebuddy/skills\` 的具体 UI 展示取决于当前产品版本。本文只将仓库已证明的文件协议作为确定事实。

---

## 14. 补充：从用户实际痛点看 OpenSpec 的本质

如果不从源码模块，而是从 AI 开发过程中的实际问题出发，OpenSpec 的本质可以进一步化简为：

> \*\*让 AI 分步骤分析，把每一步的有效结论结构化总结，并持久化到项目文件。\*\*

也就是：

\\\[
\\text{OpenSpec}
=
\\text{分步骤}
+
\\text{结构化总结}
+
\\text{文件持久化}
\\\]

这里不需要再把“每一步有明确产物”“结构化持久化”包装成额外的独立概念：

- “每一步有明确产物”，本质上就是\*\*分步骤 + 持久化\*\*；
- “结构化持久化”，本质上就是要求 AI \*\*按固定结构总结 + 写入文件\*\*。

### 14.1 它解决的真实问题

传统 AI 开发中，需求分析、技术方案、改动点、影响范围、测试建议和中途修正通常都停留在聊天记录里。这会造成：

1. 一次完成的内容太多，分析、设计、编码和验证容易混在一起；
2. 会话中断后，新的 AI 难以判断此前已经确认了什么、做到哪里；
3. 旧方案、用户纠正和新方案不断叠加，容易相互干扰；
4. 为恢复任务而反复加载历史聊天，导致上下文越来越大；
5. 有价值的分析没有形成正式文档，任务结束后也难以复查和交接。

OpenSpec 的处理方式很直接：

\`\`\`text
AI 分析需求
    ↓
结构化总结到 proposal.md
    ↓
AI 分析目标行为和边界
    ↓
结构化总结到 specs/\*\*/\*.md
    ↓
AI 分析技术方案和影响范围
    ↓
结构化总结到 design.md
    ↓
AI 拆解实际改动
    ↓
结构化总结到 tasks.md
    ↓
AI 按 tasks.md 修改代码并更新进度
\`\`\`

这样，聊天只承担当前一轮思考；文件承担跨轮次、跨会话的长期记忆。

### 14.2 为什么它能降低上下文爆炸

没有 OpenSpec 时，恢复任务通常需要重新加载：

\`\`\`text
原始需求
+ 第一轮分析
+ 用户纠正
+ 被否决的旧方案
+ 新方案
+ 已完成改动
+ 中途发现的问题
+ 当前剩余工作
\`\`\`

使用 OpenSpec 后，新会话主要读取当前有效状态：

\`\`\`text
proposal.md  → 当前需求、目标和目标
specs/       → 当前有效的行为约束和验收场景
design.md    → 当前采用的技术方案、影响范围和风险
tasks.md     → 已完成和未完成的实际任务
Git diff     → 当前真实代码改动
测试结果      → 当前验证证据
\`\`\`

旧聊天不必反复进入上下文，因为其中仍然有效的结论已经被提炼到文件中；被否决的结论也不会继续与当前方案平铺混杂。

说白了就是：

> \*\*聊天负责思考，文件负责记忆；下一次聊天不需要记住上一次聊天，只需要重新读取当前文件。\*\*

### 14.3 OpenSpec 相比“自己让 AI 写几个文档”多做了什么

用户完全可以不用 OpenSpec，直接要求 AI：

\`\`\`text
分析完写 requirement-analysis.md
方案确认后写 design.md
改代码前写 tasks.md
验证后写 verification-report.md
\`\`\`

这已经实现了 OpenSpec 最核心的思想。

OpenSpec 源码额外做的，是把这套约定标准化和自动化：

- 用 \`schema.yaml\` 规定分哪些步骤以及步骤依赖；
- 用模板规定每一步按什么结构总结；
- 用 ArtifactGraph 判断下一步是 \`ready\` 还是 \`blocked\`；
- 用 \`status\` 从磁盘重新计算当前进度；
- 用 \`instructions\` 给 AI 返回当前步骤的模板、路径、依赖和规则；
- 用 \`validate\` 检查规格结构和 delta 冲突；
- 用 \`sync\`、\`archive\` 维护当前规格和变更历史；
- 用 adapter 把同一套流程分发到 CodeBuddy、Claude Code 等 Harness。

所以应当区分：

- \*\*概念本质\*\*：AI 分步骤分析、结构化总结并持久化；
- \*\*OpenSpec 的工程实现\*\*：把这套做法封装成统一目录、模板、依赖图、状态查询、校验、归档和 Harness 适配。

### 14.4 最终化简

\*\*术语版：\*\*OpenSpec 将 AI 会话中的临时推理结果转换为结构化、版本化的工程制品，使开发任务能够跨会话恢复、按步骤推进并保留变更历史。

\*\*说白了就是：\*\*

> \*\*OpenSpec 就是让 AI 不要把需求分析、方案、影响范围、改动点和测试建议只说在聊天里，而是分阶段总结成文件；以后中断了就读文件继续，避免所有上下文一直堆在对话里。\*\*

因此，对其本质最简洁且准确的概括是：

> \*\*OpenSpec = 分步骤 + 结构化总结 + 持久化。\*\*