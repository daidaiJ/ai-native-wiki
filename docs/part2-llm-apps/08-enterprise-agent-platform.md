# 2.8 企业智能体平台方向认知 [进阶]

> 来源：各方向开源项目官方站点（见各小节项目表）· 验证环境：待验证（本章为认知章节，无命令）

## 概念：为什么需要

**业务痛点**：Agent 从 demo 到生产，会撞上一堵「业务代码解决不了」的墙：Agent 要执行不可信代码、要接多家模型、要代表用户操作、token 成本失控、能力悄悄退化、出问题查不了账（[已必备]：企业规模化上 Agent 时，这些墙必然出现）。这些问题的答案不在某个框架里，而在「平台」里——于是企业开始建设 Agent 平台。

- **从哪来**：单 Agent 应用（调 API + 写业务逻辑）→ 多 Agent 规模化 → 平台化（安全/网关/身份/算力/质量/成本统一治理）
- **是什么**：企业 Agent 平台 = 九个方向的组合，每个方向解决一类「规模化后才暴露」的问题
- **往哪去**：方向正在收敛——网关、可观测、评估快速标准化（LiteLLM / OTel GenAI / RAGAS），沙箱、身份、运行时仍是平台团队的核心地盘

**定位红线**：这些方向多数是平台团队的地盘（学习者不建设平台）。学习者的价值在于：**知道有这些方向、能识别企业需求、能与平台团队对齐**，并从中选择适合自己的入门/择业方向。本章给一张「能力地图」，不教建设。

## 2.8.1 沙箱（Agent 代码/工具执行隔离）

**方向是什么**：Agent 要执行代码、调工具，但代码与工具输出不可信——沙箱（sandbox，隔离执行环境）把执行放进受限环境（微虚机 / 容器 / WASM，WASM 即 WebAssembly，可移植字节码格式），限制资源与网络，出事不连坐。这是 2.7 防御分层里「隔离执行」的平台侧实现。

| 项目 | 来源 | 一句话定位 |
|---|---|---|
| E2B | https://e2b.dev/ | Agent 云沙箱：为 AI Agent 提供即开即用的隔离执行环境 |
| Daytona | https://github.com/daytonaio/daytona | 开发环境即代码：沙箱化开发环境管理 |
| Firecracker | https://github.com/firecracker-microvm/firecracker | AWS 微虚机（MicroVM）：毫秒级启动、单虚机开销最小 |
| gVisor | https://github.com/google/gvisor | Google 容器沙箱运行时：用户态内核拦截系统调用 |
| Extism | https://github.com/extism/extism | WASM 插件框架：不可信代码编译成 WASM 隔离执行 |

**含金量与难度：高**。安全敏感——沙箱逃逸等于平台沦陷；需要内核/虚拟化知识，是安全团队与平台团队的地盘。

**企业着力点**：不可信代码执行（数据分析、代码生成产物）、资源/网络隔离、安全边界。

**入门择业建议**：业务开发者了解能力边界即可——知道「Agent 执行要进沙箱」、能向平台提需求（隔离级别、网络策略）。想理解隔离模型，可先玩 Extism（WASM 门槛低）。

## 2.8.2 LLM/Agent 网关

**方向是什么**：所有模型请求过一道统一网关（gateway，统一入口代理）：多 provider 路由、key 治理、限流、成本观测。业务代码只认一个 OpenAI 兼容端点，换模型不改代码。

| 项目 | 来源 | 一句话定位 |
|---|---|---|
| LiteLLM | https://github.com/BerriAI/litellm | 统一代理事实标准：100+ provider 一个接口，key 治理 + 预算 |
| Envoy AI Gateway | https://github.com/envoyproxy/ai-gateway | CNCF 沙箱：Envoy 系 AI 网关，provider 路由/fallback |
| Portkey | https://github.com/Portkey-AI/gateway | AI 网关：路由/重试/缓存/观测一体 |
| Helicone | https://github.com/Helicone/helicone | LLM 网关 + 观测：代理层记录 token 与成本 |

