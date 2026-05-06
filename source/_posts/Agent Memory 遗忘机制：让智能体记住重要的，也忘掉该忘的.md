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


在讨论 Agent Memory 时，我们往往更关注“如何记住”。

如何保存用户偏好？
如何记住历史对话？
如何复用过往经验？
如何让 Agent 在多次会话之间保持连续性？

但长期运行的 Agent 还有另一个同样重要的问题：

> 如果一个 Agent 只会记忆，不会遗忘，它最终会被自己的历史污染。

记忆让 Agent 拥有连续性，遗忘则让这种连续性不会变成负担。

这篇文章想讨论的不是简单的 `delete memory`，而是 Agent Memory 系统里的一个核心能力：**遗忘机制**。

---

## 1. 为什么 Agent 需要遗忘？

一个普通聊天机器人没有长期记忆。每次对话结束后，它的状态基本就消失了。

但 Memory-Aware Agent 不一样。它会持续积累：

* 用户偏好；
* 历史对话；
* 任务上下文；
* 工具调用经验；
* 工作流模式；
* 外部知识；
* 用户修正；
* 个性化画像；
* 长期关系和状态。

这让 Agent 可以变得更“懂你”。
但问题也随之出现。

如果所有信息都被长期保存，Agent 很快会遇到几类问题。

### 1.1 过期记忆污染当前判断

例如用户曾经说：

```text
我最近在准备 App Store 审核。
```

这句话在当时很重要。
但几周之后，它可能已经不应该继续影响 Agent 的回答。

如果 Agent 一直记住这件事，并在后续所有产品讨论中都假设用户仍在准备审核，就会产生错误上下文。

这类记忆不是错的，只是**过期了**。

### 1.2 旧偏好和新偏好冲突

例如：

```text
用户以前说：我喜欢黑白极简风。
用户后来又说：我现在更喜欢复古色彩和松弛感穿搭。
```

如果 memory 系统只是不断追加记录，那么检索时可能同时取出两条偏好：

```text
用户喜欢黑白极简风。
用户喜欢复古色彩。
```

Agent 不知道哪个是当前有效偏好，就会出现混乱。

这类问题不是“记忆不够”，而是**记忆缺少版本和失效机制**。

### 1.3 临时任务被误存为长期事实

用户说：

```text
这周我要集中处理 Google Cloud 试用额度的问题。
```

这应该是一个短期任务上下文。
但如果系统把它写入长期用户画像，几个月后仍然召回，就会变得很奇怪。

Agent Memory 里最常见的污染之一，就是把 temporary context 当成 long-term memory。

### 1.4 低价值记忆越来越多

如果系统把每一句话都存进去，memory store 会迅速膨胀。

例如：

```text
用户今天问了一个 Docker 命令。
用户今天打开了某个网页。
用户今天提到“继续”。
用户今天说“这个可以”。
```

这些信息对当前任务有用，但不一定值得长期保存。

长期记忆不是聊天记录备份。
它应该保存对未来有复用价值的信息。

### 1.5 错误记忆会被不断强化

更危险的是错误记忆。

例如用户说：

```text
我可能对青霉素过敏。
```

如果 Agent 错误总结成：

```text
用户对青霉素过敏。
```

这就把不确定信息变成了确定事实。

后续 Agent 每次召回这条记忆，都会继续强化这个错误。

所以遗忘机制不只是为了节省存储空间，更是为了避免错误上下文长期污染 Agent 的判断。

---

## 2. 遗忘不是删除

很多人一说到遗忘，就会想到：

```text
delete memory
```

但在 Agent Memory 系统中，遗忘不应该只等同于物理删除。

更准确地说，遗忘是一套 memory lifecycle 管理机制。

一条记忆从产生到退出活跃上下文，大致会经历：

```text
写入 → 去重 → 合并 → 强化 → 衰减 → 归档 → 删除
```

也就是：

```text
Write
  ↓
Deduplicate
  ↓
Consolidate
  ↓
Reinforce
  ↓
Decay
  ↓
Archive
  ↓
Delete
```

其中真正的 hard delete 只是一种极端情况。

更多时候，系统需要的是：

```text
不再主动召回
降低排序权重
标记为过期
被新记忆替代
转入历史归档
仅在特定场景下可访问
```

所以可以把遗忘分成三类。

---

## 3. 三种遗忘：Soft Forget、Archive、Hard Delete

### 3.1 Soft Forget：软遗忘

软遗忘不是删除，而是降低记忆对当前上下文的影响。

