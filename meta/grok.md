# grok.md — 面向「平台使用者 / 二开适配」的知识源补强

> 配合 `docs/appendix/knowledge-retrieval-guide.md` 使用。手册已覆盖 K8s 核心、CNCF 生态、GPU/推理、LLM/Agent 框架；本文补上**更贴合本项目读者**的入口与查法。
>
> **读者定位**（与 PLAN / AGENTS.md 一致）：业务开发、中台开发、基础设施二开适配者。日常是写业务、调接口、对接公司 PaaS、基于 K8s API 做适配。**不建设集群、不运维 GPU 池、不实现服务网格。**
>
> 日期：2026-08-28

---

## 0. 怎么用这份补强

遇到问题时仍先打开知识检索手册，按主题域查官方文档。若题目是下面几类，再转到本文对应节：

| 你在做的事 | 翻本文哪一节 |
|---|---|
| 本机搭 kind、写 Dockerfile、查 kubectl | §2 环境与应用交付 |
| 对标 CKAD、Helm、探针、HPA、安全基线 | §3 应用在集群上怎么跑 |
| 写 RAG/Agent、评估质量、防提示注入 | §4 LLM 应用（业务侧） |
| 申请 GPU、接推理 API、看 token/延迟、走网关 | §5 AI 上云（接入侧） |
| 跟平台团队对齐能力、GitOps 交付、读 CRD、写适配器 | §6 驾驭平台与二开 |
| 写技术方案、定 SLO、做权衡 | §7 方案设计 |
| 不确定该读多深、文档会不会过期 | §1 检索约定 |

每条来源带阅读深度，避免把「平台建设文档」读成自己的工作：

| 深度 | 你要达到 | 典型动作 |
|---|---|---|
| **L1 会用** | 能提交清单、能排自己的应用 | 填 YAML、调 OpenAI 兼容 API、看探针失败原因 |
| **L2 会选 / 会对齐** | 能判断用不用、能向平台提需求 | 「我们要 InferencePool / Kueue 队列 / OTel GenAI 字段」 |
| **L3 会建** | 实现或运维平台本身 | **本项目不展开**；正文最多写「找基础设施团队」 |

Istio 数据面、Cilium eBPF、Volcano 调度插件、GPU Operator 装机，一律停在 L2。

---

## 1. 检索约定（补手册 §0）

手册原则仍然有效：官方文档 → 官方博客 → 大牛博客；版本看发布博客 + KEP；生态看 CNCF 状态。下面补**本角色常用**的查法。

### 1.1 按问题类型选入口

