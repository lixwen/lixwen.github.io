---
title: Agent 记忆系统：从 Stateless Agent 到 Memory-Aware Agent
date: 2026-05-06 12:00:00
tags:
  - AI
  - Agent
  - Memory
categories:
  - 学习笔记
---


过去几年，很多 AI 应用的重点都放在 **Prompt Engineering** 和 **Context Engineering** 上：如何把更好的提示词、更相关的上下文塞进一次 LLM 调用里。

但当我们开始构建真正的 Agent 时，会发现一个更基础的问题：

**一个 Agent 如果不能记住过去发生过什么，它就很难完成长期任务，也很难真正代表用户持续工作。**

这也是 DeepLearning.AI 课程《Agent Memory: Building Memory-Aware Agents》重点讨论的问题：如何从一个“单轮可用”的 Agent，演进到一个具备持久记忆、可跨会话延续、能随着交互不断改进的 **Memory-Aware Agent**。

---

## 1. 为什么 Agent 需要记忆？

一个典型的 AI Agent，通常具备几个能力：

* 接收输入，也就是感知环境；
* 使用工具，也就是执行动作；
* 基于 LLM 进行推理；
* 根据目标完成任务。

但如果没有记忆，这个 Agent 本质上仍然是 **stateless agent**。

例如：

用户第一次问：

> 帮我推荐附近的几家餐厅。

Agent 搜索后返回了几家餐厅。

用户接着问：

> 帮我预订第一家。

如果 Agent 没有记忆，它可能会回答：

> 请说明你想预订哪家餐厅。

问题在于，Agent 不知道“第一家”指的是刚才推荐列表中的第一家。它无法跨轮次理解上下文。

这类无记忆 Agent 的主要问题包括：

1. **无法完成长周期任务**
   比如持续几小时、几天，甚至几周的任务。如果每一步都不能记住之前做过什么，就很难稳定推进。

2. **跨会话上下文丢失**
   用户离开后再回来，Agent 不知道之前讨论过什么。

3. **无法适应用户偏好**
   用户告诉 Agent 的偏好、约束、习惯，如果不被存储，就无法在未来被使用。

4. **上下文成本高**
   为了维持连续性，只能不断把历史对话塞进 context window，导致 token 成本、延迟和上下文噪音增加。

因此，Agent 要从“能回答问题”走向“能持续协作”，记忆系统是基础设施，而不是附加功能。

---

## 2. Conversation History 不等于完整的 Agent Memory

很多人一开始会把“记忆”理解成“保存聊天记录”。

这确实是记忆的一种形式，课程里称为 **Conversational Memory**，也可以理解为一种 episodic memory。

一个最简单的 conversational memory unit 通常包括：

```text
thread_id
role
content
timestamp
metadata
summary_id
created_at
```

它可以帮助 Agent 知道：

* 用户之前说过什么；
* Assistant 之前回答过什么；
* 当前对话处于什么阶段；
* 某些历史消息是否已经被摘要压缩过。

但仅靠聊天记录是不够的。

因为 Agent 需要的不只是“过去说过什么”，还包括：

* 用户偏好；
* 任务执行步骤；
* 工具使用经验；
* 外部知识；
* 实体关系；
* 历史工作流；
* 长期积累的 domain knowledge；
* 已经压缩过的 summary memory。

所以，真正的 Agent Memory 应该是多类型、多结构、多检索方式的系统，而不是一个简单的 chat log 表。

---

## 3. Agent Memory 的几种核心类型

课程中将 Agent Memory 分为短期记忆和长期记忆。

### 3.1 短期记忆

短期记忆主要服务于当前会话或短时间内的任务执行。

#### Working Memory

可以理解为 LLM 当前 context window 里的内容，也包括 session-based memory。

它像一个临时草稿纸：

* 当前任务可用；
* 当前 session 结束后通常丢失；
* 不适合作为长期知识来源。

#### Semantic Cache

Semantic Cache 是基于向量相似度的缓存机制。

当用户的问题与之前的问题高度相似时，可以复用已有结果，减少重复调用 LLM 或工具。

例如：

```text
Query A: What is the weather in London tomorrow?
Query B: London weather tomorrow?
```

如果语义相似，可以直接从缓存中取回历史结果。

---

### 3.2 长期记忆

长期记忆是 Memory-Aware Agent 的重点。

课程中主要提到几类长期记忆。

#### Episodic Memory

记录具体发生过的事件或交互。

典型例子就是 conversational memory：

```text
用户什么时候问了什么？
Agent 当时怎么回答？
这次交互属于哪个 thread？
```