**含金量与难度：中**。生态成熟（LiteLLM 已是事实标准），概念不深；做扎实要懂限流/路由/可观测，但业务开发者完全能上手。

**企业着力点**：多 provider 接入与切换、key 治理（防泄露/防滥用）、限流配额、成本观测。

**入门择业建议**：**最推荐入门方向之一**。自建一个 LiteLLM 代理接 2~3 个模型，理解路由/预算/观测，即可在企业落地；呼应手册 3.5（推理网关）。

## 2.8.3 统一授权认证管理（Agent 身份与权限）

**方向是什么**：Agent 不是「一个用户」，是「一组身份」——它要代表用户调系统、跨系统协作。需要身份认证（你是谁）、授权（你能干什么）、最小权限与审计追踪。

| 项目 | 来源 | 一句话定位 |
|---|---|---|
| Keycloak | https://github.com/keycloak/keycloak | Red Hat 维护的开源 IAM：OIDC/SAML 单点登录事实标准 |
| Ory（Kratos/Hydra） | https://github.com/ory/kratos | 开源身份栈：Kratos 注册登录、Hydra OAuth2/OIDC |
| OpenFGA | https://github.com/openfga/openfga | Zanzibar 系细粒度授权：关系模型做权限判断 |
| SPIFFE/SPIRE | https://github.com/spiffe/spire | 服务身份标准：为每个工作负载发短期身份证书 |

**含金量与难度：高**。安全敏感 + 协议多（OIDC/OAuth2/关系模型），概念门槛高，平台/安全团队地盘。

**企业着力点**：Agent 身份建模（Agent 代表谁、能做什么）、最小权限、审计追踪、跨系统授权。

**入门择业建议**：业务开发者了解概念即可——能说清「Agent 身份 ≠ 用户身份」、知道 OIDC（OpenID Connect，基于 OAuth2 的身份认证协议）与细粒度授权是两回事。想深入从 OIDC 概念入手读 Keycloak 文档。

## 2.8.4 智能体运行时（共享/高并发）

**方向是什么**：模型推理是 Agent 平台的「算力底座」：有限 GPU 服务更多用户——批处理、共享推理、弹性伸缩、语义缓存（相似问题直接命中缓存，免去重复调用）。这是 Part 3 的平台侧全景。

| 项目 | 来源 | 一句话定位 |
|---|---|---|
| vLLM | https://github.com/vllm-project/vllm | 推理引擎事实标准：PagedAttention 高吞吐 |
| Ray Serve | https://github.com/ray-project/ray | 分布式模型服务：Python 原生、弹性扩缩 |
| BentoML | https://github.com/bentoml/BentoML | AI 应用产品化：模型打包/部署/服务 |
| KServe | https://kserve.github.io/website/ | 推理平台层标准：serverless/金丝雀 |
| K8s Inference Extension | https://gateway-api-inference-extension.sigs.k8s.io/ | 新秀：网关感知推理负载，InferencePool 路由 |

**含金量与难度：高**。基础设施：GPU 调度、显存管理、分布式推理——平台团队核心地盘。

**企业着力点**：批处理、语义缓存、共享推理、弹性伸缩。

**入门择业建议**：云原生工程师切入 AI 的第一梯队（复用 K8s 知识）：vLLM → KServe → Inference Extension（手册 Part 3 主线）；业务开发者了解能力边界即可。

## 2.8.5 A2A 多智能体协作

**方向是什么**：Agent 之间怎么协作？两个层面：协议标准（A2A 跨企业 Agent 互操作、MCP 工具互操作）+ 编排框架（单应用内多 Agent 任务编排）。

