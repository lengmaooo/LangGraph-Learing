# LangGraph概述

**是什么？**

LangGraph是基于LangChain构建的，面向智能体多轮交互/状态持久化/分支并行执行的图结构工作流框架。

LangGraph=LangChain+状态机+图编排

**AB法则：**

- before：LangChain是链是线性的，只适合做一次调用的场景，当遇到一些比较复杂的业务，需要循环、回退、分支执行这种的话就不行了。虽然LangChain可以用Agent智能体，让智能体来决定循环等操作，但是这个过程是黑盒的，人为无法干预，可能执行半天消耗一堆token却没执行到想要的结果。
- after：LangGraph是图形化的，通过定义节点和边，可以精准控制Agent的执行逻辑，包括条件分之、循环和并行执行等。

LangGraph的灵魂：

1. State(状态)
2. Nodes（节点)
3. Edges(边)
4. Graph(图)


# Graph API

## Graph API之Graph(图)

图的构建流程：

1. 初始化一个StateGraph实例
2. 加节点
3. 定义边，将所有的节点连接起来
4. 设置特殊节点，入口和出口（可选）
5. 编译图
6. 执行工作流

## Graph API之State(状态)

状态由两部分组成：Schema和Reducers。

**Schema**

Schema是图的模式，它定义了state长什么样。

Schema有三种：

1. state\_schema：图的完整内部状态，包含了所有节点可能读写的字段，必须指定，不能为空。
2. input\_schema：定义图接受什么输出，可选参数，默认等于state\_schema。
3. output\_schema：定义图返回什么输出，可选参数，默认等于state\_schema。

> State可以是TypedDict类型， 也可以是pydantic中的BaseModel类型

**Reducers**

Reducers是一种规约函数，规定了每次节点返回值各个字段的操作类型。

Reducer常用函数有以下几种：

1. default：未指定Reducer时使用覆盖更新
2. add\_messages：用于消息列表追加
3. operator.add：将元素追加到现有元素中， 支持列表、字符串、数值类型的追加
4. operator.mul：用于数值相乘
5. 自定义Reducer：支持用户自定义合并逻辑


## Graph API之Node(节点)

Node是图的一个节点，是LangGraph中的一个基本处理单元，可以做任何事，比如调用大模型、agent、调用一个函数等等。

Node除了简单的调用函数，还有两个特性：

1. 节点缓存Node Caching：节点的处理结果可以缓存起来，当下次走到这个节点时可以直接从缓存获取数据。（类似redis那一套规则，有就从缓存拿没有就执行出结果保存到缓存中）
2. 错误处理和重试机制： LangGraph提供了节点的重试机制，在添加节点的时候可以配置`retry_policy`，设定一些参数：最大重试次数，自定义重试判断机制等等


## Graph API之Edge(边)

Edge定义了不同节点之间的连接和执行顺序。

Edge边分为一下四种类型：

1. Normal Edges 普通边：固定了节点的先后执行顺序。
2. Conditional Edges 条件边：根据判断函数来决定节点走哪个节点。
3. Entry Point 入口：入口点上图启动时运行的第一个节点，可以指定执行的第一个节点。
4. Conditional Entry Point 条件入口边：针对入口点，可以用条件判断入口走不同的节点。


## Graph API之Send/Command/Runtime context

Graph特殊的几个API如下：

1. Send函数：特点是**多路并进，汇总规约**。适用于Map-Reduce模式(映射 - 归约)，也就是把一个大任务拆解成多个小任务处理完之后再合并结果。为了支持这种设计模式， LangGraph支持从条件边返回 Send 对象。Send接受两个参数：第一个是节点的名称，第二个是传递给该节点的状态。
2. Command函数：

   Command函数的特点是**节点路由+修改状态**。可以指定下一个节点，同时修改state。注意只能指定一个节点，而Send可以分发多个节点并行。
3. Runtime context：

   Runtime context就是**上下文**，类似于springboot的application.yaml，项目中的一些配置项啊什么的，不便于放进state中，可以放在上下文传入，其他节点可以随时引用。132


# 高级特性

## 流式处理(Streaming)

 流式处理核心就是让AI应用能“实时传输结果”，不用等整个流程跑完，体验更流畅。比如返回每一步的状态，自定义输出进度等等。