例如：

```text
用户去年说喜欢黑白极简风。
用户最近频繁讨论复古风穿搭。
```

系统不一定要删除“黑白极简风”这条记忆。
但在生成当前穿搭建议时，它的权重应该下降。

软遗忘可以通过以下方式实现：

```text
降低 recall score
降低 freshness score
标记为 inactive
减少进入 prompt 的概率
只在历史查询时召回
```

软遗忘适合大部分普通记忆。

### 3.2 Archive：归档

归档适合那些不再高频使用，但仍有历史价值的记忆。

例如：

```text
用户 2025 年做过一个 Spec Center 项目。
用户曾经调研过 OpenSearch hybrid search。
用户之前设计过 D5 Works 审核系统。
```

这些信息可能不应该每次都进入上下文，但在用户重新讨论相关主题时，它们仍然有价值。

归档不是忘掉，而是从“活跃记忆”转为“低频记忆”。

### 3.3 Hard Delete：硬删除

硬删除是真正删除数据。

它适合几类场景：

```text
用户明确要求删除
敏感信息不应保存
错误记忆无法修正
合规要求删除
测试数据清理
```

例如用户说：

```text
忘掉我刚才说的地址。
```

这类场景就不应该只是降低权重，而应该真正删除。

---

## 4. 什么样的记忆应该被遗忘？

Agent Memory 里不是所有内容都应该长期保存。
可以按类型来看。

### 4.1 临时上下文

例如：

```text
我今天要修一个 Docker compose 报错。
这周我要准备 App Store 审核。
我正在看这篇论文。
```

这些信息只对当前 session 或短期任务有用。

推荐策略：

```text
绑定 run_id / session_id
设置 TTL
任务结束后清理
不进入长期用户画像
```

### 4.2 过期偏好

例如：

```text
用户以前喜欢 A 风格。
用户现在喜欢 B 风格。
```

偏好不是静态事实。
尤其在穿搭、音乐、饮食、内容推荐、创作工具里，偏好会不断变化。

推荐策略：

```text
保留新偏好
旧偏好标记为 superseded
检索时优先 current preference
保留历史偏好用于趋势分析
```

### 4.3 错误记忆

错误记忆应该被修正或删除。

例如：

```text
Agent 记住：用户正在做 Android App。
实际情况：用户正在做 iOS App。
```

推荐策略：

```text
用户纠正时触发 update
旧记忆标记为 corrected
必要时 hard delete
保存 correction source
```

### 4.4 重复记忆

Agent Memory 很容易出现重复：

```text
用户喜欢极简风。
用户偏好极简设计。
用户倾向于简洁风格。
```

这些可能语义相近，但被写成多条 memory。

推荐策略：

```text
embedding similarity 去重
LLM 判断是否可合并
合并 access_count
保留 source_refs
```

### 4.5 低置信记忆

用户有时候表达的是猜测：

```text
我可能会用 CoreML。
我也许想做一个 AI 衣橱 App。
我不确定是不是 OpenSearch 的问题。
```

这类内容不能直接变成确定记忆。

推荐策略：

```text
标记 confidence_score
低置信内容不进长期记忆
或仅作为 temporary hypothesis
```

### 4.6 敏感信息

例如：

```text
身份证号
详细住址
健康信息
财务信息
账号密钥
隐私关系
```

这类信息不应该默认写入 memory。
即使业务上确实需要，也必须有明确权限、用途和删除机制。

推荐策略：

```text
默认不存
存储前确认
加密
设置短 TTL
用户可删除
审计访问
```

---

## 5. 记忆生命周期设计

一个成熟的 memory 系统，不应该只提供：

```text
add_memory()
search_memory()
delete_memory()
```

还应该考虑完整生命周期。

### 5.1 Write：写入

写入阶段要判断：

```text
这句话是否值得记住？
属于哪种 memory type？
是长期事实还是临时上下文？
置信度是多少？
是否包含敏感信息？
是否需要过期时间？
```

例如：

```text
用户说：我现在主要用 Java 和 Spring Boot 做后端。
```

这适合写入长期画像。

但：

```text
用户说：我今天下午要调一个 Docker 问题。
```

更适合写入短期任务上下文。

### 5.2 Deduplicate：去重

写入前或写入后，需要检查是否已有相似记忆。

例如已有：

```text
用户主要使用 Java。
```

新记忆：

```text
用户常用 Spring Boot 和 MyBatis。
```

这不是完全重复，而是可以合并增强。