| 项目 | 来源 | 一句话定位 |
|---|---|---|
| Google A2A | https://a2a-protocol.org/ | Agent 互操作协议（Linux Foundation 托管）：Agent 间发现/协商/协作 |
| MCP | https://modelcontextprotocol.io/ | 工具互操作事实标准：Agent 连接工具/数据的统一协议 |
| CrewAI | https://github.com/crewAIInc/crewAI | 角色扮演多智能体框架：入门友好 |
| AutoGen | https://github.com/microsoft/autogen | 多智能体框架（新项目走 Microsoft Agent Framework 继任者） |
| LangGraph | https://github.com/langchain-ai/langgraph | 图状态机编排：2026 首选编排框架 |

**含金量与难度：中高**。标准演进中（A2A/MCP 还在变），跟标准要持续投入；编排框架本身不难。

**企业着力点**：协议标准选型、任务编排、状态管理、可靠性（多 Agent 失败处理）。

**入门择业建议**：**推荐**。MCP 是当下最值得投入的协议——工具互操作是 Agent 落地的关键；从「给 Agent 写一个 MCP server」入手，再学 LangGraph 编排；A2A 观察即可。

## 2.8.6 知识工程（wiki 记忆）

**方向是什么**：Agent 的知识从哪来？RAG 数据底座（解析/切块/检索）+ 记忆分层（短期对话记忆/长期用户记忆/组织知识）。

| 项目 | 来源 | 一句话定位 |
|---|---|---|
| pgvector | https://github.com/pgvector/pgvector | PostgreSQL 向量扩展：中台优先，复用现有库 |
| Qdrant | https://qdrant.tech/documentation/ | 专用向量库：Rust 实现，性能好 |
| Milvus | https://milvus.io/docs | 分布式向量库：大规模场景 |
| GraphRAG | https://github.com/microsoft/graphrag | 知识图谱 RAG：实体关系建图，全局问答 |
| Mem0 | https://github.com/mem0ai/mem0 | Agent 记忆层：跨会话长期记忆 |
| Letta | https://github.com/letta-ai/letta | 记忆型 Agent 框架（MemGPT 继任）：上下文管理 |

**含金量与难度：中**。贴近业务，业务开发者友好；难点在「质量」不在「跑通」。

**企业着力点**：文档解析、切块策略、检索质量、记忆分层（呼应手册 2.3）。

**入门择业建议**：**推荐**。业务开发者最自然的切入点：先跑通 2.3 的 RAG，再深入解析/切块/检索质量；GraphRAG 与记忆层作为进阶。

## 2.8.7 Token 成本优化

**方向是什么**：LLM 成本 = token 消耗 × 单价。优化手段：缓存（语义缓存）、模型路由（便宜模型先试、分级模型）、压缩（prompt 精简）、预算管控。

| 项目 | 来源 | 一句话定位 |
|---|---|---|
| LiteLLM 预算 | https://github.com/BerriAI/litellm | 预算/限流：按 key/团队设 token 预算 |
| Langfuse | https://github.com/langfuse/langfuse | 观测：token 用量与成本归因 |
| promptfoo | https://github.com/promptfoo/promptfoo | 回归测试：改 prompt 前先跑评估，防「省成本省出质量事故」 |
| GPTCache | https://github.com/zilliztech/GPTCache | 语义缓存：相似问题命中缓存免调用 |

**含金量与难度：中**。概念简单，但「省成本不省质量」需要评估配合（呼应 2.5）。

**企业着力点**：缓存、模型路由（便宜模型先试）、压缩、预算管控。

**入门择业建议**：**推荐**。与网关方向天然搭配：LiteLLM 预算 + Langfuse 观测 + promptfoo 回归，一套组合拳即可在企业落地。

## 2.8.8 智能体评估与版本控制

**方向是什么**：Agent 是概率系统，改一个 prompt/工具/模型都可能让能力退化。评估 = 用评估集 + 指标量化质量；版本控制 = 评估结果进 CI 回归门禁，用金标集（人工标注的高质量标准答案集）对比版本。