它适合回答：

* “我刚才问过什么？”
* “我们上次讨论到哪？”
* “这个任务之前做到哪一步了？”

#### Semantic Memory

记录稳定的知识。

例如：

* 领域知识库；
* 文档内容；
* arXiv 论文；
* 产品知识；
* 用户明确偏好；
* 外部检索结果。

这类记忆通常适合用向量数据库或支持 vector search 的数据库来存储。

#### Procedural Memory

记录“如何做事”。

例如，用户问：

> 获取当前天气。

Agent 可能需要执行以下流程：

```text
1. 获取用户位置
2. 调用天气 API
3. 传入经纬度
4. 获取天气结果
5. 整理成自然语言返回
```

这组步骤可以作为 workflow memory 保存下来。

下次遇到类似任务时，Agent 不需要重新推理完整流程，而是可以复用过去成功执行过的 workflow。

#### Toolbox Memory

课程中非常重要的一个设计是：**把工具也当作记忆来管理。**

当 Agent 拥有几十个、几百个工具时，如果把所有 tool definitions 都塞进 prompt，会导致：

* context bloat；
* tool selection degradation；
* token 成本上升；
* latency 增加；
* LLM 更容易选错工具。

更好的方式是：

1. 把工具名称、描述、参数结构转成文本；
2. 使用 embedding model 生成向量；
3. 存入数据库；
4. 用户发起请求时，先基于语义相似度检索最相关的工具；
5. 只把少量相关工具提供给 LLM。

这就是课程里的 **Toolbox Pattern**。

---

## 4. Memory Core：数据库是 Agent 记忆系统的核心

在 Agent 系统里，记忆可能存在于多个地方：

* LLM 的参数记忆；
* Embedding Model 的语义表示能力；
* 数据库中的外部记忆。

但对于可控、可更新、可检索的 Agent Memory 来说，最核心的是数据库。

课程中把数据库称为 **Agent Memory Core**。

它承担三类关键职责：

```text
1. Storage：保存记忆
2. Retrieval：检索记忆
3. Optimization：优化记忆组织和访问
```

也就是说，Agent Memory 不是简单把东西写进数据库，而是需要围绕数据库设计完整的 memory lifecycle。

---

## 5. Memory Manager：记忆系统的操作层

如果 Memory Core 是数据库，那么 **Memory Manager** 就是数据库之上的操作抽象层。

它负责统一管理不同 memory store 的读写操作。

例如：

```text
write_conversational_memory()
read_conversational_memory()

write_knowledge_base()
read_knowledge_base()

write_workflow()
read_workflow()

write_toolbox()
read_toolbox()

write_summary()
read_summary()

write_entity()
read_entity()
```

Memory Manager 的价值在于：

* 屏蔽底层数据库实现；
* 对不同记忆类型提供统一接口；
* 支持 create/read/update/delete；
* 支持向量检索、全文检索、混合检索；
* 支持 Agent 通过工具调用记忆操作；
* 支持 deterministic memory operation。

课程中使用 Oracle AI Database 作为基础设施层，并通过 LangChain 的 Oracle vector store integration 来构建不同 memory stores。

不过这个设计思想并不依赖 Oracle，本质上也可以迁移到：

* PostgreSQL + pgvector；
* OpenSearch / Elasticsearch；
* Milvus；
* Qdrant；
* MongoDB Atlas Vector Search；
* Redis Vector；
* 自研 SQL + 向量索引方案。

核心不是具体数据库，而是记忆类型、生命周期和操作抽象。

---

## 6. Memory Operation：确定性触发与 Agent 自主触发

课程中把 memory operation 分成两类。

### 6.1 Deterministic Memory Operation

由程序规则触发，Agent 无需主动决定。

例如：

```text
每次用户输入后，写入 conversational memory。
每次工具调用后，写入 tool log。
当 context window 超过 80% 时，自动触发 summarization。
任务结束后，写入 workflow memory。
```

这种操作适合保证系统稳定性。

### 6.2 Agent-Triggered Memory Operation

把记忆操作作为工具暴露给 Agent，让 Agent 自己决定什么时候调用。

例如：

```text
search_knowledge_base()
expand_summary()
summarize_conversation()
read_toolbox()
fetch_and_save_paper_to_kb()
```

这样 Agent 不只是“被动使用记忆”，而是知道自己有哪些 memory stores，并能主动读、写、检索、压缩或展开记忆。

这也是从 **Memory-Augmented Agent** 走向 **Memory-Aware Agent** 的关键区别。

---