但如果新记忆是：

```text
用户主要写 Java 后端。
```

它可能就只是已有记忆的改写，不一定需要新增一条。

### 5.3 Consolidate：合并

多条碎片记忆可以合并成更稳定的结构化记忆。

例如：

```text
用户经常问 Spring Boot。
用户经常问 MyBatis。
用户经常问 RabbitMQ。
用户经常问 OpenSearch。
```

可以合并成：

```text
用户主要关注 Java 后端、消息队列和搜索系统工程。
```

合并的好处是减少 memory 噪声。
风险是摘要或归纳可能出错。

所以合并后的记忆最好保留来源：

```text
source_message_ids
source_memory_ids
last_verified_at
confidence_score
```

### 5.4 Reinforce：强化

如果某条记忆被多次使用，而且持续被验证，它应该被强化。

例如：

```text
用户多次讨论 D5 Works。
用户多次提到 OpenSearch 2.19。
用户多次询问 iOS App 和 AI 衣橱。
```

这说明这些记忆对未来对话有较高价值。

可以通过以下字段强化：

```text
access_count + 1
last_accessed_at = now
importance_score 上升
confidence_score 上升
```

### 5.5 Decay：衰减

随着时间推移，部分记忆应该自然衰减。

例如：

```text
用户上个月正在准备某次发布。
用户去年对某个框架感兴趣。
用户曾经临时处理某个部署问题。
```

这些信息不是立刻删除，而是逐渐降低影响力。

一个简单的衰减思路：

```text
时间越久，freshness_score 越低
长期稳定偏好衰减慢
临时任务衰减快
被反复访问的记忆衰减慢
```

### 5.6 Archive：归档

当一条记忆长时间未被访问，但仍可能有历史价值，可以转入归档。

例如：

```text
用户曾经调研 AWS Image Similarity Solution Kit。
```

这不需要频繁进入上下文，但当用户再次讨论图像相似度时，它仍然有价值。

### 5.7 Delete：删除

删除应该谨慎，但必须存在。

常见触发条件：

```text
用户主动要求忘记
信息被确认错误
信息涉及敏感隐私
合规要求删除
测试数据过期
```

---

## 6. 一个可落地的 Memory Schema

如果要支持遗忘，memory 表不能只有：

```sql
id
user_id
content
embedding
created_at
```

至少应该多一些生命周期字段。

示例：

```sql
CREATE TABLE agent_memory (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    user_id VARCHAR(64) NOT NULL,
    agent_id VARCHAR(64) NULL,
    app_id VARCHAR(64) NULL,
    run_id VARCHAR(64) NULL,

    memory_type VARCHAR(32) NOT NULL,
    content TEXT NOT NULL,

    source_type VARCHAR(32) NULL,
    source_ref VARCHAR(255) NULL,

    importance_score DECIMAL(5,4) DEFAULT 0.5000,
    confidence_score DECIMAL(5,4) DEFAULT 0.8000,

    access_count INT DEFAULT 0,
    last_accessed_at DATETIME NULL,

    valid_from DATETIME NULL,
    valid_until DATETIME NULL,
    expires_at DATETIME NULL,

    superseded_by BIGINT NULL,
    status VARCHAR(32) DEFAULT 'active',

    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

几个关键字段：

| 字段                           | 作用                                                 |
| ---------------------------- | -------------------------------------------------- |
| `memory_type`                | 区分用户偏好、任务上下文、事实、流程、工具经验                            |
| `importance_score`           | 重要性                                                |
| `confidence_score`           | 可信度                                                |
| `access_count`               | 被召回次数                                              |
| `last_accessed_at`           | 最近使用时间                                             |
| `expires_at`                 | 过期时间                                               |
| `valid_from` / `valid_until` | 事实有效时间                                             |
| `superseded_by`              | 被哪条新记忆替代                                           |
| `status`                     | active / archived / expired / superseded / deleted |
| `run_id`                     | 单次会话或任务作用域                                         |

有了这些字段，系统才有能力判断：

```text
哪些记忆仍然有效？
哪些记忆已经过期？
哪些记忆被新事实替代？
哪些记忆只是当前任务上下文？
哪些记忆应该降低召回优先级？
```

---

## 7. 遗忘不只发生在删除阶段，也发生在检索阶段

很多 Agent 系统的 memory 流程是：

```text
用户输入
  ↓
向量检索 memory
  ↓
把 topK 塞进 prompt
  ↓