| 项目 | 来源 | 一句话定位 |
|---|---|---|
| RAGAS | https://docs.ragas.io/ | RAG 评估指标事实标准（呼应手册 2.5） |
| promptfoo | https://github.com/promptfoo/promptfoo | CI 级回归：prompt/模型对比测试 |
| LangSmith / Langfuse 评估 | https://docs.langchain.com/ | 商业/开源评估平台：在线评估与追踪 |
| Braintrust | https://github.com/braintrustdata/braintrust-sdk ⏳ | 评估平台：数据集/评分/回归 |
| AgentBench | https://github.com/THUDM/AgentBench | 清华：Agent 能力基准 |

**含金量与难度：中**。价值高——评估是 AI 产品最先该建的东西；隐性难点在「评估集质量」。

**企业着力点**：评估集建设、回归门禁、金标集、版本对比（呼应手册 2.5）。

**入门择业建议**：**最推荐**。业务开发者从 RAGAS + promptfoo 入手，把评估接进 CI——这是「会选会对齐」的典型能力。

## 2.8.9 Agent 可观测性

**方向是什么**：Agent 调用链长（LLM→工具→RAG→多轮），出问题要能定位：调用链追踪、token 计量、质量指标。埋点（在代码里记录可观测数据）口径要统一，否则各团队各埋各的。

| 项目 | 来源 | 一句话定位 |
|---|---|---|
| Langfuse | https://github.com/langfuse/langfuse | AgentOps 开源首选：追踪/评估/成本一体 |
| OpenInference | https://github.com/Arize-ai/openinference ⏳ | 开源追踪标准：LLM 调用链语义约定 |
| OTel GenAI 语义约定 | https://github.com/open-telemetry/semantic-conventions-genai | 统一埋点口径：gen_ai.usage.* 等字段（呼应 SPEC §7） |

**含金量与难度：中**。概念不深，但「埋点口径统一」是组织级难题。

**企业着力点**：调用链、token 计量、质量指标（与评估方向配合）。

**入门择业建议**：与评估/成本方向搭配学习：Langfuse 跑起来，理解 OTel GenAI 字段；业务开发者能直接做埋点。

## 入门择业建议：四个适合学习者的方向

九个方向里，平台团队核心地盘（沙箱、授权认证、运行时）了解能力边界即可；业务开发者优先切入这四个：

| 方向 | 难度 | 理由 | 学习路径 |
|---|---|---|---|
| LLM/Agent 网关 + Token 成本优化 | 中 | 生态成熟、上手快、企业当下刚需 | 手册 3.5（推理网关）→ LiteLLM 文档 → 自建代理接 2~3 个模型 → 预算/限流 → Langfuse 成本归因 |
| 智能体评估与版本控制 | 中 | 价值最高（评估是 AI 产品最先该建的）；「会选会对齐」的典型能力 | 手册 2.5（RAGAS 四指标）→ promptfoo CI 回归 → 给 lab2 加评估门禁 |
| 知识工程 | 中 | 业务开发者最自然的切入点，贴近业务价值 | 手册 2.3（RAG 底座）→ 解析/切块质量 → GraphRAG/记忆层进阶 |
| A2A/MCP 协议方向 | 中高 | 标准红利期，工具互操作是 Agent 落地关键 | MCP 文档 → 给 lab2 的 Agent 写一个 MCP server → LangGraph 编排 → A2A 观察 |

共同点：这四个方向都是「业务开发者能直接上手、能向平台提需求、企业当下刚需」，且与手册已有章节（2.3 / 2.5 / 3.5）直接衔接——学完就能用。

## 实用技巧

