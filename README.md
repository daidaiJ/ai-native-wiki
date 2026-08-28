# 从云原生到 AI 云原生：开发者转型学习手册

面向业务开发/中台开发 + 基础设施二开适配者的转型学习手册 + 知识 wiki。目标读者从「实现功能」转向「设计系统」，成长为 AI 时代有架构视野和方案设计能力的资深开发/架构师（业务/中台视角）。

**定位红线**：学习者不建设基础设施，是使用、适配、集成平台的人。内容教「驾驭平台、适配平台、与基础设施团队协作」。

## 快速开始

```bash
# 1. 环境准备（30 分钟）：Docker + kind + kubectl
#    详见 docs/part0-intro/（规划中，先按附录 A 准备）
# 2. 读学习路径：LEARNING.md（三阶段 + 四层提升）
# 3. 用 AI 学：把 AGENTS.md 喂给 AI 助手，让它当你的学习教练
```

## 学习导航

| 入口 | 用途 |
|---|---|
| [LEARNING.md](LEARNING.md) | 学习路径：三阶段 + 四层提升（核心技能→业务心法→框架理论→趋势前瞻）+ 验收标准 |
| [SPEC.md](SPEC.md) | 学习规格：四层目标模型、章节结构、节奏与自测、AI 学习法 |
| [AGENTS.md](AGENTS.md) | AI 辅助学习规约：AI 四角色（陪练/老师/评审/检索加速器）+ 使用纪律三条 |
| [docs/appendix/knowledge-retrieval-guide.md](docs/appendix/knowledge-retrieval-guide.md) | 知识检索手册：按主题域定位权威来源（遇到问题先查这里） |

## 内容目录

```
docs/part0-intro/          转型导论（规划中）
docs/part1-cloud-native/   云原生基础（CKAD 全覆盖）[实用]
docs/part2-llm-apps/       LLM 应用开发（RAG/Agent/评估/安全）[实用]
docs/part3-ai-on-k8s/      AI 应用上云（推理/GPU/网关/成本）[实用]
docs/part4-arch-advanced/  云原生架构进阶（模式/GitOps/二开适配）[进阶]
docs/part5-ai-arch/        AI 架构进阶（规划中）[进阶]
docs/part6-architect/      架构师修炼（ADR/SLO/案例研读）[进阶]
docs/appendix/             附录（CKAD 备考/知识检索手册/术语表）
```

标注：**[实用]** = 主线必学；**[进阶]** = 支线按需。每章结构：章 README 概览 → 正文详解（概念→动手→实用技巧→考察问题→经验之谈→架构师视角）→ 章尾收束（核心收获/推荐项目/排查）。

## 用 AI 学习（本项目的核心机制）

AI 是**学习教练**，不是答案机。四个角色：

- **陪练**：出题/批改/追问——考察问题当讨论伙伴
- **老师**：概念类比/术语人话/追问到懂
- **评审**：YAML/方案找漏洞——对照「架构师视角固定三问」
- **检索加速器**：定位官方文档/KEP/论文——先查检索手册

**纪律三条**：AI 会编造（事实回官方验证）；AI 不是验证环境（命令本地 kind 跑通）；AI 不替代思考（先自己想）。

## 项目状态

- **内容主体已完成**（2026-08）：Part 1/2/3/4/6 核心章节 + 附录 A/B + 检索手册
- **规划中**：Part 0、Part 5、各 Part 未写章节（README 导航表有标注）
- **创作期文档归档**：`meta/archive/`（PLAN/SPEC/REQUIREMENTS/旧 AGENTS，只读引用）
- **AI 草稿归档**：`meta/`（glm/grok/kimi 增补原文）
- **工作流 skill**：`skills/wiki-evolution/`（AI 驱动 wiki 演进：创作 + 学习双期）

## 贡献与反馈

- 内容纠错/补充：按 AGENTS.md 内容纪律（来源标注 + 事实第一），合入 docs/ 需符合 SPEC 章节结构
- 学习问题：让 AI 当教练（AGENTS.md），或按检索手册查权威来源