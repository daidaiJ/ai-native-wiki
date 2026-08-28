# 2.3 前置：RAG 的数据底座——Embedding 与向量库选型 [实用]

> 来源：[MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard) · [pgvector](https://github.com/pgvector/pgvector) · [Qdrant](https://qdrant.tech/documentation/) · [Milvus](https://milvus.io/docs) · 验证环境：待验证

## 概念：为什么需要

**业务痛点**：企业知识库 RAG 问答（把散落文档变成 7×24 问答）是 [已必备] 场景——但很多团队把精力花在框架上，效果却差：答非所问、检索不到关键信息。原因在于 RAG 的效果上限不在「框架」，而在检索；检索的上限在两件事：

- **Embedding**（向量化）：把文字压成向量，语义相近 = 向量距离近。检索质量的「天花板」由它决定
- **向量库**：存向量并做近邻搜索的存储。决定天花板能撑到的规模与过滤能力

- **从哪来**：BM25 关键词检索 → word2vec 静态词向量 → BERT 时代句向量 → 对比学习检索模型（bge / gte / jina 等开源系）→ 多语言统一
- **是什么**：选型题 = 一个 embedding 模型 + 一个向量库
- **往哪去**：embedding 与 reranker（重排序器，对召回结果做精排的模型）组合成为标配两段式：先向量召回，后精排

## 选型决策表（中台开发者视角）

| 路线 | 何时选 | 一句话理由 |
|---|---|---|
| pgvector（PG 插件） | 千万级以下向量、公司已有 PostgreSQL | 零新增组件，运维成本最低，中台团队首选 |
| Qdrant | 需要独立部署、云原生向量库 | Rust 实现、过滤检索能力强、部署简单 |
| Milvus | 十亿级向量、有专职运维 | 分布式架构、生态大、但运维重 |

Embedding 选型：MTEB 榜按任务类型查分（中文场景看 C-Retrieval / Retrieval 任务），注意维度——维度越高存储与内存成本越高，不是越大越好。

## 动手（pgvector 15 分钟最短路径）

> 验证环境：待验证（需本地 PostgreSQL + pgvector 插件）

```sql
CREATE EXTENSION vector;
CREATE TABLE chunks (id bigserial PRIMARY KEY, content text, embedding vector(1024));
-- <=> 是向量库里的「最近距离」比较算子（余弦距离）
SELECT content FROM chunks ORDER BY embedding <=> $1 LIMIT 5;
```

## 实用技巧

- 数据量上来后必须建近似最近邻索引（pgvector 的 HNSW / IVFFlat 两类索引，以官方文档为准），否则全表扫描，查询会越来越慢
- 给 chunk 打上文档 ID / 章节 / 权限标签，检索时先过滤再排序，兼顾权限与精度
- 切块（chunking）策略值得先调：段落切块 + 重叠窗口是常见起点，效果影响常大于换 embedding 模型
- 用 LangChain / LlamaIndex 的 VectorStore 接口接向量库，业务代码不直连向量库 SDK——为将来迁移留防腐层

## 考察问题

- 为什么很多团队最后没用专用向量库？什么时候必须换？（线索：数据规模、QPS、组合过滤复杂度、运维预算）
- 检索结果差，先查 embedding 还是先查切块？（线索：见「经验之谈」）

## 经验之谈

多数企业 RAG 卡在文档解析与切块（chunking）而非向量库选型；切块策略的影响常大于换 embedding 模型——这是社区通行经验，具体引用待补。

## 架构师视角

- 解决什么问题：让 RAG 检索质量可预期、可扩展
- 何时用：先 pgvector 起步（零新增组件），量级上来再迁移
- 何时不用：数据量小到关键词检索就够时，不必上向量库
- 权衡：迁移成本主要在重建索引；选型时用 VectorStore 接口做防腐层
- **平台提供了什么能力边界**：向量库是平台组件还是业务自建，决定运维责任归属；公司已有 PG 集群时优先申请 pgvector 插件
- **业务接入点在哪**：通过 VectorStore 接口 / SQL 接入，不直连向量库 SDK
- **需要和基础设施团队对齐什么**：pgvector 插件版本、索引与内存配额、是否需要独立向量库实例

## 核心收获表

| 能力维度 | 具体收获 |
|---|---|
| 选型 | 能按数据规模与运维预算在 pgvector / Qdrant / Milvus 间决策 |
| 检索 | 能跑通「embedding 入库 → 向量检索」最短路径 |
| 架构 | 理解两段式检索（召回 + 精排）与防腐层设计 |

## 推荐开源项目表

| 项目 | 链接 | 研读重点 |
|---|---|---|
| pgvector | https://github.com/pgvector/pgvector | 索引类型与 SQL 用法 |
| Qdrant | https://qdrant.tech/documentation/ | 过滤检索能力 |
| Milvus | https://milvus.io/docs | 分布式架构 |
| MTEB | https://huggingface.co/spaces/mteb/leaderboard | embedding 选型基准 |

## 常见问题排查

| 现象 | 排查路径 |
|---|---|
| 检索结果答非所问 | 先查切块粒度与重叠，再查 embedding 模型与领域匹配度 |
| 查询越来越慢 | 检查是否建了索引、数据量是否超过当前方案规模 |
| 内存/存储暴涨 | 检查 embedding 维度与向量库配置，维度不是越大越好 |