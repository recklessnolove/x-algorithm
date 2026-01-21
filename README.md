# X For You Feed Algorithm

<p>
  <strong>Language / 语言：</strong>
  <a href="#en">English</a> | <a href="#zh">中文</a>
</p>

<details open>
<summary><strong>English</strong></summary>

<a id="en"></a>

This repository contains the core recommendation system powering the "For You" feed on X. It combines in-network content (from accounts you follow) with out-of-network content (discovered through ML-based retrieval) and ranks everything using a Grok-based transformer model.

> **Note:** The transformer implementation is ported from the [Grok-1 open source release](https://github.com/xai-org/grok-1) by xAI, adapted for recommendation system use cases.

<h2 id="en-table-of-contents">Table of Contents</h2>

- [Overview](#en-overview)
- [System Architecture](#en-system-architecture)
- [Components](#en-components)
  - [Home Mixer](#en-home-mixer)
  - [Thunder](#en-thunder)
  - [Phoenix](#en-phoenix)
  - [Candidate Pipeline](#en-candidate-pipeline)
- [How It Works](#en-how-it-works)
  - [Pipeline Stages](#en-pipeline-stages)
  - [Scoring and Ranking](#en-scoring-and-ranking)
  - [Filtering](#en-filtering)
- [Key Design Decisions](#en-key-design-decisions)
- [License](#en-license)

---

<h2 id="en-overview">Overview</h2>

The For You feed algorithm retrieves, ranks, and filters posts from two sources:

1. **In-Network (Thunder)**: Posts from accounts you follow
2. **Out-of-Network (Phoenix Retrieval)**: Posts discovered from a global corpus

Both sources are combined and ranked together using **Phoenix**, a Grok-based transformer model that predicts engagement probabilities for each post. The final score is a weighted combination of these predicted engagements.

We have eliminated every single hand-engineered feature and most heuristics from the system. The Grok-based transformer does all the heavy lifting by understanding your engagement history (what you liked, replied to, shared, etc.) and using that to determine what content is relevant to you.

---

<h2 id="en-system-architecture">System Architecture</h2>

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    FOR YOU FEED REQUEST                                     │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                         HOME MIXER                                          │
│                                    (Orchestration Layer)                                    │
├─────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                   QUERY HYDRATION                                   │   │
│   │  ┌──────────────────────────┐    ┌──────────────────────────────────────────────┐   │   │
│   │  │ User Action Sequence     │    │ User Features                                │   │   │
│   │  │ (engagement history)     │    │ (following list, preferences, etc.)          │   │   │
│   │  └──────────────────────────┘    └──────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                  CANDIDATE SOURCES                                  │   │
│   │         ┌─────────────────────────────┐    ┌────────────────────────────────┐       │   │
│   │         │        THUNDER              │    │     PHOENIX RETRIEVAL          │       │   │
│   │         │    (In-Network Posts)       │    │   (Out-of-Network Posts)       │       │   │
│   │         │                             │    │                                │       │   │
│   │         │  Posts from accounts        │    │  ML-based similarity search    │       │   │
│   │         │  you follow                 │    │  across global corpus          │       │   │
│   │         └─────────────────────────────┘    └────────────────────────────────┘       │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                      HYDRATION                                      │   │
│   │  Fetch additional data: core post metadata, author info, media entities, etc.       │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                      FILTERING                                      │   │
│   │  Remove: duplicates, old posts, self-posts, blocked authors, muted keywords, etc.   │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                       SCORING                                       │   │
│   │  ┌──────────────────────────┐                                                       │   │
│   │  │  Phoenix Scorer          │    Grok-based Transformer predicts:                   │   │
│   │  │  (ML Predictions)        │    P(like), P(reply), P(repost), P(click)...          │   │
│   │  └──────────────────────────┘                                                       │   │
│   │               │                                                                     │   │
│   │               ▼                                                                     │   │
│   │  ┌──────────────────────────┐                                                       │   │
│   │  │  Weighted Scorer         │    Weighted Score = Σ (weight × P(action))            │   │
│   │  │  (Combine predictions)   │                                                       │   │
│   │  └──────────────────────────┘                                                       │   │
│   │               │                                                                     │   │
│   │               ▼                                                                     │   │
│   │  ┌──────────────────────────┐                                                       │   │
│   │  │  Author Diversity        │    Attenuate repeated author scores                   │   │
│   │  │  Scorer                  │    to ensure feed diversity                           │   │
│   │  └──────────────────────────┘                                                       │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                      SELECTION                                      │   │
│   │                    Sort by final score, select top K candidates                     │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                              FILTERING (Post-Selection)                             │   │
│   │                 Visibility filtering (deleted/spam/violence/gore etc)               │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                     RANKED FEED RESPONSE                                    │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

<h2 id="en-components">Components</h2>

<h3 id="en-home-mixer">Home Mixer</h3>

**Location:** [`home-mixer/`](home-mixer/)

The orchestration layer that assembles the For You feed. It leverages the `CandidatePipeline` framework with the following stages:

| Stage | Description |
|-------|-------------|
| Query Hydrators | Fetch user context (engagement history, following list) |
| Sources | Retrieve candidates from Thunder and Phoenix |
| Hydrators | Enrich candidates with additional data |
| Filters | Remove ineligible candidates |
| Scorers | Predict engagement and compute final scores |
| Selector | Sort by score and select top K |
| Post-Selection Filters | Final visibility and dedup checks |
| Side Effects | Cache request info for future use |

The server exposes a gRPC endpoint (`ScoredPostsService`) that returns ranked posts for a given user.

---

<h3 id="en-thunder">Thunder</h3>

**Location:** [`thunder/`](thunder/)

An in-memory post store and realtime ingestion pipeline that tracks recent posts from all users. It:

- Consumes post create/delete events from Kafka
- Maintains per-user stores for original posts, replies/reposts, and video posts
- Serves "in-network" post candidates from accounts the requesting user follows
- Automatically trims posts older than the retention period

Thunder enables sub-millisecond lookups for in-network content without hitting an external database.

---

<h3 id="en-phoenix">Phoenix</h3>

**Location:** [`phoenix/`](phoenix/)

The ML component with two main functions:

#### 1. Retrieval (Two-Tower Model)
Finds relevant out-of-network posts:
- **User Tower**: Encodes user features and engagement history into an embedding
- **Candidate Tower**: Encodes all posts into embeddings
- **Similarity Search**: Retrieves top-K posts via dot product similarity

#### 2. Ranking (Transformer with Candidate Isolation)
Predicts engagement probabilities for each candidate:
- Takes user context (engagement history) and candidate posts as input
- Uses special attention masking so candidates cannot attend to each other
- Outputs probabilities for each action type (like, reply, repost, click, etc.)

See [`phoenix/README.md`](phoenix/README.md) for detailed architecture documentation.

---

<h3 id="en-candidate-pipeline">Candidate Pipeline</h3>

**Location:** [`candidate-pipeline/`](candidate-pipeline/)

A reusable framework for building recommendation pipelines. Defines traits for:

| Trait | Purpose |
|-------|---------|
| `Source` | Fetch candidates from a data source |
| `Hydrator` | Enrich candidates with additional features |
| `Filter` | Remove candidates that shouldn't be shown |
| `Scorer` | Compute scores for ranking |
| `Selector` | Sort and select top candidates |
| `SideEffect` | Run async side effects (caching, logging) |

The framework runs sources and hydrators in parallel where possible, with configurable error handling and logging.

---

<h2 id="en-how-it-works">How It Works</h2>

<h3 id="en-pipeline-stages">Pipeline Stages</h3>

1. **Query Hydration**: Fetch the user's recent engagements history and metadata (eg. following list)

2. **Candidate Sourcing**: Retrieve candidates from:
   - **Thunder**: Recent posts from followed accounts (in-network)
   - **Phoenix Retrieval**: ML-discovered posts from the global corpus (out-of-network)

3. **Candidate Hydration**: Enrich candidates with:
   - Core post data (text, media, etc.)
   - Author information (username, verification status)
   - Video duration (for video posts)
   - Subscription status

4. **Pre-Scoring Filters**: Remove posts that are:
   - Duplicates
   - Too old
   - From the viewer themselves
   - From blocked/muted accounts
   - Containing muted keywords
   - Previously seen or recently served
   - Ineligible subscription content

5. **Scoring**: Apply multiple scorers sequentially:
   - **Phoenix Scorer**: Get ML predictions from the Phoenix transformer model
   - **Weighted Scorer**: Combine predictions into a final relevance score
   - **Author Diversity Scorer**: Attenuate repeated author scores for diversity
   - **OON Scorer**: Adjust scores for out-of-network content

6. **Selection**: Sort by score and select the top K candidates

7. **Post-Selection Processing**: Final validation of post candidates to be served

---

<h3 id="en-scoring-and-ranking">Scoring and Ranking</h3>

The Phoenix Grok-based transformer model predicts probabilities for multiple engagement types:

```
Predictions:
├── P(favorite)
├── P(reply)
├── P(repost)
├── P(quote)
├── P(click)
├── P(profile_click)
├── P(video_view)
├── P(photo_expand)
├── P(share)
├── P(dwell)
├── P(follow_author)
├── P(not_interested)
├── P(block_author)
├── P(mute_author)
└── P(report)
```

The **Weighted Scorer** combines these into a final score:

```
Final Score = Σ (weight_i × P(action_i))
```

Positive actions (like, repost, share) have positive weights. Negative actions (block, mute, report) have negative weights, pushing down content the user would likely dislike.

---

<h3 id="en-filtering">Filtering</h3>

Filters run at two stages:

**Pre-Scoring Filters:**
| Filter | Purpose |
|--------|---------|
| `DropDuplicatesFilter` | Remove duplicate post IDs |
| `CoreDataHydrationFilter` | Remove posts that failed to hydrate core metadata |
| `AgeFilter` | Remove posts older than threshold |
| `SelfpostFilter` | Remove user's own posts |
| `RepostDeduplicationFilter` | Dedupe reposts of same content |
| `IneligibleSubscriptionFilter` | Remove paywalled content user can't access |
| `PreviouslySeenPostsFilter` | Remove posts user has already seen |
| `PreviouslyServedPostsFilter` | Remove posts already served in session |
| `MutedKeywordFilter` | Remove posts with user's muted keywords |
| `AuthorSocialgraphFilter` | Remove posts from blocked/muted authors |

**Post-Selection Filters:**
| Filter | Purpose |
|--------|---------|
| `VFFilter` | Remove posts that are deleted/spam/violence/gore etc. |
| `DedupConversationFilter` | Deduplicate multiple branches of the same conversation thread |

---

<h2 id="en-key-design-decisions">Key Design Decisions</h2>

### 1. No Hand-Engineered Features
The system relies entirely on the Grok-based transformer to learn relevance from user engagement sequences. No manual feature engineering for content relevance. This significantly reduces the complexity in our data pipelines and serving infrastructure.

### 2. Candidate Isolation in Ranking
During transformer inference, candidates cannot attend to each other—only to the user context. This ensures the score for a post doesn't depend on which other posts are in the batch, making scores consistent and cacheable.

### 3. Hash-Based Embeddings
Both retrieval and ranking use multiple hash functions for embedding lookup

### 4. Multi-Action Prediction
Rather than predicting a single "relevance" score, the model predicts probabilities for many actions.

### 5. Composable Pipeline Architecture
The `candidate-pipeline` crate provides a flexible framework for building recommendation pipelines with:
- Separation of pipeline execution and monitoring from business logic
- Parallel execution of independent stages and graceful error handling
- Easy addition of new sources, hydrations, filters, and scorers

---

<h2 id="en-license">License</h2>

This project is licensed under the Apache License 2.0. See [LICENSE](LICENSE) for details.

</details>

<details>
<summary><strong>中文</strong></summary>

<a id="zh"></a>

该仓库包含为 X 的“为你推荐（For You）”信息流提供核心推荐系统。它将关注内内容（来自你关注的账号）与关注外内容（通过机器学习检索发现）结合在一起，并使用基于 Grok 的 Transformer 模型对所有内容进行排序。

> **注意：** 本仓库中的 Transformer 实现移植自 xAI 的 [Grok-1 开源版本](https://github.com/xai-org/grok-1)，并针对推荐系统场景进行了适配。

<h2 id="zh-table-of-contents">目录</h2>

- [概览](#zh-overview)
- [系统架构](#zh-system-architecture)
- [组件](#zh-components)
  - [Home Mixer](#zh-home-mixer)
  - [Thunder](#zh-thunder)
  - [Phoenix](#zh-phoenix)
  - [Candidate Pipeline](#zh-candidate-pipeline)
- [工作流程](#zh-how-it-works)
  - [流水线阶段](#zh-pipeline-stages)
  - [评分与排序](#zh-scoring-and-ranking)
  - [过滤](#zh-filtering)
- [关键设计决策](#zh-key-design-decisions)
- [许可证](#zh-license)

---

<h2 id="zh-overview">概览</h2>

“为你推荐”信息流算法从两类来源检索、排序并过滤帖子：

1. **关注内（Thunder）**：来自你关注的账号的帖子
2. **关注外（Phoenix Retrieval）**：从全量内容池中检索出的帖子

两类来源会被合并，并由 **Phoenix**（基于 Grok 的 Transformer 模型）统一排序。模型会预测每条帖子的多种用户互动概率，最终分数为这些预测的加权组合。

系统已经移除了所有手工特征以及绝大部分启发式规则。Grok 版本的 Transformer 会根据你的互动历史（点赞、回复、分享等）理解兴趣偏好，从而判断哪些内容与你相关。

---

<h2 id="zh-system-architecture">系统架构</h2>

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                         为你推荐请求                                        │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                          HOME MIXER                                         │
│                                         （编排层）                                          │
├─────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                     查询补全                                       │   │
│   │  ┌──────────────────────────┐    ┌──────────────────────────────────────────────┐   │   │
│   │  │ 用户行为序列             │    │ 用户特征                                     │   │   │
│   │  │（互动历史）              │    │（关注列表、偏好等）                           │   │   │
│   │  └──────────────────────────┘    └──────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                     候选来源                                       │   │
│   │         ┌─────────────────────────────┐    ┌────────────────────────────────┐       │   │
│   │         │        THUNDER              │    │     PHOENIX RETRIEVAL          │       │   │
│   │         │     （关注内帖子）           │    │     （关注外帖子）              │       │   │
│   │         │                             │    │                                │       │   │
│   │         │  来自你关注账号的帖子       │    │  基于机器学习的相似检索         │       │   │
│   │         │                             │    │  （全量语料）                  │       │   │
│   │         └─────────────────────────────┘    └────────────────────────────────┘       │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                      特征补全                                      │   │
│   │  补充数据：帖子元数据、作者信息、媒体等                                        │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                        过滤                                        │   │
│   │  移除：重复、过旧、本人内容、屏蔽作者、屏蔽词等                                   │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                        打分                                        │   │
│   │  ┌──────────────────────────┐                                                       │   │
│   │  │  Phoenix 打分器          │    基于 Grok 的 Transformer 预测：                     │   │
│   │  │  （模型预测）            │    P(like), P(reply), P(repost), P(click)...          │   │
│   │  └──────────────────────────┘                                                       │   │
│   │               │                                                                     │   │
│   │               ▼                                                                     │   │
│   │  ┌──────────────────────────┐                                                       │   │
│   │  │  加权打分器              │    加权得分 = Σ (权重 × P(行为))                         │   │
│   │  │  （组合预测）            │                                                       │   │
│   │  └──────────────────────────┘                                                       │   │
│   │               │                                                                     │   │
│   │               ▼                                                                     │   │
│   │  ┌──────────────────────────┐                                                       │   │
│   │  │  作者多样性打分器        │    削弱重复作者得分                                     │   │
│   │  │                          │    保证信息流多样性                                   │   │
│   │  └──────────────────────────┘                                                       │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                        选择                                        │   │
│   │                    按最终得分排序，选出 Top K 候选                                   │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                   过滤（后置）                                     │   │
│   │                 可见性过滤（删除/垃圾/暴力/血腥等）                                 │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                       排序后的信息流响应                                     │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

<h2 id="zh-components">组件</h2>

<h3 id="zh-home-mixer">Home Mixer</h3>

**位置：** [`home-mixer/`](home-mixer/)

作为编排层，负责组装“为你推荐”信息流，并基于 `CandidatePipeline` 框架完成以下阶段：

| 阶段 | 描述 |
|------|------|
| Query Hydrators | 拉取用户上下文（互动历史、关注列表等） |
| Sources | 从 Thunder 与 Phoenix 获取候选 |
| Hydrators | 补充候选的额外数据 |
| Filters | 移除不符合条件的候选 |
| Scorers | 预测互动并计算最终分数 |
| Selector | 按分数排序并选出 Top K |
| Post-Selection Filters | 最终可见性与去重检查 |
| Side Effects | 缓存请求信息，便于后续使用 |

服务端通过 gRPC 暴露 `ScoredPostsService`，返回指定用户的排序结果。

---

<h3 id="zh-thunder">Thunder</h3>

**位置：** [`thunder/`](thunder/)

一个内存级帖子存储与实时摄取管道，用于追踪全量用户的最新帖子，具体包括：

- 消费 Kafka 中的发帖/删帖事件
- 按用户维护原帖、回复/转帖、视频帖等存储
- 向关注内请求提供候选
- 自动清理超过保留期的帖子

Thunder 让关注内内容的查找可在亚毫秒级完成，无需访问外部数据库。

---

<h3 id="zh-phoenix">Phoenix</h3>

**位置：** [`phoenix/`](phoenix/)

机器学习核心组件，包含两部分功能：

#### 1. 检索（双塔模型）
用于发现关注外内容：

- **用户塔**：将用户特征与行为序列编码为向量
- **候选塔**：将全量帖子编码为向量
- **相似度检索**：用点积相似度找 Top-K 帖子

#### 2. 排序（候选隔离的 Transformer）
预测每个候选的互动概率：

- 输入用户上下文与候选帖子
- 特殊注意力掩码防止候选相互注意
- 输出各类动作概率（点赞、回复、转发、点击等）

详见 [`phoenix/README.md`](phoenix/README.md)。

---

<h3 id="zh-candidate-pipeline">Candidate Pipeline</h3>

**位置：** [`candidate-pipeline/`](candidate-pipeline/)

可复用推荐流水线框架，定义了以下 Trait：

| Trait | 用途 |
|------|------|
| `Source` | 从数据源获取候选 |
| `Hydrator` | 为候选补充特征 |
| `Filter` | 移除不应展示的候选 |
| `Scorer` | 计算排序分数 |
| `Selector` | 排序并选出 Top K |
| `SideEffect` | 异步副作用（缓存、日志等） |

该框架在可能情况下并行执行 sources 与 hydrators，并提供可配置的错误处理与日志能力。

---

<h2 id="zh-how-it-works">工作流程</h2>

<h3 id="zh-pipeline-stages">流水线阶段</h3>

1. **Query Hydration**：拉取用户近期互动历史与元数据（如关注列表）

2. **Candidate Sourcing**：候选召回来自：
   - **Thunder**：关注内的近期帖子
   - **Phoenix Retrieval**：关注外的机器学习检索结果

3. **Candidate Hydration**：补齐候选特征：
   - 帖子核心内容（文本、媒体等）
   - 作者信息（用户名、认证状态等）
   - 视频时长（视频类候选）
   - 订阅状态

4. **Pre-Scoring Filters**：过滤：
   - 重复内容
   - 过旧帖子
   - 用户本人发布内容
   - 被屏蔽/拉黑的作者
   - 含屏蔽关键词的内容
   - 已看或已服务的内容
   - 无权限订阅内容

5. **Scoring**：串行应用多个 scorer：
   - **Phoenix Scorer**：从 Phoenix Transformer 获取预测
   - **Weighted Scorer**：加权组合得到最终分数
   - **Author Diversity Scorer**：降低重复作者的分数
   - **OON Scorer**：调节关注外内容分数

6. **Selection**：按分数排序并选出 Top K

7. **Post-Selection Processing**：最终校验候选可见性

---

<h3 id="zh-scoring-and-ranking">评分与排序</h3>

Phoenix 的 Grok Transformer 会预测多种互动行为概率：

```
Predictions:
├── P(favorite)
├── P(reply)
├── P(repost)
├── P(quote)
├── P(click)
├── P(profile_click)
├── P(video_view)
├── P(photo_expand)
├── P(share)
├── P(dwell)
├── P(follow_author)
├── P(not_interested)
├── P(block_author)
├── P(mute_author)
└── P(report)
```

**Weighted Scorer** 将这些概率合并为最终分数：

```
Final Score = Σ (weight_i × P(action_i))
```

正向动作（点赞、转发、分享等）权重为正，负向动作（拉黑、屏蔽、举报）权重为负，用于抑制不受欢迎内容。

---

<h3 id="zh-filtering">过滤</h3>

过滤在两个阶段进行：

**预打分过滤：**
| 过滤器 | 作用 |
|--------|------|
| `DropDuplicatesFilter` | 移除重复帖子 ID |
| `CoreDataHydrationFilter` | 移除补全核心数据失败的帖子 |
| `AgeFilter` | 过滤超过阈值的旧帖 |
| `SelfpostFilter` | 过滤用户本人帖子 |
| `RepostDeduplicationFilter` | 转帖去重 |
| `IneligibleSubscriptionFilter` | 过滤无权限订阅内容 |
| `PreviouslySeenPostsFilter` | 过滤已看内容 |
| `PreviouslyServedPostsFilter` | 过滤会话内已服务内容 |
| `MutedKeywordFilter` | 过滤含屏蔽词内容 |
| `AuthorSocialgraphFilter` | 过滤被拉黑/屏蔽作者内容 |

**后置过滤：**
| 过滤器 | 作用 |
|--------|------|
| `VFFilter` | 过滤删除/垃圾/暴力/血腥等内容 |
| `DedupConversationFilter` | 会话级去重（同一对话线程） |

---

<h2 id="zh-key-design-decisions">关键设计决策</h2>

### 1. 无手工特征
系统完全依赖 Grok Transformer 从用户互动序列中学习相关性，避免手工特征工程，显著降低数据与服务复杂度。

### 2. 排序阶段候选隔离
Transformer 推理时，候选之间禁止相互注意，只能关注用户上下文，确保分数与批次无关，便于缓存与复现。

### 3. 哈希嵌入
检索与排序都使用多哈希函数进行嵌入查表。

### 4. 多行为预测
模型不是只预测一个“相关度”，而是同时预测多种用户动作概率。

### 5. 可组合流水线架构
`candidate-pipeline` 提供灵活的推荐流水线框架：
- 业务逻辑与执行框架解耦
- 并行执行独立阶段，并提供容错与日志
- 便于新增数据源、补全器、过滤器与打分器

---

<h2 id="zh-license">许可证</h2>

本项目采用 Apache License 2.0 许可，详见 [LICENSE](LICENSE)。

</details>