- **用「能力地图」做认知框架**：把九个方向画成一张表，标注「平台地盘 vs 业务切入」，面试/对齐时按图索骥
- **识别企业需求**：听到「Agent 执行代码」→ 沙箱；「多模型切换/换模型要改代码」→ 网关；「token 超预算」→ 成本；「Agent 变笨了/说不清」→ 评估；「出问题查不了账」→ 可观测
- **与平台团队对齐的提问模板**：这个能力平台提供吗？接入点是什么（API / CRD / 控制台）？配额与 SLA 是多少？
- **择业信号**：招聘 JD 里出现 LiteLLM / RAGAS / Langfuse / MCP 的岗位，都是业务侧能切入的方向

## 考察问题

1. 为什么「Agent 身份 ≠ 用户身份」？企业里 Agent 代表谁执行操作，审计怎么记？（线索：OIDC 用户身份 vs SPIFFE 服务身份；Zanzibar 关系模型）
2. LiteLLM 已是事实标准，为什么大厂还要自研 AI 网关？（线索：成本、数据合规、与内部平台/计费系统集成）
3. 「评估最先该建」但多数团队最后才建，为什么？（线索：评估集建设成本高、见效慢、需要标注人力）

## 经验之谈

- AI 产品最先该建的是评估——没有评估，改 prompt、换模型、加工具都是盲改；评估集是团队最值钱的资产（观点转述，不署名）
- 平台方向（沙箱/授权/运行时）是平台团队的地盘，业务开发者的精力放在「识别需求 + 对齐」，而不是「自己建」
- 标准演进期（A2A/MCP）投入要「跟协议不跟实现」：协议是长期资产，框架会换

## 架构师视角

- 解决什么问题：企业 Agent 从 demo 到生产的平台能力地图
- 何时用：企业要规模化上 Agent 时；何时不用：单应用/小团队可先用 SaaS 或轻量方案，不必自建平台
- 权衡：平台建设成本 vs 业务速度；自建 vs 采购（网关/评估/观测都有成熟 SaaS）
- **平台提供了什么能力边界**：沙箱隔离、推理服务、身份授权、网关、可观测——都是平台能力，业务侧不重复建设
- **业务接入点在哪**：模型 API（网关）、评估 CI、埋点 SDK、MCP 工具
- **需要和基础设施团队对齐什么**：隔离级别、配额与预算、身份模型、埋点口径（OTel GenAI）

## 核心收获表

| 能力维度 | 具体收获 |
|---|---|
| 方向认知 | 能说出企业 Agent 平台九个方向，及各自的难度、着力点、代表项目 |
| 定位 | 能区分平台团队地盘（沙箱/授权/运行时）与业务开发者切入方向 |
| 择业 | 能说出 4 个适合学习者的入门方向及学习路径 |
| 对齐 | 能用固定三问与平台团队对齐需求（能力边界/接入点/对齐项） |

## 推荐开源项目表

| 项目 | 链接 | 研读重点 |
|---|---|---|
| LiteLLM | https://github.com/BerriAI/litellm | 网关 + 预算：统一代理怎么治理 key 与成本 |
| RAGAS | https://docs.ragas.io/ | 评估指标：四指标怎么算、怎么接 CI |
| Langfuse | https://github.com/langfuse/langfuse | 观测 + 评估：调用链与 token 计量 |
| MCP | https://modelcontextprotocol.io/ | 协议：工具互操作标准，写一个 server 试试 |
| vLLM | https://github.com/vllm-project/vllm | 推理引擎：理解「运行时」在解决什么问题 |

## 常见问题排查

| 现象 | 排查路径 |
|---|---|
| 换模型要改业务代码 | 接网关（LiteLLM），业务只认 OpenAI 兼容端点 |
| token 成本失控 | 网关预算 + Langfuse 归因 + 语义缓存 + 便宜模型路由 |
| Agent 能力退化说不清 | 建评估集 + promptfoo 回归门禁，版本对比 |
| Agent 执行代码怕出事 | 找平台要沙箱隔离执行，业务侧做工具白名单（2.7） |
| 出问题查不了账 | 埋点接 OTel GenAI 口径，Langfuse 看调用链 |