# kimi.md — 内容补强：权威来源增补与检索指南完善

> 日期：2026-08-28 · 性质：对现有文档（PLAN / SPEC / knowledge-retrieval-guide）的**补强**，可直接并入 `docs/appendix/knowledge-retrieval-guide.md`
> 链接可访问性：除标注「待验证」外，均于 2026-08-28 实测 HTTP 200

---

## 一、补强思路：对照定位找缺口

项目定位（REQUIREMENTS R2/R3/R27/R28/R29）：

- 读者是**业务/中台开发 + 基础设施二开适配者**，目标是成长为有架构视野的资深开发/架构师
- **驾驭平台、适配平台，不建设平台**——所有基础设施内容只讲能力边界与业务接入
- 纯中文内容、官方来源优先、经验之谈必须真实可查

现有检索手册已较好覆盖「平台能力」三域（K8s 核心 / AI 基础设施 / LLM 框架），但对照定位，支撑读者成长的四条腿还缺权威来源：

| 定位要求 | 现状 | 补强域 |
|---|---|---|
| R28 二开适配（client-go / CRD / Operator / 平台集成） | PLAN Part 4.6 有章节，手册无任何来源 | 域 B |
| R3 架构师成长（方案设计 / 可靠性 / 案例研读） | Part 6 有章节，手册无方法论来源 | 域 D |
| LLM 应用工程方法论（Agent 模式 / 评估 / 安全边界） | 手册只有框架文档，缺「怎么设计」的来源 | 域 C |
| SPEC 纯中文定位 | 手册几乎全英文来源，缺中文入门阶梯 | 域 H |
| PLAN §5.1 核心叙事线「流量层 / 可观测层」 | 叙事里点了名，手册无检索入口 | 域 E / F |

---

## 二、来源增补（按主题域）

### A. CKAD 备考与考纲（对应 Part 1 / 附录 A）

| 问题类型 | 检索入口 | 说明 |
|---|---|---|
| 考纲原文（唯一权威） | https://github.com/cncf/curriculum | CKAD/CKA/CKS 考纲仓库，Part 1 权重对照表以此为准，标注版本日期 |
| 官方模拟考试 | https://killer.sh | 购买考试即送两套模拟，难度高于真题 |
| 免费练习题库 | https://github.com/dgkanatsios/CKAD-exercises | 社区经典题库，按考纲域组织，适合每章配套练习 |
| 中文文档 | https://kubernetes.io/zh-cn/docs/ | 概念入门用；考试时建议直接用英文版练检索速度 |

### B. K8s 二开适配（对应 Part 4.6，R28 核心落点）

> 使用者视角要点：重点是「怎么用 CRD/Operator 集成平台能力、怎么基于 K8s API 做定制」，不深入 controller 内部实现与分布式原理。

| 问题类型 | 检索入口 | 说明 |
|---|---|---|
| 客户端库选型 | https://kubernetes.io/docs/reference/using-api/client-libraries/ | 官方 client-go 等客户端列表 |
| client-go 用法 | https://github.com/kubernetes/client-go | informer/lister/workqueue 模式看源码 examples |
| Controller 开发 | https://github.com/kubernetes-sigs/controller-runtime | 事实标准的控制器框架，二开绕不开 |
| Operator 脚手架 | https://book.kubebuilder.io/ | Kubebuilder 官方 book，**二开入门最佳主线**——先脚手架跑通，再回头看原理 |
| API 设计规范 | https://github.com/kubernetes/community/tree/master/sig-architecture | SIG Architecture 的 API conventions，设计 CRD 字段前必读 |

**检索路径**：二开问题 → Kubebuilder book（入门跑通）→ controller-runtime（深入）→ API conventions（设计 CRD 时）。不建议从裸写 client-go 开始学。

### C. LLM 应用工程方法论（对应 Part 2 / 5.3）

> 使用者视角要点：框架文档只回答「API 怎么调」，本域来源回答「场景该选什么模式、质量怎么评、风险在哪」——这是「调 API」到「设计 AI 系统」的关键一跃。