LLM 生成回答
```

这个流程有一个问题：
它只看语义相似度，不看记忆是否新鲜、可信、重要、有效。

例如用户问：

```text
帮我推荐今天的穿搭。
```

系统可能召回：

```text
用户去年喜欢黑白极简风。
用户上周说最近喜欢复古色彩。
用户昨天说今天要参加户外活动。
```

这三条都相关，但权重不应该一样。

更合理的 recall score 应该综合：

```text
语义相似度
重要性
可信度
新鲜度
访问频率
作用域
是否过期
是否被替代
```

一个简单公式可以是：

```text
recall_score =
    semantic_similarity * 0.40
  + importance_score   * 0.20
  + confidence_score   * 0.15
  + freshness_score    * 0.15
  + access_score       * 0.10
```

然后再加上过滤规则：

```text
status = active
expires_at is null or expires_at > now
superseded_by is null
scope matches current user / agent / run
```

这样，遗忘就不只是“删掉某条 memory”，而是在检索阶段让不合适的记忆自然退出当前上下文。

---

## 8. 不同 Memory 类型的遗忘规则

不同类型的 memory，遗忘策略不应该一样。

| Memory 类型             | 示例        | 遗忘策略              |
| --------------------- | --------- | ----------------- |
| Conversational Memory | 历史聊天记录    | 摘要、归档、按 thread 检索 |
| User Preference       | 用户喜欢某种风格  | 支持更新、替代、置信度管理     |
| Temporary Task Memory | 本周要处理某个问题 | TTL、任务结束清理        |
| Semantic Memory       | 稳定知识和事实   | 版本化、来源追踪          |
| Episodic Memory       | 某次经历或任务结果 | 按重要性保留            |
| Procedural Memory     | 成功工作流     | 根据成功率强化或淘汰        |
| Toolbox Memory        | 工具能力说明    | 随工具版本更新           |
| Sensitive Memory      | 隐私信息      | 默认不存，必要时硬删除       |
| Summary Memory        | 长对话摘要     | 保留来源，定期校验         |

这里最容易出错的是 User Preference。

用户偏好既不是绝对稳定，也不是完全临时。
它应该是“可演化”的。

例如：

```text
旧偏好：用户喜欢黑白极简风。
新偏好：用户最近更喜欢复古色彩和松弛感穿搭。
```

不应该简单变成两条并列记忆。
更好的处理是：

```text
旧偏好：
  content: 用户曾经喜欢黑白极简风
  status: superseded
  valid_until: 2026-05-06

新偏好：
  content: 用户最近更喜欢复古色彩和松弛感穿搭
  status: active
  valid_from: 2026-05-06
```

这就是语义层面的遗忘。

---

## 9. Mem0、Zep、Letta 对遗忘的启发

现在一些 Agent Memory 产品已经开始处理这个问题，但方向不完全一样。

### 9.1 Mem0：偏生命周期 API，遗忘需要应用层策略

Mem0 比较适合快速给 AI App 加长期记忆。

它的思路更偏：

```text
写入记忆
检索相关记忆
支持 update / delete
支持过期和作用域
通过应用层控制哪些该存、哪些该删
```

Mem0 的优势是接入简单，适合 MVP。
但如果要做复杂的“用户偏好变迁”“旧记忆自动失效”，通常还需要自己在业务层加一层 `MemoryPolicy`。

例如：

```text
用户说：我现在不喜欢极简风了，最近喜欢复古风。
```

应用层应该做：

```text
search related preference memories
  ↓
detect conflict
  ↓
mark old memory as superseded or delete
  ↓
add new active preference
```

也就是说，Mem0 提供基础设施，但“什么该忘”仍然需要产品策略决定。

### 9.2 Zep：偏时间感知和事实失效

Zep 的重要启发是：
有些记忆不应该被删除，而应该被标记为不再有效。

例如：

```text
用户以前住在上海。
用户现在搬到了杭州。
```

这不是简单删除“上海”，而是：

```text
用户曾经住在上海：历史事实
用户现在住在杭州：当前事实
```

这类 temporal memory 很适合处理：

```text
用户状态变化
偏好变化
关系变化
项目阶段变化
长期陪伴 Agent
CRM / 客服 Agent
```

它的核心价值是把“遗忘”变成“当前有效性管理”。

### 9.3 Letta：偏 Agent 自主管理记忆

Letta / MemGPT 的思路更像是让 Agent 自己管理上下文和记忆。

Agent 可以决定：

```text
哪些内容放入 core memory
哪些内容移到 archival memory
什么时候搜索长期记忆
什么时候编辑已有记忆
```

这种设计更接近真正的 Memory-Aware Agent。

它的启发是：

> 遗忘机制不一定只是后台定时任务，也可以成为 Agent Loop 的一部分。

也就是说，Agent 在执行任务时，可以主动判断：

```text
这条记忆已经不准确了。
这个偏好需要更新。
这段历史只需要归档。
这个临时任务已经结束，可以清理。
```

---

## 10. 一个 MemoryPolicy 层

如果自己实现 Agent Memory，我建议不要把遗忘逻辑散落在业务代码里。
可以抽象出一个 `MemoryPolicy` 层。

它负责：

```text
记忆分类
写入判断
TTL 分配
冲突检测
重复合并
召回排序
过期清理
用户删除请求
```

示意结构：

```text
User Input
  ↓