## 7. Context Engineering 与 Memory Engineering 的关系

Context Engineering 的目标是：

> 在有限的 context window 中，放入最有价值、最高信噪比的信息。

它关注的是“当前这一次 LLM 调用应该看到什么”。

Memory Engineering 则更进一步：

> 设计、构建和维护 Agent 的长期记忆系统，使 Agent 能够存储、检索、更新、压缩、遗忘和复用信息。

二者关系可以这样理解：

```text
Memory Engineering 负责长期信息资产的管理。
Context Engineering 负责每次调用时的信息选择和组装。
```

没有 Memory Engineering，Context Engineering 只能在当前上下文里做取舍。

有了 Memory Engineering，Context Engineering 可以从外部 memory stores 中检索最相关的信息，再组装给 LLM。

---

## 8. Context Window Reduction：摘要与压缩

当上下文越来越长时，课程介绍了两种主要的 context window reduction 技术。

### 8.1 Context Summarization

把长上下文交给 LLM，总结成更短的版本。

目标是：

* 保留高价值信息；
* 去除重复内容；
* 保留关键事实和关系；
* 删除低信号细节；
* 控制最终长度。

但 summarization 有一个天然问题：

**它一定是有损的。**

只要做摘要，就一定会丢失部分细节。

所以摘要 prompt 很关键，需要明确要求：

```text
保留具体事实；
区分事实和待解决问题；
不要臆造；
不要添加原文不存在的信息；
尽量简洁但保持未来可用性。
```

### 8.2 Context Compaction

Context Compaction 不是简单总结，而是把原始上下文转移到数据库中，然后在当前 context 里只保留一个引用。

例如：

```text
Summary ID: summary_123
Description: 这段对话讨论了 MemGPT 论文搜索、保存和主要观点提取。
```

当 Agent 未来需要更完整的信息时，可以通过 `expand_summary(summary_id)` 从数据库中展开原始内容。

这比单纯摘要更可靠，因为它没有完全丢弃原始信息，只是把原始上下文 offload 到数据库中。

---

## 9. Memory-Aware Agent 的 Agent Loop

课程最后把前面的内容组合成完整的 Memory-Aware Agent。

一个典型 Agent Loop 可以抽象成：

```text
1. 接收用户输入
2. 读取相关记忆
3. 组装上下文
4. 检索相关工具
5. 调用 LLM
6. 判断是回答还是调用工具
7. 执行工具
8. 写入工具日志
9. 继续循环，直到完成任务
10. 写入最终对话、工作流、实体和摘要记忆
```

其中，记忆操作既发生在 Agent Loop 外部，也发生在 Agent Loop 内部。

### Agent Loop 外部的记忆操作

通常用于构建初始上下文：

```text
读取 conversational memory
读取 knowledge base memory
读取 workflow memory
读取 entity memory
读取 summary memory
检索相关 tools
写入用户当前 query
```

### Agent Loop 内部的记忆操作

通常与工具调用和上下文管理有关：

```text
调用 search 工具
调用 arXiv 工具
调用 expand_summary 工具
调用 summarize_conversation 工具
写入 tool log
进行 context offloading
```

### Agent Loop 结束后的记忆操作

用于沉淀本次执行经验：

```text
写入 assistant final answer
写入 workflow memory
写入 entity memory
写入 summary memory
```

这使 Agent 不只是完成当前任务，还会把执行过程转化为未来可复用的经验。

---

## 10. 一个典型案例：研究型 Agent

课程中演示了一个研究型 Agent。

用户第一次问：

> Can you get me the paper on MemGPT?

Agent 会：

```text
1. 构建上下文
2. 查询已有 knowledge base
3. 发现没有足够信息
4. 调用 arXiv search tool
5. 找到 MemGPT 论文
6. 返回论文信息
7. 写入 conversational memory 和 workflow memory
```

用户接着问：

> Can you save the content of the paper?

因为 Agent 有 conversational memory，它知道“the paper”指的是 MemGPT 论文。

于是它会调用保存论文内容的工具，把论文内容写入 knowledge base。

用户再问：

> Pull out some main takeaways from the paper.

此时 Agent 不需要重新搜索论文，因为论文已经进入 semantic memory。它可以直接从 knowledge base 中检索相关内容，然后总结主要观点。

这就是记忆系统带来的连续性。

更进一步，用户还可以问：

> What was my first question?

即使早期对话已经被摘要压缩，Agent 也可以通过 summary memory 找到 summary ID，再调用 expand summary，把原始上下文展开，回答用户最早的问题。

---

