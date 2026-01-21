# 项目总览：X For You 推荐系统

本文档对仓库进行较为细致的中文介绍，覆盖系统目标、整体架构、组件职责、数据流与关键设计点，便于快速理解代码组织与运行方式。

---

## 1. 背景与目标

该项目实现 X（原 Twitter）“For You” 信息流推荐系统的核心逻辑。系统通过融合两类内容来源并进行多阶段处理，输出最终排序结果：

- **关注内内容（In-Network）**：来自用户已关注账号的帖子。
- **关注外内容（Out-of-Network）**：从全量内容池中通过机器学习检索得到的候选。

系统目标包括：

- 提供高相关、高多样的内容排序
- 保持低延迟与稳定的在线服务
- 降低人工特征工程，尽量由模型端学习用户偏好

---

## 2. 系统架构概览

整体由四个核心模块组成：

1. **Home Mixer**：编排层，负责完整推荐流水线
2. **Thunder**：关注内候选内容的实时存储与检索
3. **Phoenix**：机器学习检索与排序模型（JAX 实现）
4. **Candidate Pipeline**：通用候选流水线框架

推荐请求的高层流程为：

1. 查询上下文补全（Query Hydration）
2. 候选召回（Sources）
3. 候选特征补全（Hydration）
4. 过滤与去重（Filtering）
5. 打分排序（Scoring）
6. 选取 Top K（Selection）
7. 后置过滤与副作用（Post-Selection / Side Effects）

---

## 3. 目录结构与职责

### 3.1 `home-mixer/`（编排层）

**定位：** 推荐请求的主入口与管控中心，负责各阶段组件的组织与执行。

**关键子模块：**

- `query_hydrators/`：拉取用户行为序列、用户画像等查询上下文。
- `sources/`：从 Thunder 与 Phoenix Retrieval 获取候选。
- `candidate_hydrators/`：补齐候选内容与作者信息、订阅状态、视频时长等。
- `filters/`：完成多类过滤（过期、去重、屏蔽词、已看等）。
- `scorers/`：模型评分、加权融合、多样性修正、关注外内容调节等。
- `selectors/`：按分数选出 Top K。
- `side_effects/`：缓存请求信息等异步副作用。

**服务入口：**

`home-mixer/main.rs` 和 `home-mixer/server.rs` 实现 gRPC 服务端，向外提供排序后的帖子列表。

---

### 3.2 `thunder/`（关注内实时候选）

**定位：** 关注内内容的实时存储与快速检索。

**主要能力：**

- 消费 Kafka 事件（发帖/删帖）
- 维护内存中按用户分桶的帖子缓存
- 支持超低延迟的关注内候选拉取
- 自动清理超期内容

**典型目录：**

- `thunder/kafka/`：Kafka 消费与事件监听
- `thunder/posts/`：帖子存储结构与索引维护
- `thunder/thunder_service.rs`：对外服务逻辑

---

### 3.3 `phoenix/`（机器学习检索与排序）

**定位：** 实现两阶段推荐模型（检索 + 排序），JAX 栈代码。

**两阶段流程：**

1. **Retrieval（双塔检索）**
   - 用户塔：编码用户特征与行为序列
   - 候选塔：编码全量内容
   - 用向量相似度检索 Top‑K 候选

2. **Ranking（Transformer 排序）**
   - 模型输入：用户上下文 + 候选列表
   - 预测多种用户行为概率（点赞、回复、转发等）
   - 通过加权组合输出最终排序分数

**关键特性：Candidate Isolation**

排序阶段采用特殊注意力掩码，防止候选之间相互注意，避免“候选批次依赖”，保证分数稳定可复现。

**运行方式：**

- `uv run run_retrieval.py`
- `uv run run_ranker.py`
- `uv run pytest ...` 运行测试

---

### 3.4 `candidate-pipeline/`（通用推荐流水线框架）

