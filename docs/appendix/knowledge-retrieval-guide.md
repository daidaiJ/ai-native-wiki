# 知识检索指导手册（Knowledge Retrieval Guide）

> 用途：为 AI agent 与开发者提供权威知识检索路径。遇到问题时按主题域查对应来源，避免在低质量内容上浪费时间。
> 原则：**官方文档/官方博客优先**（事实与机制），**大牛博客补充**（架构视角与踩坑经验），**版本敏感内容以官方发布博客 + KEP 为准**。
> 读者定位：业务开发/中台开发 + 基础设施二开适配者（**不建设平台**，使用/适配/集成平台）。平台建设类文档一律停在 L2（见 §0.1 深度分级）。

---

## 0. 给 agent 的使用说明

1. **先定位主题域**（下表 1-12），按「检索入口」顺序查：官方文档 → 官方博客 → 大牛博客
2. **版本敏感问题**（如「某特性在哪个版本 GA」）：查 Kubernetes 发布博客 + KEP（github.com/kubernetes/enhancements）
3. **项目选型/生态判断**：查 CNCF 项目状态（毕业/孵化/沙箱）+ GitHub star 量级
4. **趋势与报告**：查 CNCF 白皮书/年度调查，不要依赖二手转述
5. **引用规范**：wiki 内容标注来源（项目名 + URL + 作者），见 AGENTS.md
6. **考纲/认证类问题**：只信 https://github.com/cncf/curriculum ，任何博客转述的考纲权重一律视为可能过期
7. **方法论类问题**（Agent 模式 / SLO / 架构权衡）：优先官方工程博客与免费官方书籍（Anthropic Engineering、sre.google、abseil.io），其次才是个人博客
8. **中文来源只用于概念入门**，事实、版本、数据一律回英文官方来源复核；中文社区内容不作「经验之谈」的引用依据

### 0.1 深度分级（每条来源标注阅读深度）

| 深度 | 你要达到 | 典型动作 |
|---|---|---|
| **L1 会用** | 能提交清单、能排自己的应用 | 填 YAML、调 OpenAI 兼容 API、看探针失败原因 |
| **L2 会选 / 会对齐** | 能判断用不用、能向平台提需求 | 「我们要 InferencePool / Kueue 队列 / OTel GenAI 字段」 |
| **L3 会建** | 实现或运维平台本身 | **本项目不展开**；正文最多写「找基础设施团队」 |

Istio 数据面、Cilium eBPF、Volcano 调度插件、GPU Operator 装机，一律停在 L2。

### 0.2 按问题类型选入口

| 问题 | 先打开 | 读到哪停 |
|---|---|---|
| 这个字段什么意思、YAML 怎么写 | 项目官方 docs，K8s 再用 `kubectl explain` | L1，能复制能跑 |
| 特性哪一版才能用 | Kubernetes 发布博客 + KEP | 记下 GA/默认启用，再写进方案 |
| 平台有没有这项能力 | 公司平台文档 → 对不上再查项目 docs 的「用户/应用」章节 | L2，整理成给平台的接口清单 |
| 指标/忠实度/token 含义 | 规范或评估库文档，不要只用 SaaS 产品页 | 定义以规范为准，产品只负责把数跑出来 |
| 安全底线 | OWASP LLM Top 10、K8s Pod Security Standards | 应用侧可落地的缓解，不写攻击步骤 |
| 趋势与采用率 | CNCF reports | 引用必须带调查年份 |

### 0.3 查 KEP（版本问题）

1. https://github.com/kubernetes/enhancements 搜特性名或编号（如 DRA、in-place resize）
2. 读 `keps/` 里 README 的 **status** 与 **milestone**
3. 对照同版本发布博客，确认 Alpha / Beta / GA、是否默认启用
4. 引用格式：KEP 编号 + 版本 + 博客日期

对本读者：KEP 用来回答「我们集群版本够不够、该不该向平台申请打开某特性」，不用来讲控制面实现。

### 0.4 查 CNCF 地位（选型）

1. https://landscape.cncf.io/ 看 Graduated / Incubating / Sandbox
2. 核对项目页接受日期、所属 SIG
3. **SIG 官方项目**（Kueue、JobSet、Gateway API）即使 star 低，也优先于高星玩具
4. 沙箱项目（llm-d、KAITO、ModelPack、Envoy AI Gateway）默认 L2「观察与对齐」：知道平台可能提供什么，主线实验仍用手册第一梯队（GPU/DRA 文档 → Kueue → vLLM → KServe）

star 只表示热度，标注日期后当附录；架构地位看成熟度 + SIG + 是否出现在平台目录里。

### 0.5 查论文（机制一句话）

arXiv abs 页优先（有版本历史）。写法：**论文解释为什么，官方 docs 解释现在怎么配。** 本项目常引的论文见 §11。

### 0.6 文档搬家