| 问题类型 | 检索入口 | 说明 |
|---|---|---|
| Agent 设计模式 | https://www.anthropic.com/engineering/building-effective-agents | Anthropic「Building Effective Agents」：workflow vs agent 的权威区分，Part 2.1「四种模式怎么选」的核心引用 |
| Prompt 工程系统教程 | https://www.promptingguide.ai/ | DAIR.AI 出品，有中文版，术语统一性好 |
| OpenAI 实战配方 | https://cookbook.openai.com/ | RAG / 评估 / 工具调用的官方示例 |
| LLM 系统设计 | https://huyenchip.com/ | Chip Huyen（《AI Engineering》作者），ML 系统设计视角，「经验之谈」优质来源 |
| Agent/LLM 原理综述 | https://lilianweng.github.io/ | Lilian Weng（Lil'Log），Agent / Prompt / RLHF 长文综述，引用价值极高 |
| LLM 评估工具 | https://www.promptfoo.dev/ | promptfoo，开源 eval 框架，补 Langfuse 之外的评估手段 |
| LLM 安全边界 | https://genai.owasp.org/ | OWASP Top 10 for LLM Applications，Part 5.3「安全边界」必备引用 |

### D. 架构与可靠性方法论（对应 Part 6，R3 架构师成长落点）

| 问题类型 | 检索入口 | 说明 |
|---|---|---|
| SLO / 可靠性设计 | https://sre.google/sre-book/table-of-contents/ ⚠️待验证 | Google SRE Book 全文免费，Part 6.2 核心来源 |
| SRE 实操案例 | https://sre.google/workbook/table-of-contents/ ⚠️待验证 | SRE Workbook，SLO 落地与故障演练案例 |
| 架构演进观点 | https://martinfowler.com/ | Martin Fowler，微服务/演进式架构，「经验之谈」高频来源 |
| 大厂架构实录 | https://newsletter.pragmaticengineer.com/ | Gergely Orosz（The Pragmatic Engineer），真实工程决策复盘 |
| 工程方法论 | https://abseil.io/resources/swe-book | 《Software Engineering at Google》免费版 |
| 应用设计基线 | https://12factor.net/ | 12-Factor 原文（Part 4.1 引用源） |
| 案例研读素材 | https://github.com/GoogleCloudPlatform/microservices-demo | Google 官方微服务演示应用，适合作 Part 6.3 首个研读案例 |

**检索路径**：可靠性 / SLO 问题 → SRE Book + Workbook 原文（不依赖中文二手解读）；架构权衡观点 → Martin Fowler / Pragmatic Engineer。

### E. AI 流量与推理编排（PLAN §5.1 核心叙事线「流量层」）

> 使用者视角要点：业务团队关心「模型请求怎么路由 / 灰度 / Fallback」，不关心网关内部实现。

| 问题类型 | 检索入口 | 说明 |
|---|---|---|
| 模型路由标准 | https://gateway-api-inference-extension.sigs.k8s.io/ | Gateway API Inference Extension 官方文档，叙事线流量层的权威来源 |
| AI 原生网关实现 | https://kgateway.dev/ | kgateway（CNCF），Inference Extension 参考实现 |
| 分布式推理编排 | https://github.com/llm-d/llm-d | llm-d（CNCF 沙箱），prefill/decode 分离、KV 缓存感知调度，Part 5.5 前沿方向 |

### F. AI 可观测性（PLAN §5.1 核心叙事线「可观测层」）

| 问题类型 | 检索入口 | 说明 |
|---|---|---|
| GenAI 埋点标准 | https://opentelemetry.io/docs/specs/semconv/gen-ai/ | OTel GenAI 语义约定：token 用量、模型名、调用链的标准字段，Part 3.4 / 4.2 的关键引用 |

**检索路径**：「AI 应用怎么埋点」→ OTel GenAI semconv（字段标准）→ Langfuse（平台落地）。

### G. 云原生供应链安全（对应 Part 4 平台能力边界）

> 使用者视角要点：了解能力边界即可——知道平台提供镜像签名/扫描能力、业务怎么接入 CI。

| 问题类型 | 检索入口 | 说明 |
|---|---|---|
| 镜像签名/验签 | https://www.sigstore.dev/ | Sigstore/Cosign，CNCF 毕业 |
| 漏洞扫描 | https://aquasecurity.github.io/trivy/ | Trivy，镜像/IaC 扫描事实标准 |
| 供应链安全框架 | https://slsa.dev/ | SLSA 框架，了解分级概念即可 |