## 11. Memory-Aware Agent 的关键设计原则

基于课程内容，可以总结出几个设计原则。

### 1. 不要只保存聊天记录

聊天记录只是 episodic memory 的一种。

真正的 Agent Memory 至少应该区分：

```text
Conversational Memory
Semantic Memory
Workflow Memory
Toolbox Memory
Entity Memory
Summary Memory
Tool Log Memory
```

### 2. 数据库是记忆系统核心

LLM 本身不适合作为可控记忆系统。

真正可控的记忆应该在外部数据库中：

```text
可写入
可更新
可删除
可检索
可压缩
可审计
可迁移
```

### 3. 工具也应该被检索，而不是全部塞进 prompt

当工具数量增长时，Toolbox Pattern 非常重要。

它可以减少：

```text
context bloat
tool confusion
token cost
latency
tool selection error
```

### 4. 摘要不能替代原始数据

摘要是有损的。

更好的方式是：

```text
摘要 + summary_id + 原始数据可展开
```

这样既能节省 context，又不会完全丢失历史信息。

### 5. 记忆操作既要有规则触发，也要允许 Agent 自主触发

例如：

```text
确定性触发：
- 每轮写入 conversation
- 超过 80% context 自动 summarization
- 每次工具调用写入 tool log

Agent 触发：
- 主动查询知识库
- 主动展开 summary
- 主动保存新知识
- 主动检索 workflow
```

### 6. Context Window 应该分区

课程中强调，可以在 prompt 中明确告诉 LLM 当前上下文有哪些 memory partitions：

```text
## Conversational Memory
## Knowledge Base Memory
## Workflow Memory
## Entity Memory
## Summary Memory
## Available Tools
```

这种结构化上下文能帮助 LLM 理解不同信息的用途。

---

## 12. 对实际 Agent 系统的启发

如果要把这套方法应用到真实产品里，我认为可以抽象成下面这个架构：

```text
Application Layer
    ↓
Agent Harness
    ↓
Agent Loop
    ↓
Memory Manager
    ↓
Memory Stores
    ├── Conversational Memory
    ├── Semantic Memory
    ├── Workflow Memory
    ├── Toolbox Memory
    ├── Entity Memory
    ├── Summary Memory
    └── Tool Log Memory
    ↓
Database / Vector Store / Search Engine
```

对于不同业务场景，可以有不同重点。

### 个人助理 Agent

重点是：

```text
用户偏好
历史任务
日程上下文
长期目标
联系人关系
```

### 代码 Agent

重点是：

```text
项目结构
历史修改记录
常用命令
debug workflow
代码规范
review 结果
```

### 研究型 Agent

重点是：

```text
论文库
研究问题
引用关系
阅读笔记
实验流程
摘要记忆
```

### 游戏 Agent

重点是：

```text
玩家行为
NPC 关系
世界状态
剧情事件
角色长期记忆
```

### 电商 / 推荐 Agent

重点是：

```text
用户偏好
浏览历史
购买历史
商品知识库
推荐反馈
```

不同场景的 memory schema 会不同，但核心思路一致：

> 把 Agent 的历史交互、外部知识、任务经验和工具能力，结构化地存入外部记忆系统，再通过 Memory Manager 在合适的时机读写和组装。

---

## 13. 总结

Agent Memory 的本质，不是简单让模型“记住更多聊天记录”。

它是一套完整的系统工程：

```text
存储什么？
如何表示？
如何检索？
何时写入？
何时遗忘？
何时摘要？
何时展开？
如何让 Agent 知道自己有哪些记忆？
如何让记忆参与 Agent Loop？
```

从课程内容看，Agent 的演进路径可以概括为：

```text
Stateless Agent
    ↓
Conversational Memory Agent
    ↓
Memory-Augmented Agent
    ↓
Memory-Aware Agent
```

其中最大的变化是：

**记忆不再只是外挂数据库，而是 Agent 推理、工具选择、上下文管理和长期学习的一部分。**

未来真正有价值的 Agent，不只是能调用工具，也不只是能执行多步任务，而是能够在长期交互中形成连续性，积累经验，更新知识，并理解自己有哪些记忆能力。

这也是 Agent 从“会话式工具”走向“长期协作系统”的关键一步。

---

## 来源

本文基于 DeepLearning.AI 课程《Agent Memory: Building Memory-Aware Agents》的学习内容整理与总结。
课程地址：
[https://learn.deeplearning.ai/courses/agent-memory-building-memory-aware-agents](https://learn.deeplearning.ai/courses/agent-memory-building-memory-aware-agents)