| 问题 | 先打开 | 读到哪停 |
|---|---|---|
| 这个字段什么意思、YAML 怎么写 | 项目官方 docs，K8s 再用 `kubectl explain` | L1，能复制能跑 |
| 特性哪一版才能用 | [Kubernetes 发布博客](https://kubernetes.io/blog/) + [KEP](https://github.com/kubernetes/enhancements) | 记下 GA/默认启用，再写进方案 |
| 平台有没有这项能力 | 公司平台文档 → 对不上再查项目 docs 的「用户/应用」章节 | L2，整理成给平台的接口清单 |
| 指标/忠实度/token 含义 | 规范或评估库文档，不要只用 SaaS 产品页 | 定义以规范为准，产品只负责把数跑出来 |
| 安全底线 | OWASP LLM Top 10、K8s Pod Security Standards | 应用侧可落地的缓解，不写攻击步骤 |
| 趋势与采用率 | [CNCF reports](https://www.cncf.io/reports/) | 引用必须带调查年份 |

### 1.2 查 KEP（版本问题）

1. https://github.com/kubernetes/enhancements 搜特性名或编号（如 DRA、in-place resize）
2. 读 `keps/` 里 README 的 **status** 与 **milestone**
3. 对照同版本发布博客，确认 Alpha / Beta / GA、是否默认开启
4. 引用格式：KEP 编号 + 版本 + 博客日期

对本读者：KEP 用来回答「我们集群版本够不够、该不该向平台申请打开某特性」，不用来讲控制面实现。

### 1.3 查 CNCF 地位（选型）

1. https://landscape.cncf.io/ 看 Graduated / Incubating / Sandbox
2. 核对项目页接受日期、所属 SIG
3. **SIG 官方项目**（Kueue、JobSet、Gateway API）即使 star 低，也优先于高星玩具
4. 沙箱项目（llm-d、KAITO、ModelPack、Envoy AI Gateway）默认 L2「观察与对齐」：知道平台可能提供什么，主线实验仍用手册第一梯队（GPU/DRA 文档 → Kueue → vLLM → KServe）

star 只表示热度，标注日期后当附录；架构地位看成熟度 + SIG + 是否出现在平台目录里。

### 1.4 查论文（机制一句话）

arXiv abs 页优先（有版本历史）。写法：**论文解释为什么，官方 docs 解释现在怎么配。** 本项目常引的几篇见 §4、§5。

### 1.5 文档搬家

写作前若官方页写 `Moved` / `archived` / successor，改跟新仓库。GenAI 语义约定已从 `opentelemetry.io/docs/specs/semconv/gen-ai/` 迁到 [semantic-conventions-genai](https://github.com/open-telemetry/semantic-conventions-genai)，旧链接只作跳转。

引用格式与 AGENTS.md 一致：

```text
> 来源：[项目或作者](URL) · 版本或日期：YYYY-MM / vX.Y · 验证环境：kind v0.x / K8s v1.x
```

---

## 2. 环境与应用交付（补 Part 0 / 日常干活）

手册从 K8s 概念开始；本地 30 分钟集群和镜像构建需要这些官方页（全 L1）。

| 你要做的事 | 权威入口 | 检索提示 |
|---|---|---|
| 本机集群（项目默认） | https://kind.sigs.k8s.io/ | Quick Start；本地镜像用 `kind load docker-image` |
| 备选集群 | [minikube](https://minikube.sigs.k8s.io/docs/) · [k3s](https://docs.k3s.io/) | lab 脚本不要和 kind 混用组件假设 |
| 安装 kubectl | https://kubernetes.io/docs/tasks/tools/ | 与集群小版本大致对齐即可 |
| 高频命令 | [kubectl cheatsheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) | CKAD 考场可查；日常排障也够用 |
| 镜像构建 | https://docs.docker.com/ | 多阶段构建走 Dockerfile best practices |
| 「镜像」和 Dockerfile 不是一回事 | [OCI image-spec](https://github.com/opencontainers/image-spec) | 扫 layout/manifest 即可，建立心智模型 |
| 运行时只需知道 CRI | https://kubernetes.io/docs/setup/production-environment/container-runtimes/ · https://containerd.io/docs/ | 不展开 CRI-O 运维 |

**检索口令**：`kind load docker-image`、`kubectl cheatsheet`、`multi-stage Dockerfile`。

---

## 3. 应用在集群上怎么跑（补 Part 1 / CKAD 锚定）

手册把 K8s 指向整站 `kubernetes.io/docs`。下面按**应用开发者任务**拆开，并接上现行 CKAD 官方材料（本项目用 CKAD 当学习标准）。

现行公开考纲在 [cncf/curriculum](https://github.com/cncf/curriculum) 的 `CKAD_Curriculum_v*.pdf`（撰写时为 v1.35）。CNCF 考试页按五域陈述：Application Design and Build、Application Deployment、Observability and Maintenance、Environment/Configuration/Security、Services and Networking。形式与版本：[CNCF CKAD](https://www.cncf.io/training/certification/ckad/) · [Linux Foundation 考试页](https://training.linuxfoundation.org/certification/certified-kubernetes-application-developer-ckad/)。及格线与模拟器说明见 [LF FAQ](https://docs.linuxfoundation.org/tc-docs/certification/faq-cka-ckad-cks.md)（CKAD 为 66%；官方模拟为 Killer.sh）。

考场可查阅：`kubernetes.io/docs`、[Helm 文档](https://helm.sh/docs/)、kubectl cheatsheet。正文示例优先用这三处能搜到的字段和命令。

| 你要做的事 | 权威入口 | 深度 | 检索提示 |
|---|---|---|---|
| 按任务找 YAML | https://kubernetes.io/docs/tasks/ | L1 | 比从概念页抄更接近日常和考试 |
| Helm 发版/回滚 | https://helm.sh/docs/ | L1 | Chart、values、`install/upgrade/rollback` |
| 目录补丁 | [Kustomize](https://kubectl.docs.kubernetes.io/references/kustomize/) | L1 | 与 Helm 对照：overlay vs 模板 |
| 按 CPU/内存扩缩 | [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) | L1 | GPU/队列场景转到 KEDA（手册已有） |
| 工作负载安全基线 | [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) | L1 | privileged / baseline / restricted；Secret 不只是 base64 |
| 探针与排障 | 官方 probes 任务 + [Debug Running Pods](https://kubernetes.io/docs/tasks/debug/) | L1 | 字段以官方为准；图解用心智（learnkube、jvns，手册已有） |
| 声明存储 | https://kubernetes.io/docs/concepts/storage/ | L1 | 会写 PVC/StorageClass；不搭建 Ceph |
| 北向流量标准 | https://gateway-api.sigs.k8s.io/ | L1–L2 | 先 HTTPRoute；Ingress 当仍常见的旧接口 |
| 平台用策略拦了什么 | [ValidatingAdmissionPolicy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/) · Kyverno/OPA（手册已有） | L2 | 会读拒绝原因、会提豁免；不写全集群策略 |

版本特性（In-Place Pod Resize、Native Sidecar、DRA、Metrics API）继续用手册的发布博客 + KEP。写进方案时钉集群小版本：考试环境跟随上游约 4–8 周，公司集群往往更慢，以平台实际版本为准。

**检索口令**：`CKAD curriculum pdf`、`helm rollback`、`pod security restricted`、`gateway API HTTPRoute`。

---

## 4. LLM 应用（补 Part 2：提示词、仓库、评估、安全）

手册已覆盖 LangGraph / LlamaIndex / Dify / MCP / A2A / Langfuse / 框架演进。业务开发还缺：**官方提示词、模型仓库、指标定义、应用安全**。

| 你要做的事 | 权威入口 | 深度 | 检索提示 |
|---|---|---|---|
| 提示词怎么写才算有依据 | [OpenAI Cookbook](https://cookbook.openai.com/) · [Anthropic prompting](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering) | L1 | 厂商官方 guide，不用无出处「神 prompt」 |
| 对哪一种 HTTP | [OpenAI API](https://platform.openai.com/docs) + 各引擎「OpenAI compatible」章节 | L1 | 自建推理默认按兼容 API 对接 |
| 选模型、看许可证与卡 | [HF Hub](https://huggingface.co/docs/hub/) · [Transformers](https://huggingface.co/docs/transformers) | L1 | Hub 管权重与卡片；Transformers 管生态共用的模型定义 |
| RAG 三阶段怎么划分 | https://arxiv.org/abs/2312.10997 | L1 | Naive / Advanced / Modular 用这篇，不转述博客分层 |
| GraphRAG 方法 vs 实现 | 论文 https://arxiv.org/abs/2404.16130 · https://github.com/microsoft/graphrag | L2 | lab 能跑再用实现；主线可先 Advanced RAG |
| LightRAG | https://arxiv.org/abs/2410.05779 | L2 | 与 GraphRAG 对照轻重，二选一深入 |
| 本地向量库 | [Chroma](https://docs.trychroma.com/) · [Qdrant](https://qdrant.tech/documentation/) · [Milvus](https://milvus.io/docs) · pgvector（Postgres 扩展文档） | L1 锁一个 | 主线建议 Chroma 或 pgvector，求本机可跑 |
| 忠实度/相关性定义 | https://docs.ragas.io/ · [DeepEval](https://docs.confident-ai.com/) | L1 | Langfuse/LangSmith（手册已有）负责把评估跑起来 |
| Agent 基石 | ReAct https://arxiv.org/abs/2210.03629 | L1 | 工具循环的原始定义 |
| AutoGen 之后用什么 | [Microsoft Agent Framework](https://learn.microsoft.com/en-us/agent-framework/overview/) | L2 | 与手册「已过气」表一致，新项目走继任者 |
| Agent/RAG 安全底线 | [OWASP GenAI LLM Top 10 2026](https://genai.owasp.org/resource/owasp-genai-llm-top-10-2026/) · [源码仓](https://github.com/GenAI-Security-Project/GenAI-LLM-Top10) | L1 | 至少：Prompt Injection、Excessive Agency、Unbounded Consumption、Vector and Embedding Weaknesses。2026-08-04 发布，用现行十条 |
| 国内模型对照 | Qwen / DashScope、DeepSeek 各自官方 API 文档 | L1 | 实验对象可用国产模型；协议仍以 OpenAI 兼容文档为准 |

**检索口令**：`RAGAS faithfulness`、`OWASP LLM Top 10 2026`、`Hugging Face model card license`、`chat completions compatible`。

---

## 5. AI 上云：接入推理、GPU、流量、可观测（补 Part 3 / 桥梁四层）

手册第一梯队（GPU/DRA → Kueue → vLLM → KServe）保持不变。下面补 PLAN 叙事里的**流量层、可观测层**，以及「向平台申请」时要叫得上名的对象。全部按接入写，不按装机写。

| 你要做的事 | 权威入口 | 深度 | 检索提示 |
|---|---|---|---|
| 模型路由 / 推理网关（标准） | https://gateway-api-inference-extension.sigs.k8s.io/ · [仓库](https://github.com/kubernetes-sigs/gateway-api-inference-extension) | L2 | 读 InferencePool：应用如何指后端。EPP 实现是平台的事 |
| 分布式推理菜谱（观察） | https://llm-d.ai/ · [docs](https://llm-d.ai/docs) | L2 | Well-Lit Paths 假设平台已提供栈。P/D 分离、前缀缓存路由用它解释「平台可能怎么做」。背景：[CNCF 欢迎 llm-d](https://www.cncf.io/blog/2026/03/24/welcome-llm-d-to-the-cncf-evolving-kubernetes-into-sota-ai-infrastructure/) |
| 声明式「跑某个模型」 | https://kaito-project.github.io/kaito/docs/ | L2 | Workspace / InferenceSet：填模型 ID 与 GPU 规格，对应申请单而不是装 GPU |
| 模型当 OCI 制品 | https://www.cncf.io/projects/modelpack/ | L2 | 只关心「模型进仓库、版本可回滚」 |
| 为什么 vLLM 省显存 | PagedAttention https://arxiv.org/abs/2309.06180 | L2 | 与 vLLM 文档成对：论文讲 KV 分页，文档讲启动参数 |
| 读 GPU 利用率含义 | [DCGM](https://docs.nvidia.com/datacenter/dcgm/) + 手册中 GPU Operator 文档 | L2 | 会读数；Exporter 谁装问平台 |
| 按队列/自定义指标扩缩 | https://keda.sh/docs/ | L1–L2 | ScaledObject；推理不要只 HPA CPU |
| 缩到零 | https://knative.dev/docs/serving/ | L2 | GPU 冷启动不友好，写进「何时不用」 |
| 多 Pod 协同推理原语 | https://github.com/kubernetes-sigs/lws （LeaderWorkerSet） | L2 | 与 JobSet 同类：star 少、对接平台时有用 |
| 这套集群算不算 AI 就绪 | https://github.com/cncf/k8s-ai-conformance | L2 | 和平台对话用（DRA、In-Place Resize、推理网关等）。读者不去给集群做认证 |
| Token / 延迟怎么埋点 | [semantic-conventions-genai](https://github.com/open-telemetry/semantic-conventions-genai) 的 `docs/gen-ai/` | L1–L2 | `gen_ai.usage.*`、agent spans。规范状态多为 Development，引用要写 status。解读可看 [OTel 2026 博文](https://opentelemetry.io/blog/2026/genai-observability) |
| 语言 SDK 怎么接 OTel | https://opentelemetry.io/docs/languages/python/ （Java 等同理） | L1 | 业务代码埋点；Collector 部署是平台的事 |
| 账单从哪看 | 模型官方定价页 · [OpenCost](https://www.opencost.io/) · [FinOps](https://www.finops.org/) | L2 | Token 用应用计量；GPU 分摊用平台成本视图。不编行业均价 |

推理优化关键词（TTFT、KV cache、前缀缓存、P/D 分离）用 vLLM/SGLang 文档的 architecture 页 + NVIDIA Kubernetes 博客（手册已有）。Dynamo 等停在 L2。

WG Serving 已于 2026-02 收官（[CNCF 说明](https://www.cncf.io/blog/2026/02/26/kubernetes-wg-serving-concludes-following-successful-advancement-of-ai-inference-support/)）。之后跟 SIG Network（Inference Extension）、SIG Apps（LWS）、DRA/设备管理，以及 llm-d 社区即可，不必再跟已解散 WG。

**检索口令**：`InferencePool CRD`、`llm-d well-lit path`、`KEDA scaler vLLM metrics`、`gen_ai.usage.input_tokens`。

---

## 6. 驾驭平台、GitOps、二开适配（补 Part 4）

手册有 Istio/Cilium/Argo/Backstage 产品链接。本角色更需要：**平台是什么、黄金路径怎么用、CRD 怎么读、适配器怎么写。**

| 你要做的事 | 权威入口 | 深度 | 检索提示 |
|---|---|---|---|
| 12-Factor 对照 K8s | https://12factor.net/ | L1 | Config→ConfigMap/Secret，日志→stdout，进程无状态→Deployment |
| 工作负载模式 | 《Kubernetes Patterns》（Ibryam & Huß）· https://www.ofbizian.com/ | L1–L2 | Sidecar / Ambassador / Adapter / Controller 以书中定义为准 |
| 内部平台该提供什么 | [CNCF Platforms White Paper](https://tag-app-delivery.cncf.io/whitepapers/platforms/) | L2 | **与平台团队对齐的共同语言**：自助、模板、黄金路径 |
| 评估公司平台成熟度 | [成熟度模型源稿](https://github.com/cncf/tag-app-delivery/blob/main/platforms-maturity-model/v1/index.md) · [社区页](https://cloudnativeplatforms.com/whitepapers/platform-eng-maturity-model/) | L2 | 用来判断「缺的是文档还是能力」，不是用来建 IDP |
| 用 Backstage 模板 | https://backstage.io/docs | L1 | Catalog / Software Template：创建服务、不部署 Backstage 本身 |
| GitOps 交应用 | Argo CD / Flux（手册已有）· [Argo Rollouts](https://argoproj.github.io/rollouts/) | L1 | 提交 PR、看 Application 健康、金丝雀；不搭 GitOps 控制面 |
| 网关：标准 vs 实现 | Gateway API（§3）· [Envoy Gateway](https://gateway.envoyproxy.io/) | L2 | 应用写 HTTPRoute；后面是 Envoy/Istio/nginx 由平台选 |
| 读官方 Operator 概念 | [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) · [Operator SDK 概念](https://sdk.operatorframework.io/) | L2 | 解释「为什么有这个 CRD、怎么用别人的」 |
| Python 调 K8s API | https://github.com/kubernetes-client/python | L1–L2 | 与本项目 AI 示例语言一致，适合写适配器、对账、调谐外围 |
| Go 调 K8s API | https://github.com/kubernetes/client-go | L2 | 公司二开若是 Go 再用 |
| 写小控制器（选修） | https://book.kubebuilder.io/ | L2 | 理解 reconcile 循环；不要求上生产 OperatorHub |
| 应用 SLO | [SRE 书 SLO 章](https://sre.google/sre-book/table-of-contents/) · https://openslo.com/ | L2 | 错误预算怎么跟发布节奏绑；OpenSLO 仅当平台已有该对象 |
| 镜像是否被平台信任 | https://www.sigstore.dev/ | L2 | 会看 Cosign 校验失败；策略谁配问平台 |
| CNCF 上「AI 工作组」口径 | https://tag-runtime.cncf.io/wgs/cnaiwg/ | L2 | 白皮书与会议材料，趋势判断用它 + CNCF 博客 |

Kagent 若写「引擎基于 Google ADK」，文档入口：https://google.github.io/adk-docs/（Kagent 本身见手册）。

**检索口令**：`CNCF platforms whitepaper golden path`、`kubernetes python client patch deployment`、`kubebuilder reconcile`。

---

## 7. 方案设计与持续学习（补 Part 6）

手册没有「方案文档怎么写」的源。下面都是方法论，直接服务于「能独立出技术方案」。

| 你要做的事 | 权威入口 | 深度 | 检索提示 |
|---|---|---|---|
| 记录一次选型 | https://adr.github.io/ | L1 | 一页：上下文 / 决策 / 后果 |
| 画给研发看的结构 | https://c4model.com/ | L1 | 容器图 + 部署图足够 |
| 检查单（运营/安全/可靠/性能/成本） | AWS Well-Architected 及对应 GenAI Lens（以 AWS 文档目录为准） | L2 | 用支柱提问，不必绑定 AWS 服务名 |
| 表达 Adopt / Trial / Hold | https://www.thoughtworks.com/radar | L2 | 学分类法；具体点名以本仓库日期与手册为准 |
| 演化式架构文风 | https://martinfowler.com/ | L2 | 引用具体文章 URL |
| 可靠性语言 | SRE 书（上） | L2 | SLO、错误预算、事故复盘结构 |
| 选修认证地图 | [cncf/curriculum](https://github.com/cncf/curriculum) 中 CGOA / CNPA / OTCA 等 | L2 | 主线仍是 CKAD；GitOps/平台工程/OTel 按岗选修 |

案例研读只引用可打开的原文（官方博客、RFC、事故报告）。前沿 WASM/边缘：[SpinKube](https://www.spinkube.dev/) · [WasmEdge](https://wasmedge.org/)，观察即可。

---

## 8. 经验之谈：可核验的补充作者

手册已有 Brendan Burns、Bilgin Ibryam、Julia Evans、Andrew Ng、Simon Willison。写「经验之谈」时可按主题加**能链到原文**的人；没有 URL 就只写观点不写人名（AGENTS.md 规范 7）。

| 主题 | 人 / 出处类型 |
|---|---|
| K8s 操作哲学 | Kelsey Hightower（公开演讲与仓库；引用标日期） |
| 可观测与生产调试 | Charity Majors（公开文章 / 演讲） |
| eBPF 能力边界 | Liz Rice（著作与 CNCF 演讲） |
| 分布式排障 | Cindy Sridharan（公开长文） |
| 架构表达 | Martin Fowler；SRE 书作者 |
| 框架演进 | Harrison Chase（LangChain 官方博客）；Jerry Liu（LlamaIndex 官方） |
| 推理机制 | PagedAttention 论文作者 Kwon et al.（用论文） |

---

## 9. 按学习阶段的最小打开清单

**阶段一（能干活）**  
kind 文档 → kubectl cheatsheet → kubernetes.io/docs/tasks → Helm 文档 → PSS → Gateway API 概览 → CKAD curriculum PDF（对照练习）。

**阶段二（会做 AI）**  
OpenAI API + 厂商 prompt 文档 → HF Hub → RAG 综述论文 → RAGAS → OWASP LLM Top 10 2026 → vLLM 文档（手册）→ Kueue（手册）→ KEDA → OTel GenAI 仓库 → Inference Extension 概览。

**阶段三（懂架构）**  
Platforms 白皮书 → 12-factor → Kubernetes Patterns → python/client-go → SRE SLO 章 → ADR → 手册中的 Istio/GitOps **用户指南**（L2 停）→ llm-d / KAITO / AI Conformance（与平台对齐用）。

---

## 10. 权威链接速查（本文新增或强调）

| 主题 | URL |
|---|---|
| CKAD 考纲仓 | https://github.com/cncf/curriculum |
| CKAD 介绍 | https://www.cncf.io/training/certification/ckad/ |
| 考试 FAQ | https://docs.linuxfoundation.org/tc-docs/certification/faq-cka-ckad-cks.md |
| kind | https://kind.sigs.k8s.io/ |
| kubectl cheatsheet | https://kubernetes.io/docs/reference/kubectl/cheatsheet/ |
| Helm | https://helm.sh/docs/ |
| Gateway API | https://gateway-api.sigs.k8s.io/ |
| Inference Extension | https://gateway-api-inference-extension.sigs.k8s.io/ |
| PSS | https://kubernetes.io/docs/concepts/security/pod-security-standards/ |
| HPA | https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/ |
| 12-Factor | https://12factor.net/ |
| Platforms 白皮书 | https://tag-app-delivery.cncf.io/whitepapers/platforms/ |
| python client | https://github.com/kubernetes-client/python |
| client-go | https://github.com/kubernetes/client-go |
| kubebuilder | https://book.kubebuilder.io/ |
| OTel GenAI 规范仓 | https://github.com/open-telemetry/semantic-conventions-genai |
| OWASP LLM Top 10 2026 | https://genai.owasp.org/resource/owasp-genai-llm-top-10-2026/ |
| RAGAS | https://docs.ragas.io/ |
| OpenAI Cookbook | https://cookbook.openai.com/ |
| HF Hub | https://huggingface.co/docs/hub/ |
| llm-d | https://llm-d.ai/ |
| KAITO | https://kaito-project.github.io/kaito/docs/ |
| AI Conformance | https://github.com/cncf/k8s-ai-conformance |
| CNAI WG | https://tag-runtime.cncf.io/wgs/cnaiwg/ |
| PagedAttention | https://arxiv.org/abs/2309.06180 |
| RAG 综述 | https://arxiv.org/abs/2312.10997 |
| ReAct | https://arxiv.org/abs/2210.03629 |
| GraphRAG | https://arxiv.org/abs/2404.16130 |
| SRE 书 | https://sre.google/sre-book/table-of-contents/ |
| ADR | https://adr.github.io/ |
| C4 | https://c4model.com/ |
| Landscape | https://landscape.cncf.io/ |

手册原文中的 kubernetes.io、Kueue、vLLM、KServe、LangGraph、MCP、CNCF 报告等继续作主索引，本文不重复抄表。