**定位：** 提供可复用、可组合的推荐流水线抽象。

**核心 Trait：**

- `Source`：候选召回
- `Hydrator`：候选补全
- `Filter`：候选过滤
- `Scorer`：候选打分
- `Selector`：候选选择
- `SideEffect`：副作用处理

**设计价值：**

- 业务逻辑与执行框架解耦
- 支持并行执行与可配置错误处理
- 便于扩展新的数据源与规则

---

## 4. 推荐流水线细节

### 4.1 Query Hydration

在进入推荐主流程前，系统先补齐用户上下文：

- 最近互动行为序列（点赞、转发、停留等）
- 用户关注列表、偏好配置

这些信息影响后续召回与模型输入。

---

### 4.2 Candidate Sources

候选来源主要有两类：

- **Thunder**：关注内内容
- **Phoenix Retrieval**：关注外的机器学习检索结果

两类候选会合并进入后续流程。

---

### 4.3 Candidate Hydration

补齐候选对象的必要信息，例如：

- 帖子核心元数据（文本、媒体、话题等）
- 作者信息（是否认证、社交关系等）
- 订阅/付费状态
- 视频时长（用于视频内容排序）

---

### 4.4 Filtering

过滤分为两阶段：预打分过滤与后置过滤。

**预打分过滤示例：**

- 去重（重复帖子、重复转推）
- 超期内容过滤
- 用户本人内容过滤
- 屏蔽作者 / 屏蔽关键词
- 已看 / 已服务内容过滤
- 订阅内容可见性过滤

**后置过滤示例：**

- 内容可见性过滤（删除、违规、敏感等）
- 会话级去重（同一会话内重复分支）

---

### 4.5 Scoring

排序阶段由多种 scorer 组成，核心思路为“模型预测 + 加权融合 + 多样性修正”：

1. **Phoenix Scorer**：使用 Transformer 预测用户多种行为的概率
2. **Weighted Scorer**：将多种行为概率加权合并为最终分数
3. **Author Diversity Scorer**：对重复作者进行惩罚，提升内容多样性
4. **OON Scorer**：调整关注外内容权重

最终分数形式类似：

```
Final Score = Σ (weight_i × P(action_i))
```

其中正向行为（点赞、转发、分享等）权重为正，负向行为（拉黑、举报等）权重为负。

---

### 4.6 Selection

基于最终分数排序，选取 Top K 候选进入最终列表。

---

### 4.7 Post-Selection & Side Effects

排序完成后还会进行：

- 最终可见性校验
- 去重与合规性检查
- 请求信息缓存、统计记录等副作用

---

## 5. 关键设计点

### 5.1 尽量消除手工特征

系统强调由模型从用户行为中学习相关性，减少人工规则和特征工程，降低系统复杂度。

---

### 5.2 Candidate Isolation

排序阶段通过注意力掩码限制候选之间相互注意，确保候选分数不受 batch 内容影响，提升可复现性与缓存能力。

---

### 5.3 多行为联合预测

模型同时预测多种用户行为，从而让最终分数更细粒度、更稳定。

---

### 5.4 可组合流水线

`candidate-pipeline` 抽象出统一接口，支持并行与模块化组合，使不同业务线能快速接入新特征与规则。

---

## 6. 运行与测试（Phoenix 模型部分）

依赖使用 `uv` 管理：

```
uv run run_retrieval.py
uv run run_ranker.py
uv run pytest test_recsys_model.py test_recsys_retrieval_model.py
```

---

## 7. 适合阅读的关键入口

如果希望从代码角度快速理解系统，可按以下入口阅读：

- `home-mixer/main.rs`：服务启动与编排入口
- `home-mixer/candidate_pipeline/`：流水线构建与执行
- `thunder/thunder_service.rs`：关注内候选服务
- `phoenix/run_ranker.py`：排序模型运行示例
- `candidate-pipeline/lib.rs`：通用流水线框架抽象