写作前若官方页写 `Moved` / `archived` / successor，改跟新仓库。GenAI 语义约定已从 `opentelemetry.io/docs/specs/semconv/gen-ai/` 迁到 [semantic-conventions-genai](https://github.com/open-telemetry/semantic-conventions-genai)，旧链接只作跳转。

### 0.7 典型问题检索路径速查

| 典型问题 | 检索路径 |
|---|---|
| CKAD 考纲现在是什么？ | cncf/curriculum → 对应证书目录最新版 |
| 怎么在平台上加一个自定义资源/能力？ | Kubebuilder book → controller-runtime → API conventions |
| 这个 AI 场景该用 workflow 还是 Agent？ | Anthropic Building Effective Agents → Lilian Weng 综述 |
| RAG / Agent 质量怎么评？ | RAGAS/promptfoo → OpenAI Cookbook 评估示例 → Langfuse 跑起来 |
| 怎么给服务定 SLO？ | SRE Book 第 2-4 章 → Workbook 落地案例 |
| 模型请求怎么路由 / 灰度？ | Gateway API Inference Extension → kgateway 实践 |
| AI 调用链怎么埋点？ | OTel GenAI semconv → Langfuse |
| 这个英文概念有没有靠谱中文解释？ | K8s 官方中文文档 → jimmysong.io（再回英文原文复核） |
| 本机搭 kind / 写 Dockerfile / 查 kubectl？ | §1 环境与应用交付 |
| 对标 CKAD / Helm / 探针 / HPA / 安全基线？ | §1 Kubernetes 核心 |
| 写 RAG/Agent / 评估质量 / 防提示注入？ | §5 LLM 应用 |
| 申请 GPU / 接推理 API / 看 token / 走网关？ | §4 AI 基础设施 |
| 跟平台团队对齐 / GitOps 交付 / 读 CRD / 写适配器？ | §6 K8s 二开与 API 编程 |
| 写技术方案 / 定 SLO / 做权衡？ | §7 架构与可靠性方法论 |
| 不确定该读多深、文档会不会过期？ | §0 检索约定 |

引用格式与 AGENTS.md 一致：

```text
> 来源：[项目或作者](URL) · 版本或日期：YYYY-MM / vX.Y · 验证环境：kind v0.x / K8s v1.x
```

---

## 1. Kubernetes 核心与演进

| 问题类型 | 检索入口 | 说明 |
|---|---|---|
| 概念/API 用法 | https://kubernetes.io/docs/ | 官方文档，CKAD 考试唯一允许查阅的文档 |
| 按任务找 YAML | https://kubernetes.io/docs/tasks/ | 比从概念页抄更接近日常和考试 |
| 中文官方文档 | https://kubernetes.io/zh-cn/docs/ | 官方中文翻译，概念入门与术语统一用；考试建议用英文版练检索速度 |
| 版本特性/新功能 | https://kubernetes.io/blog/ | 每个版本发布博客（如 v1.35 发布博客） |
| 特性提案/设计背景 | https://github.com/kubernetes/enhancements | KEP 提案，理解「为什么这么设计」 |
| 版本时间线 | https://kubernetes.io/releases/ | 版本节奏（每年 3 个版本） |
| 本机集群（项目默认） | https://kind.sigs.k8s.io/ | Quick Start；本地镜像用 `kind load docker-image` |
| 备选集群 | https://minikube.sigs.k8s.io/docs/ · https://docs.k3s.io/ | lab 脚本不要和 kind 混用组件假设 |
| 安装 kubectl | https://kubernetes.io/docs/tasks/tools/ | 与集群小版本大致对齐即可 |
| 高频命令 | https://kubernetes.io/docs/reference/kubectl/cheatsheet/ | CKAD 考场可查；日常排障也够用 |
| 镜像构建 | https://docs.docker.com/ | 多阶段构建走 Dockerfile best practices |
| 「镜像」和 Dockerfile 不是一回事 | https://github.com/opencontainers/image-spec | 扫 layout/manifest 即可，建立心智模型 |
| 运行时只需知道 CRI | https://kubernetes.io/docs/setup/production-environment/container-runtimes/ · https://containerd.io/docs/ | 不展开 CRI-O 运维 |
| 探针与排障 | 官方 probes 任务 + https://kubernetes.io/docs/tasks/debug/ | 字段以官方为准；图解用心智（learnkube、jvns） |
| 声明存储 | https://kubernetes.io/docs/concepts/storage/ | 会写 PVC/StorageClass；不搭建 Ceph |
| 按 CPU/内存扩缩 | https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/ | GPU/队列场景转到 KEDA（§4） |
| 工作负载安全基线 | https://kubernetes.io/docs/concepts/security/pod-security-standards/ | PSS：privileged / baseline / restricted；PSP 继任机制（1.25 移除） |
| 平台用策略拦了什么 | https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/ · Kyverno/OPA（§2） | L2：会读拒绝原因、会提豁免；不写全集群策略 |
| 北向流量标准 | https://gateway-api.sigs.k8s.io/ | 先 HTTPRoute；Ingress 当仍常见的旧接口 |
| Ingress Controller 事实标准 | https://kubernetes.github.io/ingress-nginx/ | ingress-nginx，动手章节默认选择 |
| 镜像扫描 | https://github.com/aquasecurity/trivy | Trivy，容器/依赖/IaC 扫描事实标准 |
| 镜像签名 | https://docs.sigstore.dev/ | Sigstore/cosign，会看验签失败；策略谁配问平台 |
| 供应链等级 | https://slsa.dev/ | SLSA 框架（理解级别即可）[进阶] |
| 制品发现 | https://artifacthub.io/ | Helm chart/OPA 等 CNCF 制品中心 |
| 集群原理（进阶） | https://github.com/kelseyhightower/kubernetes-the-hard-way | Kelsey Hightower 的 Kubernetes The Hard Way，平台使用者按需读 |
| 架构视角 | https://brendandburns.com/ | Brendan Burns（K8s 联合创始人），AI 与云原生结合主题 |
| 可视化图解/排障 | https://learnkube.com/blog | learnk8s 更名，网络路径/排障/requests-limits 图解 |
| 调试与网络图解 | https://jvns.ca/ | Julia Evans，调试/网络/容器原理 |

**版本纪律**：版本特性（In-Place Pod Resize、Native Sidecar、DRA、Metrics API）以发布博客 + KEP 为准；写进方案时钉集群小版本——考试环境跟随上游约 4–8 周，公司集群往往更慢，以平台实际版本为准。

**CKAD 备考**（对应附录 A）：

| 项 | 入口 | 用法 |
|---|---|---|
| 考纲原文（唯一权威） | https://github.com/cncf/curriculum | CKAD/CKA/CKS 考纲仓库，权重对照以此为准，标注版本日期（撰写时 v1.35） |
| 官方考试页 | https://www.cncf.io/training/certification/ckad/ | 大纲与形式的唯一权威源；大纲会更新，引用时标注读取日期 |
| 考试 FAQ | https://docs.linuxfoundation.org/tc-docs/certification/faq-cka-ckad-cks.md | 及格线（CKAD 66%）与模拟器说明 |
| 官方模拟环境 | https://killer.sh/ | 考票附赠的官方模拟（KillerCoda 同源）；考前一周限时完整做一遍，练「查文档速度」不是「背命令」 |
| 免费练习题库 | https://github.com/dgkanatsios/CKAD-exercises | 按考点分类免费刷题 |

备考纪律：官方 curriculum 每次考试版本可能调整，模拟题资源（尤其第三方课程）可能出现考点漂移，一切以官方页当日内容为准。

**关键演进节点（2024-2026）**：Gateway API 取代 Ingress（v1.6 TCPRoute/UDPRoute 转正）、In-Place Pod Resize（1.35 GA）、DRA（1.35 GA）、Native Sidecar、Metrics API（1.37 GA）

## 2. 云原生生态（网络/可观测/GitOps/服务网格/安全/平台工程）

| 主题 | 检索入口 | 说明 |
|---|---|---|
| 生态总览/趋势 | https://www.cncf.io/blog/ | CNCF 官方博客，2026 年主题已转向 AI |
| 年度调查/报告 | https://www.cncf.io/reports/ | 云原生调查、项目旅程报告 |
| 生态全景图 | https://landscape.cncf.io/ | 选型时先看项目在图里的位置与分类 |
| 生态周报 | https://www.cncf.io/kubeweekly/ | KubeWeekly，跟踪生态动态成本最低的方式 |
| 服务网格 | https://istio.io/ · https://linkerd.io/ | Istio Ambient（sidecar-less）vs Linkerd |
| 可观测性 | https://opentelemetry.io/ · https://prometheus.io/ · https://grafana.com/ | OTel 标准 + LGTM 栈 |
| GitOps | https://argoproj.github.io/cd/ · https://fluxcd.io/ | ArgoCD vs Flux 双雄 |
| 渐进式交付 | https://argoproj.github.io/rollouts/ | Argo Rollouts：金丝雀/蓝绿（L1 使用，不搭控制面） |
| eBPF 网络/安全 | https://cilium.io/ · https://tetragon.io/ | Cilium/Hubble/Tetragon |
| Serverless/弹性 | https://knative.dev/ · https://keda.sh/ | Knative + KEDA 事件驱动扩缩容 |
| 策略/安全 | https://kyverno.io/ · https://www.openpolicyagent.org/ | Kyverno（K8s 原生）vs OPA（Rego） |
| 供应链安全 | https://www.sigstore.dev/ · https://aquasecurity.github.io/trivy/ · https://slsa.dev/ | 签名/扫描/分级框架，了解能力边界即可（L2） |
| 平台工程 | https://platformengineering.org/ · https://backstage.io/ | IDP 黄金路径、Backstage |
| 内部平台该提供什么 | https://tag-app-delivery.cncf.io/whitepapers/platforms/ | CNCF Platforms 白皮书：**与平台团队对齐的共同语言**（自助、模板、黄金路径） |
| 平台成熟度评估 | https://github.com/cncf/tag-app-delivery/blob/main/platforms-maturity-model/v1/index.md · https://cloudnativeplatforms.com/whitepapers/platform-eng-maturity-model/ | 判断「缺的是文档还是能力」，不是用来建 IDP |
| 设计模式 | https://www.ofbizian.com/ | Bilgin Ibryam：《Kubernetes Patterns》《The Platform Engineering Book》 |
| 应用设计基线 | https://12factor.net/ | 12-Factor 对照 K8s：Config→ConfigMap/Secret，日志→stdout，进程无状态→Deployment |
| 网关：标准 vs 实现 | https://gateway.envoyproxy.io/ | 应用写 HTTPRoute；后面是 Envoy/Istio/nginx 由平台选 |
| CNCF AI 工作组 | https://tag-runtime.cncf.io/wgs/cnaiwg/ | 白皮书与会议材料，趋势判断用它 + CNCF 博客 |

## 3. 应用打包与分发（Helm / Kustomize）

| 问题类型 | 检索入口 | 说明 |
|---|---|---|
| Helm 官方 | https://helm.sh/docs/ | Chart 模板/values/仓库，分发事实标准；考场可查 |
| Kustomize | https://kustomize.io/ · https://kubectl.docs.kubernetes.io/references/kustomize/ | K8s 原生 overlay，GitOps 配套 |
| 制品发现 | https://artifacthub.io/ | 找现成 chart（Bitnami 系质量较稳） |

选型判断：自有应用 + GitOps → Kustomize 更贴近声明式；分发/安装第三方中间件 → Helm 生态无可替代。差异规模是分界线（环境差异超过约三成时 overlay 会失控）。

## 4. AI 基础设施（GPU 调度/训练/推理/MLOps/网关）

| 问题类型 | 检索入口 | 说明 |
|---|---|---|
| GPU 调度机制 | https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/ | Device Plugin 机制，2 小时可读完 |
| DRA（新标准） | https://kubernetes.io/docs/concepts/resource-management/dynamic-resource-allocation/ | 1.35 GA，GPU 管理演进方向 |
| 作业排队/配额 | https://kueue.sigs.k8s.io/ | Kueue，SIG 官方项目，学习性价比最高 |
| 批量调度 | https://volcano.sh/ | Volcano，Gang 调度，国内大厂常用 |
| GPU 共享 | https://github.com/Project-HAMi/HAMi | HAMi，显存切分/利用率提升 |
| GPU 一键装机 | https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html | NVIDIA GPU Operator（L2，装机是平台的事） |
| GPU 指标 | https://github.com/NVIDIA/dcgm-exporter · https://docs.nvidia.com/datacenter/dcgm/ | DCGM Exporter，GPU→Prometheus 标准方式；会读数，Exporter 谁装问平台 |
| 分布式训练 | https://docs.ray.io/en/latest/cluster/kubernetes/index.html · https://www.kubeflow.org/ | KubeRay + Kubeflow 双主流 |
| 训练作业 API | https://jobset.sigs.k8s.io/ · https://docs.pytorch.org/docs/stable/elastic/kubernetes.html | JobSet + TorchElastic |
| 多 Pod 协同推理原语 | https://github.com/kubernetes-sigs/lws | LeaderWorkerSet，与 JobSet 同类：star 少、对接平台时有用 |
| 推理引擎 | https://docs.vllm.ai/en/latest/ · https://docs.sglang.io/ | vLLM（事实标准）vs SGLang |
| 推理平台 | https://kserve.github.io/website/ | KServe，serverless/金丝雀 |
| 推理优化（NVIDIA） | https://nvidia.github.io/TensorRT-LLM/ | TensorRT-LLM [进阶] |
| 模型路由标准 | https://gateway-api-inference-extension.sigs.k8s.io/ · https://github.com/kubernetes-sigs/gateway-api-inference-extension | 读 InferencePool：应用如何指后端；EPP 实现是平台的事 |
| AI 原生网关实现 | https://kgateway.dev/ | kgateway（CNCF），Inference Extension 参考实现 |
| 统一网关/成本治理 | https://docs.litellm.ai/ ⏳ · https://github.com/envoyproxy/ai-gateway | LiteLLM（统一代理+key 治理+预算）/ Envoy AI Gateway（CNCF 沙箱） |
| 声明式「跑某个模型」 | https://kaito-project.github.io/kaito/docs/ | KAITO：Workspace/InferenceSet，填模型 ID 与 GPU 规格，对应申请单而不是装 GPU |
| 模型当 OCI 制品 | https://www.cncf.io/projects/modelpack/ | ModelPack：只关心「模型进仓库、版本可回滚」 |
| 分布式推理编排 | https://github.com/llm-d/llm-d · https://llm-d.ai/docs | llm-d（CNCF 沙箱）：P/D 分离、前缀缓存路由，用它解释「平台可能怎么做」 |
| disaggregated inference | https://github.com/NVIDIA/dynamo ⏳ | PLAN §5.2 点名的 Dynamo 上游 |
| 按队列/自定义指标扩缩 | https://keda.sh/docs/ | ScaledObject；推理不要只 HPA CPU |
| 缩到零 | https://knative.dev/docs/serving/ | GPU 冷启动不友好，写进「何时不用」 |
| 集群 AI 就绪度 | https://github.com/cncf/k8s-ai-conformance | 和平台对话用（DRA、In-Place Resize、推理网关等）；读者不去给集群做认证 |
| Token/延迟埋点 | https://github.com/open-telemetry/semantic-conventions-genai 的 `docs/gen-ai/` | OTel GenAI 语义约定：`gen_ai.usage.*`、agent spans；规范状态多为 Development，引用要写 status |
| LLM 推理追踪 | https://github.com/arize-ai/openinference ⏳ | OpenInference（Langfuse/Arize 采用），定位建议核验 |
| 语言 SDK 接 OTel | https://opentelemetry.io/docs/languages/python/ | 业务代码埋点；Collector 部署是平台的事 |
| 账单从哪看 | 模型官方定价页 · https://www.opencost.io/ · https://www.finops.org/ | Token 用应用计量；GPU 分摊用平台成本视图；不编行业均价 |
| MLOps | https://mlflow.org/ | MLflow 开源首选 |
| NVIDIA 工程实践 | https://developer.nvidia.com/blog/tag/kubernetes/ | 16+ 篇 K8s AI 基础设施文章（Dynamo、disaggregated inference 等） |

**学习性价比排序（云原生工程师进入 AI 领域）**：
- 第一梯队：K8s GPU/DRA 官方文档 → Kueue → vLLM → KServe（直接复用 K8s 知识）
- 第二梯队：Ray/KubeRay → JobSet+TorchElastic → HAMi → MLflow
- 第三梯队：Volcano → Kubeflow → SGLang/TGI → Ollama → BentoML（特定场景再学）

**生态动态**：WG Serving 已于 2026-02 收官（[CNCF 说明](https://www.cncf.io/blog/2026/02/26/kubernetes-wg-serving-concludes-following-successful-advancement-of-ai-inference-support/)）。之后跟 SIG Network（Inference Extension）、SIG Apps（LWS）、DRA/设备管理，以及 llm-d 社区即可，不必再跟已解散 WG。

## 5. LLM 应用与 Agent

| 问题类型 | 检索入口 | 说明 |
|---|---|---|
| 框架演进/选型 | https://www.langchain.com/blog/ | LangChain 官方博客（Deep Agents、框架对比） |
| Agent 编排 | https://langchain-ai.github.io/langgraph/ | LangGraph（图状态机，2026 首选编排框架） |
| RAG/数据框架 | https://docs.llamaindex.ai/ | LlamaIndex（文档 Agent/Agentic RAG） |
| 低代码平台 | https://docs.dify.ai/ | Dify（企业落地/私有化首选） |
| MCP 协议 | https://modelcontextprotocol.io/ | MCP 规范（2026-07-28 版，含 MCP Apps） |
| A2A 协议 | https://a2a-protocol.org/ | Agent 互操作标准（Linux Foundation 托管） |
| OpenAI Agents SDK | https://openai.github.io/openai-agents-python/ | 官方轻量多智能体框架 |
| AgentOps/评估 | https://langfuse.com/ · https://docs.langchain.com/ | Langfuse（开源首选）/ LangSmith（商业） |
| Agent 设计模式 | https://www.anthropic.com/engineering/building-effective-agents | Anthropic「Building Effective Agents」：workflow vs agent 的权威区分 |
| Agent 构建（OpenAI 一手） | https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf ⏳ | 官方 Agent 实践指南 |
| OpenAI 实战配方 | https://cookbook.openai.com/ | RAG / 评估 / 工具调用的官方示例集 |
| Prompt 系统教程 | https://www.promptingguide.ai/ | DAIR.AI 开源指南（含中文），术语统一性好 |
| 厂商 prompt 指南 | https://platform.openai.com/docs/guides/prompt-engineering · https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering | 两家官方工程指南，不用无出处「神 prompt」 |
| 对哪一种 HTTP | https://platform.openai.com/docs + 各引擎「OpenAI compatible」章节 | 自建推理默认按兼容 API 对接 |
| 选模型/许可证 | https://huggingface.co/docs/hub/ · https://huggingface.co/docs/transformers | Hub 管权重与卡片；Transformers 管生态共用的模型定义 |
| RAG 演进理论 | https://arxiv.org/abs/2312.10997 | RAG Survey（Naive/Advanced/Modular 三阶段） |
| GraphRAG 方法 vs 实现 | https://arxiv.org/abs/2404.16130 · https://github.com/microsoft/graphrag | lab 能跑再用实现；主线可先 Advanced RAG |
| LightRAG | https://arxiv.org/abs/2410.05779 | 与 GraphRAG 对照轻重，二选一深入 |
| 本地向量库 | https://docs.trychroma.com/ · https://qdrant.tech/documentation/ · https://milvus.io/docs · https://github.com/pgvector/pgvector | 主线建议 Chroma 或 pgvector，求本机可跑；pgvector 中台优先 |
| Embedding 选型 | https://huggingface.co/spaces/mteb/leaderboard | MTEB 权威基准；中文场景看 C-Retrieval/Retrieval 任务 |
| RAG 质量评估 | https://docs.ragas.io/ · https://docs.confident-ai.com/ | RAGAS 四指标（faithfulness/relevancy/context precision/recall）；DeepEval 备选 |
| LLM 回归测试 | https://www.promptfoo.dev/docs/ ⏳ | promptfoo，CI 级回归：改 prompt 必须跑评估 |
| LLM 安全边界 | https://genai.owasp.org/ · https://github.com/GenAI-Security-Project/GenAI-LLM-Top10 | OWASP Top 10 for LLM Applications（2026-08-04 发布 2026 版），安全评审事实基线 |
| 提示注入一手跟踪 | https://simonwillison.net/tags/prompt-injection/ | Simon Willison 专题页（直接 vs 间接注入划分） |
| LLM 系统设计 | https://huyenchip.com/ | Chip Huyen（《AI Engineering》作者），ML 系统设计视角 |
| Agent/LLM 原理综述 | https://lilianweng.github.io/ | Lilian Weng（Lil'Log），Agent / Prompt / RLHF 长文综述 |
| AutoGen 之后用什么 | https://learn.microsoft.com/en-us/agent-framework/overview/ | Microsoft Agent Framework，新项目走继任者 |
| 国内模型对照 | Qwen / DashScope、DeepSeek 各自官方 API 文档 | 实验对象可用国产模型；协议仍以 OpenAI 兼容文档为准 |
| K8s×Agent | https://github.com/kagent-dev/kagent | Kagent（CNCF，CRD 声明式 Agent）；引擎基于 Google ADK：https://google.github.io/adk-docs/ |
| AI 网关 | https://github.com/envoyproxy/ai-gateway | Envoy AI Gateway（provider 路由/fallback） |
| 大牛博客 | https://www.deeplearning.ai/the-batch/ · https://simonwillison.net/ | Andrew Ng The Batch 周报 / Simon Willison（MCP 最早解读） |

**框架演进速查（2026 状态）**：
- ✅ 值得投入：LangGraph、Dify、OpenAI Agents SDK、MCP/A2A 协议、Langfuse、GraphRAG 系（GraphRAG/LightRAG/RAGFlow）、Kagent、Envoy AI Gateway
- ❌ 已过气/被合并：OpenAI Swarm（→Agents SDK）、AutoGen 0.2（→0.4 → Microsoft Agent Framework）、LangChain 0.x（→1.0 统一 LangGraph）、Haystack 1.x（→2.x）
- ⚠️ 演进中：AutoGen 与 Microsoft Agent Framework 并存，新项目以 Agent Framework 为主

**Agent 架构模式学习顺序**：ReAct（基石，arXiv 2210.03629）→ Plan-and-Execute → 多智能体（LangGraph/CrewAI）→ Deep Agents（2026 深度推理）

## 6. K8s 二开与 API 编程（对应 Part 4.6 / R28）

> 使用者视角要点：重点是「怎么用 CRD/Operator 集成平台能力、怎么基于 K8s API 做定制」，不深入 controller 内部实现与分布式原理。

| 问题类型 | 检索入口 | 说明 |
|---|---|---|
| 客户端库选型 | https://kubernetes.io/docs/reference/using-api/client-libraries/ | 官方客户端列表 |
| Python 调 K8s API | https://github.com/kubernetes-client/python | 与本项目 AI 示例语言一致，适合写适配器、对账、调谐外围 |
| client-go | https://github.com/kubernetes/client-go | informer/lister/workqueue 模式看源码 examples |
| Controller 样例 | https://github.com/kubernetes/sample-controller | informer+workqueue 最短可跑路径 |
| Controller 框架 | https://github.com/kubernetes-sigs/controller-runtime | 事实标准的控制器框架，二开绕不开 |
| Operator 脚手架 | https://book.kubebuilder.io/ | Kubebuilder 官方 book，**二开入门最佳主线**——先脚手架跑通，再回头看原理 |
| Operator SDK | https://sdk.operatorframework.io/ | Operator 脚手架（Red Hat 维护） |
| API 设计规范 | https://github.com/kubernetes/community/tree/master/sig-architecture | SIG Architecture 的 API conventions，设计 CRD 字段前必读 |
| Operator 概念 | https://kubernetes.io/docs/concepts/extend-kubernetes/operator/ | 解释「为什么有这个 CRD、怎么用别人的」 |
| 参考书 | 《Programming Kubernetes》O'Reilly | client-go/API machinery 成体系参考（英文）[进阶] |

**检索路径**：二开问题 → Kubebuilder book（入门跑通）→ controller-runtime（深入）→ API conventions（设计 CRD 时）。不建议从裸写 client-go 开始学。

## 7. 架构与可靠性方法论（对应 Part 6 / R3）

| 问题类型 | 检索入口 | 说明 |
|---|---|---|
| SLO/可靠性源头 | https://sre.google/books/ | Google SRE Book + Workbook，免费全文 |
| 交付效能 | https://dora.dev/ | DORA 年度报告 |
| 架构描述 | https://c4model.com/ | C4 模型（Context/Container/Component/Code 四层），容器图+部署图足够 |
| 决策记录 | https://adr.github.io/ | ADR 轻量实践：一页上下文/决策/后果 |
| 架构演进观点 | https://martinfowler.com/ | Martin Fowler，微服务/演进式架构，引用具体文章 URL |
| 大厂架构实录 | https://newsletter.pragmaticengineer.com/ | Gergely Orosz（The Pragmatic Engineer），真实工程决策复盘 |
| 工程方法论 | https://abseil.io/resources/swe-book | 《Software Engineering at Google》免费版 |
| 检查单（运营/安全/可靠/性能/成本） | AWS Well-Architected 及对应 GenAI Lens（以 AWS 文档目录为准） | 用支柱提问，不必绑定 AWS 服务名 |
| 技术雷达分类法 | https://www.thoughtworks.com/radar | 学 Adopt/Trial/Hold 分类法；具体点名以本仓库日期与手册为准 |
| 系统设计通识 | https://github.com/donnemartin/system-design-primer | 开源教材（英文）[进阶] |
| 案例研读素材 | https://github.com/GoogleCloudPlatform/microservices-demo | Google 官方微服务演示应用，适合作首个研读案例 |
| 架构访谈 | https://queue.acm.org/ | ACM Queue 一手叙事，深且慢 |
| AI 工程访谈 | https://www.latent.space/ | Latent Space（swyx & Alessio），工程视角而非学术 |
| 中文架构报道 | https://www.infoq.cn/ | 架构/AI 落地中文一手报道 |
| 前沿 WASM/边缘 | https://www.spinkube.dev/ · https://wasmedge.org/ | 观察即可 |
| 选修认证地图 | https://github.com/cncf/curriculum 中 CGOA / CNPA / OTCA 等 | 主线仍是 CKAD；GitOps/平台工程/OTel 按岗选修 |

案例研读纪律：只引用可打开的原文（官方博客、RFC、事故报告）。建议每季度从 ACM Queue / Latent Space / InfoQ 各挑 1 篇做拆解（问题 → 约束 → 决策 → 教训）。

## 8. 趋势与权威报告

| 报告 | 链接 | 价值 |
|---|---|---|
| CNCF Cloud Native AI Whitepaper | https://www.cncf.io/reports/cloud-native-artificial-intelligence-whitepaper/ | AI 云原生定调（2024-03） |
| CNCF 年度云原生调查 | https://www.cncf.io/reports/ | 采用率数据（84% 生产用 K8s） |
| Kubernetes AI Day 报告 | https://www.cncf.io/reports/kubernetes-ai-day-europe/ | AI on K8s 实践案例 |
| CNCF 沙箱项目列表 | https://www.cncf.io/sandbox-projects/ | 前沿项目雷达（KAITO/ModelPack/llm-d 等） |
| KubeWeekly | https://www.cncf.io/kubeweekly/ | 生态动态跟踪成本最低的方式 |
| KubeCon China | events.linuxfoundation.org ⏳ | 国内 AI on K8s 实践最密集来源，建议按年收录议程 |

**中文入口**（概念入门与术语统一；事实/版本/数据回英文官方复核）：

| 来源 | 链接 | 说明 |
|---|---|---|
| K8s 官方中文文档 | https://kubernetes.io/zh-cn/docs/ | 附录 C 术语表优先采用其译法，保持全书术语一致 |
| Jimmy Song（宋净超） | https://jimmysong.io/ | 云原生社区创始人，Istio/云原生中文资料最全 |
| 张磊《深入剖析 Kubernetes》 | https://time.geekbang.org/column/intro/116 | K8s 原理中文讲解标杆（付费，「经验之谈」可引用公开部分） |
| 云原生社区 | https://cloudnative.to/ ⚠️待验证 | 中文云原生社区，Istio Handbook 等译作 |
| InfoQ 中文 | https://www.infoq.cn/ | 架构/AI 落地中文一手报道 |

**业务与行业趋势**（服务「业务驱动」原则：业务痛点与场景实例须有来源支撑，禁止编造）：

| 来源 | 链接 | 说明 |
|---|---|---|
| Gartner / IDC / Forrester | gartner.com · idc.com · forrester.com | 企业 IT/AI 投入与趋势预测（付费墙，只引公开摘要，标注年份） |
| 麦肯锡 AI 报告 | https://www.mckinsey.com/capabilities/quantumblack/our-insights | 企业 AI 采用率与价值调研（免费，引用标注年份） |
| CNCF 年度调查 | https://www.cncf.io/reports/ | 云原生采用率（84% 生产用 K8s），业务侧证据 |
| 艾瑞咨询 / 亿欧 | https://www.iresearch.com.cn/ · https://www.iyiou.com/ | 国内行业报告（AI 应用/云服务市场），中文业务数据 |
| 36氪 / 虎嗅 | https://36kr.com/ · https://www.huxiu.com/ | 国内企业 AI 落地案例报道（作场景实例线索，不作数据来源） |

使用纪律：业务痛点/场景实例写进正文时，数据与趋势引用须带来源 + 年份；案例报道类只作场景线索，量化数据回行业报告/官方财报核实。

## 9. 项目生态地位速查（2026-08 验证）

| 项目 | star 量级 | 状态 | 一句话定位 |
|---|---|---|---|
| Ollama | ~180k | 开源 | 本地一键跑模型 |
| Dify | ~154k | 开源 | 低代码 LLM 应用平台（企业落地首选） |
| LangChain | ~145k | 开源 | LLM 应用框架（1.0 统一 LangGraph） |
| vLLM | ~90k | 开源 | 推理引擎事实标准 |
| RAGFlow | ~89k | 开源 | 深度文档解析 + RAG |
| AutoGen | ~61k | 开源 | 多智能体框架（→Agent Framework） |
| CrewAI | ~58k | 开源 | 角色扮演多智能体（入门友好） |
| LlamaIndex | ~52k | 开源 | RAG/文档 Agent 数据框架 |
| LangGraph | ~41k | 开源 | 图状态机 Agent 编排（2026 首选） |
| LightRAG | ~39k | 开源 | 轻量图 RAG |
| Istio | ~38k | CNCF 毕业 | 服务网格事实标准 |
| GraphRAG | ~36k | 开源 | 知识图谱 RAG（微软） |
| Langfuse | ~34k | 开源 | AgentOps 开源首选 |
| SGLang | ~33k | 开源 | vLLM 最强竞争者 |
| OpenAI Agents SDK | ~29k | 开源 | 官方轻量多智能体框架 |
| MLflow | ~28k | 开源 | MLOps 开源首选 |
| A2A | ~26k | Linux Foundation | Agent 互操作协议 |
| Haystack | ~26k | 开源 | 生产级 pipeline 编排 |
| Cilium | ~25k | CNCF 毕业 | eBPF 网络/安全平台 |
| Swarm | ~22k | 已过气 | 被 Agents SDK 取代（仅学概念） |
| Kubeflow | ~16k | CNCF 孵化 | 端到端 MLOps 平台 |
| TGI | ~11k | 开源 | HF 生态推理服务 |
| KEDA | ~10k | CNCF 毕业 | 事件驱动扩缩容 |
| MCP | ~9k | 事实标准 | Agent 连接工具协议（Anthropic） |
| BentoML | ~9k | 开源 | AI 应用产品化 |
| K8sGPT | ~8k | CNCF 沙箱 | AI 运维诊断 |
| kubectl-ai | ~7.6k | 开源 | Google AI 版 kubectl |
| KServe | ~6k | 开源 | 推理平台层标准 |
| Volcano | ~6k | CNCF 孵化 | 批量调度（Gang） |
| Knative | ~6k | CNCF 毕业 | Serverless（scale-to-zero） |
| HAMi | ~4.5k | 沙箱候选 | GPU 共享 |
| Kagent | ~3.6k | CNCF 项目 | K8s 原生 Agent 框架（CRD+MCP） |
| Kueue | ~2.9k | CNCF 孵化 | 作业排队/配额标准 |
| KubeRay | ~2.7k | 开源 | Ray on K8s |
| Envoy AI Gateway | ~2k | CNCF 沙箱 | GenAI 统一网关 |
| KubeAI | ~1.3k | 开源 | 最简 AI Inference Operator |
| JobSet | ~342 | SIG 项目 | 训练作业底层 API（star 少地位高） |

## 10. 经验之谈人物/信源表（服务章节模板第 6 节 / R20）

| 人物/信源 | 领域 | 适用章节 |
|---|---|---|
| Brendan Burns | K8s 联合创始人，AI 与云原生结合 | Part 1 / 4 |
| Kelsey Hightower | K8s 操作哲学、KTHW | Part 1 / 4 |
| Bilgin Ibryam | Kubernetes Patterns、平台工程 | Part 4 |
| Julia Evans | 调试/网络/容器原理图解 | Part 1 |
| Andrew Ng | AI 教育、The Batch 周报 | Part 0 / 2 |
| Simon Willison | 提示注入一手研究、MCP 最早解读 | Part 2.7 |
| Chip Huyen | ML/LLM 系统与工程化（《AI Engineering》O'Reilly 2025） | Part 2.1 / 5.2 |
| Eugene Yan | LLM 应用与评估实践 | Part 2.5 |
| Hamel Husain | evals 实战（「AI 产品需要评估」） | Part 2.5 |
| Lilian Weng | Agent/Prompt/RLHF 长文综述 | Part 2 / 5.3 |
| Harrison Chase / Jerry Liu | LangChain / LlamaIndex 官方博客（框架演进） | Part 2.2 |
| Kwon et al. | PagedAttention 论文作者（用论文） | Part 3.2 |
| Charity Majors | 可观测与生产调试 | Part 3.4 / 4.2 |
| Liz Rice | eBPF 能力边界 | Part 4.4 |
| Cindy Sridharan | 分布式排障 | Part 4 / 6 |
| Martin Fowler | 架构表达、演进式架构 | Part 4.7 / 6.1 |
| Gergely Orosz | 真实工程决策复盘 | Part 6.3 |
| Latent Space（swyx & Alessio） | AI 工程访谈 | Part 6.3 |
| 张磊 | K8s 原理中文讲解 | Part 1 |
| Jimmy Song | 云原生中文资料 | Part 1 / 4 |

引用纪律：以上仅收录真实存在的人物与站点；具体观点写入「经验之谈」时须回原文核对并给链接，拿不准只写观点不署名（AGENTS.md 规范 7）。

## 11. 经典论文锚点

| 论文 | arXiv | 对应章节 |
|---|---|---|
| Attention Is All You Need | https://arxiv.org/abs/1706.03762 | Part 2.1（Transformer 起点文档） |
| ReAct | https://arxiv.org/abs/2210.03629 | Part 2.4（工具循环原始定义） |
| PagedAttention（vLLM） | https://arxiv.org/abs/2309.06180 | Part 3.2（推理引擎核心论文） |
| RAG Survey | https://arxiv.org/abs/2312.10997 | Part 2.3（Naive/Advanced/Modular） |
| GraphRAG | https://arxiv.org/abs/2404.16130 | Part 2.3（方法 vs 实现） |
| LightRAG | https://arxiv.org/abs/2410.05779 | Part 2.3（与 GraphRAG 对照） |

作用：训练「读一手来源」习惯，框架文档只告诉你怎么用，论文告诉你为什么这样设计。

## 12. 按学习阶段的最小打开清单

**阶段一（能干活）**：kind 文档 → kubectl cheatsheet → kubernetes.io/docs/tasks → Helm 文档 → PSS → Gateway API 概览 → CKAD curriculum PDF（对照练习）。

**阶段二（会做 AI）**：OpenAI API + 厂商 prompt 文档 → HF Hub → RAG 综述论文 → RAGAS → OWASP LLM Top 10 2026 → vLLM 文档 → Kueue → KEDA → OTel GenAI 仓库 → Inference Extension 概览。

**阶段三（懂架构）**：Platforms 白皮书 → 12-factor → Kubernetes Patterns → python/client-go → SRE SLO 章 → ADR → 手册中的 Istio/GitOps **用户指南**（L2 停）→ llm-d / KAITO / AI Conformance（与平台对齐用）。

---

## 更新记录

- 2026-08-28：初版（云原生 + AI 基础设施 + LLM/Agent 三域，基于 web-researcher 调研报告，star 数据 2026-08-28 验证）
- 2026-08-28：整合 GLM / grok / kimi 三份 AI 增补（原文归档于 `meta/`）——新增主题域「应用打包与分发」「K8s 二开与 API 编程」「架构与可靠性方法论」；§0 增补检索约定（深度分级 L1/L2/L3、KEP/CNCF/论文查法、中文来源纪律、典型问题检索路径）；各域补 OWASP/RAGAS/MTEB/向量库/LiteLLM/DCGM/OTel GenAI/FinOps/Inference Extension/llm-d/KAITO 等入口；新增经验之谈人物表、经典论文锚点、学习阶段最小打开清单。⏳ 标注项待人工验证后去除标记。

**待人工验证清单（⏳）**：
- OpenAI building agents 指南 PDF 路径（cdn.openai.com/.../a-practical-guide-to-building-agents.pdf）
- promptfoo / litellm 文档具体路径
- NVIDIA Dynamo 仓库主地址
- OpenInference 的定位表述（Langfuse/Arize 采用）
- KubeCon China 年度议程链接
- cloudnative.to 可访问性
- CKAD 当前大纲版本与附赠模拟环境使用次数（以官方页为准）
- 手册中所有「验证环境：待验证」命令的本地 kind 实测