Memory Extractor
  ↓
MemoryPolicy
  ├── classify_memory_type
  ├── assign_importance
  ├── assign_confidence
  ├── assign_expiration
  ├── detect_duplicate
  ├── detect_conflict
  └── decide_add_update_archive_delete
  ↓
Memory Store
```

一个简单策略可以是：

```text
长期偏好：
  status = active
  expires_at = null
  支持被新偏好 supersede

临时任务：
  status = active
  expires_at = now + 7 days
  绑定 run_id 或 task_id

低置信内容：
  不写入长期 memory
  或 confidence_score < 0.5

敏感信息：
  默认拒绝写入
  必要时加密并设置短 TTL

重复记忆：
  合并，不新增

冲突记忆：
  旧记忆 superseded
  新记忆 active
```

---

## 11. Agent Loop 中的遗忘位置

遗忘机制不应该只靠离线任务。
它应该贯穿 Agent Loop。

一个完整流程可以是：

```text
用户输入
  ↓
检索相关 memory
  ↓
过滤 expired / superseded / low-confidence memory
  ↓
组装上下文
  ↓
LLM 推理
  ↓
工具调用
  ↓
生成回答
  ↓
抽取候选 memory
  ↓
MemoryPolicy 判断写入、更新、合并、过期
  ↓
保存 memory 变更
```

其中关键是最后一步。

不是每次对话结束都盲目 add memory，而是先判断：

```text
这条信息未来是否有用？
它是不是已有记忆的重复？
它是不是修正了旧记忆？
它是不是短期任务？
它有没有过期时间？
它是否应该进入长期用户画像？
```

---

## 12. 常见错误设计

### 12.1 把所有历史对话都当 memory

聊天记录是原始材料，不等于 memory。

长期 memory 应该是经过筛选、结构化、压缩和验证的信息。

### 12.2 只用向量相似度检索

向量相似度只能说明“语义相关”，不能说明“当前有效”。

过期记忆、错误记忆、被替代记忆，也可能和用户问题高度相关。

### 12.3 不区分短期和长期

这是最常见的问题。

当前任务上下文应该绑定 session / run / task。
长期用户画像应该保存稳定偏好和长期项目。

两者混在一起，Agent 会越来越混乱。

### 12.4 用户纠正时只新增，不更新

例如：

```text
用户：我不是做 Android 的，我是做 iOS 的。
```

如果系统只是新增：

```text
用户做 iOS。
```

但旧记忆：

```text
用户做 Android。
```

仍然存在，后续检索就可能冲突。

用户纠正应该触发 memory update / supersede / delete。

### 12.5 摘要当成绝对事实

summary memory 很有用，但也很危险。

因为摘要会丢失细节，也可能错误归纳。

所以摘要应该保留来源：

```text
source_message_ids
summary_level
confidence_score
last_verified_at
```

不要让摘要变成不可追溯的伪事实。

---

## 13. 总结

Agent Memory 的目标不是无限保存历史。

真正好的 memory 系统，需要持续判断：

```text
什么值得长期保留？
什么只是当前任务上下文？
什么已经过期？
什么被用户修正？
什么被新事实替代？
什么应该归档？
什么必须删除？
```

遗忘不是记忆系统的反面。
遗忘是记忆系统长期可用的前提。

一个没有记忆的 Agent，只能停留在单次对话里。
一个只会记忆、不会遗忘的 Agent，会被过期信息、错误事实和低价值历史拖垮。
一个真正成熟的 Memory-Aware Agent，必须同时具备两种能力：

```text
记住重要的。
忘掉该忘的。
```

这也是 Agent 从“会聊天”走向“能长期协作”的关键一步。