流式处理有以下5种模式：

1. values：每步结束后，输出完整的当前状态。
2. updates：每步结束后，只输出变化的部分。
3. messages：实时输出LLM的每一个字/词，还带相关信息。
4. custom：只输出你指定的消息。
5. debug：输出所有细节，方便调试。

以上五种模式，可以结合起来使用，则会进行多个模式的流式处理。


## 状态持久化(Persistence）

是什么？

状态持久化就是把状态给保存下来，无非是区分短期记忆还是长期记忆。短期记忆就是存在内存中；长期记忆就是存在数据库中。

有什么用？

保存了会话状态的话，下次来提问，可以接着上次的状态的结果继续提问。


## 时间回溯(Time-Travel)

是什么？

LangGraph的时间旅行，是一个允许你“回到对话的某个历史状态点，并从那里重新执行”的功能。

时间回溯就是可以在图的执行过程中，可以对状态进行一些设置，有以下作用；

1. 调试：想看 agent 在某个历史状态下会如何响应
2. 修复：发现某一步错误，可以回到那一步，重新走另一条路径
3. 探索分支：从同一个历史状态，分叉出多个可能的结果，做 what-if 实验
4. 人类反馈 (HITL)：如果用户拒绝了工具调用，可以退回到之前状态，重新走对话


## 子图(Subgraphs)

是什么？

LangGraph中可以把一个子图作为另一个父图某一个节点，也就是嵌套一张图。

> 在LangGraph中允许将一个完整的图作为另一个图的节点，适用于将复杂的任务拆解为多个专业智能体协同完成，每个子图都可以独立开发、测试并且可以复用。每个子图都可以拥有自己的私有数据，也可以与父图共享数据。


# A2A

## 多智能体架构 (Multi-Agent Architecture)

AI架构中，有两个协议正在重塑我们构建智能系统的方式，就是MCP和A2A。

MCP和A2A的区别：

- MCP：定义了大模型与各种工具交互的标准方式，本质上是工具访问的协议。简单来说，MCP能够让AI使用各种功能，就像程序员调用函数一样。
- A2A：定义了不同的智能体之间的交互，使得不同的AI系统能够像人类团队一样协同工作。

除了单智能体，还有常见的**多智能体架构**：

1. Network（网络型）：多个智能体平等存在，每个Agent可以和其他Agent通信，类似“去中心化网络”
2. <span data-type="text" style="background-color: var(--b3-font-background12);">Supervisor（监督者型）</span>：需要重点掌握，用的最多。一个主控Agent调度其他Agent，子Agent只负责各自领域。
3. Supervisor (as tools)（监督者作为工具）：一个 LLM 可以直接调用不同的“子智能体”当作工具，子智能体更像是 专业插件
4. Hierarchical（层级型）：多层次的监督者，顶层 Supervisor 分配任务给子 Supervisor，子 Supervisor 再调度子 Agent
5. Custom（自定义混合型）：根据业务需要自由组合（路由 + 协作 + 监督 + HITL）图结构灵活，不一定规则


## Agent Skills（智能体技能）

是什么？

skill是智能体的一种技能，着重点是提示词，当大模型有需要的时候会触发技能，提供提示词和执行一些提前预设好的指令。

Skills = 提示词的规范化+工程化规约落地实现，类似提示词版本的maven结构


**渐进式披露三层架构**

| 层级 | 组件名称    | 内容类型                 | 加载策略                     | Token 消耗权重  | 设计目的                            |
| ------ | ------------- | -------------------------- | ------------------------------ | ----------------- | ------------------------------------- |
| L1   | Metadata    | Skill 名称、描述、版本号 | Always-On (常驻)             | 极低 (\<1%)  | 供模型进行路由决策与意图识别        |
| L2   | Instruction | SKILL.md 正文规则        | On-Demand (命中后加载)       | 中等 (5-10%)    | 定义具体的业务处理逻辑与SOP         |
| L3   | Reference   | 外部文档、手册、规范     | Context-Triggered (条件触发) | 高 (可变)       | 提供必要的领域知识，用完即弃        |
| L4   | Script      | Python/Shell 脚本        | Execution-Only (仅执行)      | 零 (不读取代码) | 实现物理世界的副作用 (Side Effects) |