### H. 中文权威来源（SPEC 纯中文定位的入门阶梯）

> 使用纪律：中文来源用于**概念入门与术语统一**；事实、版本、数据一律回英文官方来源复核，中文社区内容不作「经验之谈」的引用依据。

| 来源 | 链接 | 说明 |
|---|---|---|
| K8s 官方中文文档 | https://kubernetes.io/zh-cn/docs/ | 附录 C 术语表优先采用其译法，保持全书术语一致；版本敏感内容以英文版为准 |
| Jimmy Song（宋净超） | https://jimmysong.io/ | 云原生社区创始人，Istio/云原生中文资料最全 |
| 张磊《深入剖析 Kubernetes》 | https://time.geekbang.org/column/intro/116 | 极客时间专栏，K8s 原理中文讲解标杆（付费，「经验之谈」可引用公开部分） |
| 云原生社区 | https://cloudnative.to/ ⚠️待验证 | 中文云原生社区，Istio Handbook 等译作 |

### I. 生态雷达（补现有 §5 趋势域）

| 来源 | 链接 | 说明 |
|---|---|---|
| CNCF Landscape | https://landscape.cncf.io/ | 生态全景图，选型时先看项目在图里的位置与分类 |
| KubeWeekly | https://www.cncf.io/kubeweekly/ | CNCF 官方周报，跟踪生态动态成本最低的方式 |

---

## 三、检索指南补强

### 3.1 手册 §0「给 agent 的使用说明」增补 3 条

6. **考纲/认证类问题**：只信 https://github.com/cncf/curriculum ，任何博客转述的考纲权重一律视为可能过期
7. **方法论类问题**（Agent 模式 / SLO / 架构权衡）：优先官方工程博客与免费官方书籍（Anthropic Engineering、sre.google、abseil.io），其次才是个人博客
8. **中文来源只用于概念入门**，事实、版本、数据一律回英文官方来源复核

### 3.2 典型问题检索路径速查（补强的落点）

| 典型问题 | 检索路径 |
|---|---|
| CKAD 考纲现在是什么？ | cncf/curriculum → 对应证书目录最新版 |
| 怎么在平台上加一个自定义资源/能力？ | Kubebuilder book → controller-runtime → API conventions |
| 这个 AI 场景该用 workflow 还是 Agent？ | Anthropic Building Effective Agents → Lilian Weng 综述 |
| RAG / Agent 质量怎么评？ | promptfoo + Langfuse → OpenAI Cookbook 评估示例 |
| 怎么给服务定 SLO？ | SRE Book 第 2-4 章 → Workbook 落地案例 |
| 模型请求怎么路由 / 灰度？ | Gateway API Inference Extension → kgateway 实践 |
| AI 调用链怎么埋点？ | OTel GenAI semconv → Langfuse |
| 这个英文概念有没有靠谱中文解释？ | K8s 官方中文文档 → jimmysong.io（再回英文原文复核） |

---

## 四、写作层补强建议（服务定位）

1. **每章「架构师视角」小节固定补三问**：平台提供了什么能力边界？业务接入点在哪（API / CRD / 控制台）？需要和基础设施团队对齐什么？——把「驾驭平台」的定位落到每一章的固定结构里。
2. **Part 4.6 以 Kubebuilder book 为写作主线**：先脚手架跑通一个 CRD + Controller，再回看 client-go 原理，符合「二开适配者」的学习路径，避免陷入 client-go 源码细节。
3. **附录 C 术语表以 K8s 官方中文文档译法为准**，全书统一；首次出现给一句话解释（呼应「通俗语言」原则）。
4. **「经验之谈」优先从本手册已收录来源中选取**（Lilian Weng / Chip Huyen / Martin Fowler / Pragmatic Engineer / 张磊），作者与出处固定可查，降低编造风险。
5. **AI 可观测相关内容统一以 OTel GenAI semconv 为字段标准**，Part 3.4、4.2 与 lab3 保持同一套埋点口径，形成手册内部的呼应。

---

## 更新记录
- 2026-08-28：初版。按「驾驭平台而非建设平台」定位补强：增补 9 个主题域来源（A-I）、3 条检索原则、8 条典型问题检索路径、5 条写作层建议。链接除标注「待验证」外均实测可访问。
