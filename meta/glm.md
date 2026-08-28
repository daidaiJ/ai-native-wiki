# GLM 内容增补与增强（可直接合入 docs/）

> 定位：本文不是评审意见，而是**直接可用的补充内容**——按项目章节模板（概念 → 动手 → 实用技巧 → CKAD 考点对照 → 考察问题 → 经验之谈 → 架构师视角）写成的增补章节，以及可合并进 `knowledge-retrieval-guide.md` 的权威来源检索表。
> 遵守项目纪律：术语首现给一句话解释；来源标注见各章头；按 SPEC 要求，所有命令发布前须在本地 kind 集群验证，本文命令统一标「**验证环境：待验证**」；引用拿不准的一律「观点转述 + 待补引用」，不编造人名出处。
> 标注：**[实用]** = 主线必学，**[进阶]** = 支线按需。

---

## 一、Part 1 增补

### 1.9 应用打包与分发：Helm 与 Kustomize [实用]

> 来源：[Helm 官方文档](https://helm.sh/docs/) · [Kustomize](https://kustomize.io/) · [Artifact Hub](https://artifacthub.io/)
> 验证环境：待验证（kind v0.x + Helm v3.x，发布前须本地跑通）

**概念：为什么需要**

三份环境（dev/staging/prod）× 8 个资源对象 × 镜像 tag / 副本数 / 域名差异 = 裸 YAML 复制粘贴的维护噩梦。打包解决「一份资产，多环境差异」。

- **从哪来**：手工 sed 改 YAML → 社区分流两条路——Kustomize（K8s 原生 overlay 机制，无模板语言）与 Helm（模板 + 包管理器，K8s 生态的 apt/brew）
- **是什么**：Helm 用 Chart（应用包，含模板 + 默认值）和 values.yaml（环境差异化配置）实现「参数化 + 仓库分发」；Kustomize 用 base（公共清单）+ overlay（环境叠加层）实现「声明式打补丁」，产物仍是纯 YAML
- **往哪去**：Chart 走 OCI 化（chart 存进镜像仓库，与镜像同库同权限）；渲染从客户端移到 CD 端（ArgoCD/Flux 在 GitOps 侧统一渲染，本地不再有「渲染产物漂移」问题）

**动手**（命令待验证）

```bash
# 1. 生成一个 Chart 骨架并看结构
helm create demo && tree demo

# 2. values 差异化安装（upgrade --install 幂等发布）
helm upgrade --install demo ./demo -n demo --create-namespace --set image.tag=v1.0.0

# 3. 渲染预检：先看到最终 YAML 再进集群
helm template demo ./demo | kubectl apply --dry-run=server -f -

# 4. 出问题回滚
helm rollback demo 1

# Kustomize 路线：overlays/prod/kustomization.yaml 里改 replicas/镜像 tag，然后
kubectl apply -k overlays/prod
```

**实用技巧**

- `helm diff` 插件（helm plugin install ...）：升级前看逐行差异，防止盲发
- `helm get values demo --revision 2`：查某次发布实际生效的配置
- 上 Artifact Hub 找现成 chart（Bitnami 系质量较稳），`helm pull bitnami/nginx --untar` 拆开学习模板写法
- `kubectl diff -k overlays/prod`：先 diff 再 apply，GitOps 前的最后一道保险

**CKAD 考点对照**：CKAD 不考 Helm/Kustomize；但 `kubectl apply -k` 考试可用，且「模板生成 → dry-run → apply」思路与考试 `--dry-run=client -o yaml` 一脉相承。

**考察问题**：ArgoCD 为什么能同时支持 Helm 与 Kustomize？为什么全 GitOps 流水线的团队更倾向 Kustomize？（线索：CD 端渲染 vs 客户端渲染；声明式最终状态；chart 依赖解析发生在哪一侧）

**经验之谈**：对自有应用 + GitOps，「Kustomize 更贴近声明式」是社区通行判断；Helm 的 Go template 调试成本高（模板错误难定位）也是长期争议点。对分发/安装第三方中间件，Helm 生态无可替代。具体人名引用待补。

**架构师视角**

- 解决什么问题：一份资产多环境差异；第三方应用的标准化分发与升级
- 何时用 Helm：装中间件（PG/Redis/Kafka 类）、差异维度多、团队已有 chart 资产
- 何时不用：自家应用 + 纯 GitOps → Kustomize 更轻；甚至可以直接用 Flux 的 Kustomization API 在 CD 端渲染
- 权衡：Helm 表达力强但复杂度换来的；Kustomize 简单但环境差异超过约三成时 overlay 会失控——差异规模是分界线

### 1.3 增强：Pod Security Admission 小节

> 来源：[K8s 官方：Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) · 验证环境：待验证

PSP（PodSecurityPolicy）已于 1.25 移除，继任者 PSA（Pod Security Admission）是 kube 内建准入控制。对业务开发者只需要三件事：

1. 三档安全级别：`privileged`（不限制）/ `baseline`（防明显提权）/ `restricted`（最严，要求非 root、只读根文件系统、去掉多余 capability）
2. 它是 namespace 上的标签，不是新对象：
   ```bash
   kubectl label ns prod pod-security.kubernetes.io=enforce=restricted --overwrite
   ```
3. 排障识别：Pod 被拒的报错形如「violates PodSecurity "restricted:latest"」——去找平台方要该 ns 的 profile，对照改 SecurityContext

CKAD 不考 PSA 本身，但 `restricted` 的要求（`runAsNonRoot`、`seccompProfile` 等）与 Configuration 域的 SecurityContext 考点直接呼应。

架构师视角：业务侧的义务是「知道平台设了哪档、我的 Pod 为什么被拒、怎么改到合规」；设档是平台方的职责——这正是本项目「使用者定位」的典型场景。

---

## 二、Part 2 增补

### 2.3 前置小节：RAG 的数据底座——Embedding 与向量库选型 [实用]

> 来源：[MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard) · [pgvector](https://github.com/pgvector/pgvector) · [Qdrant](https://qdrant.tech/documentation/) · [Milvus](https://milvus.io/docs)

**概念：为什么需要**

RAG 的效果上限不在「框架」，而在检索；检索的上限在两件事：

- **Embedding**（向量化）：把文字压成向量，语义相近 = 向量距离近，一句话解释完。检索质量的「天花板」由它决定
- **向量库**：存向量并做近邻搜索的存储。决定天花板能撑到的规模与过滤能力

- **从哪来**：BM25 关键词检索 → word2vec 静态词向量 → BERT 时代句向量 → 对比学习检索模型（bge / gte / jina 等开源系）→ 多语言统一
- **是什么**：选型题 = 一个 embedding 模型 + 一个向量库
- **往哪去**：embedding 与 reranker（重排序器）组合成为标配两段式：先向量召回，后精排

**选型决策表（中台开发者视角）**

| 路线 | 何时选 | 一句话理由 |
|---|---|---|
| pgvector（PG 插件） | 千万级以下向量、公司已有 PostgreSQL | 零新增组件，运维成本最低，中台团队首选 |
| Qdrant | 需要独立部署、云原生向量库 | Rust 实现、过滤检索能力强、部署简单 |
| Milvus | 十亿级向量、有专职运维 | 分布式架构、生态大、但运维重 |

Embedding 选型：MTEB 榜按任务类型查分（中文场景看 C-Retrieval / Retrieval 任务），注意维度——维度越高存储与内存成本越高，不是越大越好。

**动手**（pgvector 15 分钟最短路径，命令待验证）

```sql
CREATE EXTENSION vector;
CREATE TABLE chunks (id bigserial PRIMARY KEY, content text, embedding vector(1024));
-- <=> 是向量库里的「最近距离」比较算子（余弦距离）
SELECT content FROM chunks ORDER BY embedding <=> $1 LIMIT 5;
```

**考察问题**：为什么很多团队最后没用专用向量库？什么时候必须换？（线索：数据规模、QPS、组合过滤复杂度、运维预算）

**经验之谈**：多数企业 RAG 卡在文档解析与切块（chunking）而非向量库选型；切块策略的影响常大于换 embedding 模型——这是社区通行经验，具体引用待补。

**架构师视角**：先 pgvector 起步，量级上来再迁移；迁移成本主要在重建索引，选型时用 LangChain / LlamaIndex 的 VectorStore 接口做防腐层，业务代码不直连向量库 SDK。

### 2.5 增强：RAG 质量评估——RAGAS 四指标 + CI 回归 [实用]

> 来源：[RAGAS 文档](https://docs.ragas.com/) · [promptfoo](https://www.promptfoo.dev/docs/intro/)（路径待验证）

原 2.5 覆盖了平台层（Langfuse/LangSmith），缺「质量层」——没有它，改 prompt / 换模型全凭感觉。

**指标四件套（RAGAS 事实标准）**

| 指标 | 度量什么 | 一句话人话 |
|---|---|---|
| faithfulness（忠实度） | 答案是否只来自检索内容 | 模型有没有瞎编 |
| answer relevancy | 答案是否切题 | 答非所问吗 |
| context precision | 检索内容里有效信息的排位 | 找回来的东西排得靠前吗 |
| context recall | 应检回的内容覆盖度 | 该找的都找回来了吗 |

**动手**（待验证）：建 20~50 条「问题 + 标准答案 + 标准检索片段」的金标集 → `ragas` 跑分 → promptfoo 把跑分红绿灯接进 CI：改 prompt 必须跑评估，低于阈值阻断合并。

**经验之谈**：Hamel Husain 的核心观点——AI 产品最先该建的不是功能是评估（其 evals 系列文章，[hamel.dev](https://hamel.dev/)）；Eugene Yan 持续产出 LLM 评估实践（[eugeneyan.com](https://eugeneyan.com/)）。

**架构师视角**：没有评估集的 RAG 项目 = 没有测试的服务，不允许上线。评估集本身是资产，随 badcase 持续回填。

### 2.7 LLM 应用安全 [实用]

> 来源：[OWASP Top 10 for LLM Applications](https://genai.owasp.org/) · [Simon Willison: Prompt injection 系列](https://simonwillison.net/tags/prompt-injection/)

**概念：为什么需要**

Agent 会读网页、调工具、带权限——风险从「模型说错话」升级为「模型被人指挥做坏事」。这一章给的是给业务开发者的安全底线，不是安全专家课程。

**威胁速览**（对齐 OWASP LLM Top 10，条目顺序以官网为准）：提示注入、敏感信息泄露、供应链（第三方模型 / 插件 / MCP server 本身作恶）、输出处理不当（模型输出二跳成 XSS/命令注入）、过度代理（excessive agency：工具权限给太大）、系统提示泄露、向量与嵌入弱点、无限制消耗（被刷 token）。

**最关键的一组概念：直接 vs 间接提示注入**（Simon Willison 提出的划分）

- 直接注入：用户在输入框里直接骗模型——危害有限（用户骗自己）
- 间接注入：攻击者把指令藏进网页 / 文档 / 邮件，等 RAG 检索或 Agent 浏览时喂给模型——**这才是 RAG/Agent 的新攻击面**：你的检索源是不可信输入

**防御分层（务实版）**

1. 别指望 prompt 防注入——「请忽略以上指令」挡不住变体；prompt 是提示，不是防线
2. 权限最小化：工具只读优先、白名单化、每个工具独立配额
3. 隔离执行：Agent 的工具在独立容器 / 独立 ServiceAccount 里跑（K8s 擅长的事，呼应 Part 3）
4. 高危动作人在环节：删除、外发、付款必须人工确认
5. 输出侧防御：结构化输出（JSON Schema 约束）+ 输出按上下文编码

**动手**（待验证）：给一个工具调用 Agent 的工具集加约束——去掉写操作、加每分钟调用上限、输出走 JSON Schema 校验失败即拒绝。

**考察问题**：既然提示注入没有可靠根治解，系统凭什么还能安全上线？（线索：把模型当不可信组件，安全边界放在工具权限与数据隔离层——类比浏览器与操作系统）

**经验之谈**：Simon Willison 长期跟踪提示注入，核心判断（观点转述）——间接提示注入是 LLM 应用最棘手的安全问题，且无法靠提示词根除；原文见其博客 tag 页。

**架构师视角**：数据分级决定部署模式（公开数据走公网 API，涉密数据私有化）；Agent 永远在非信任域；上线前安全评审对 OWASP 清单逐条走查——这是给评审者的抓手。

---

## 三、Part 3 增补

### 3.5 LLM 网关与成本治理 [实用]

> 来源：[LiteLLM](https://docs.litellm.ai/) · [Envoy AI Gateway](https://github.com/envoyproxy/ai-gateway) · [OTel GenAI 语义约定](https://opentelemetry.io/docs/specs/semconv/gen-ai/)

**概念：为什么需要**

团队从「各自调 OpenAI key」长大后会撞三堵墙：key 无法统一管控、token 成本看不见、换模型要改业务代码。网关层一次解决三件事。

- **从哪来**：业务直连厂商 SDK → key 共享打 cookie → 出现统一代理 → 演化为 K8s 原生网关
- **是什么**：
  - **LiteLLM**：统一 OpenAI 兼容端点的代理——多 provider 路由、虚拟 key 池、按 key 预算限流，事实标准
  - **Envoy AI Gateway**：K8s 原生 GenAI 网关（CNCF 沙箱）——provider 路由、故障 fallback、token 级限流，与 Gateway API 体系接轨
- **往哪去**：网关成为 LLM 时代的「接入层」，配合推理侧的 Inference Extension 形成完整流量链路

**动手**（LiteLLM 最小配置，待验证）

```yaml
# config.yaml：一个虚拟 key + 月度预算 + provider fallback
model_list:
  - model_name: gpt-4o
    litellm_params:
      model: openai/gpt-4o
      api_key: os.environ/OPENAI_API_KEY
  - model_name: gpt-4o        # 同名双后端 = 免费获得 fallback
    litellm_params:
      model: anthropic/claude-sonnet
      api_key: os.environ/ANTHROPIC_API_KEY
litellm_settings:
  budget_duration: 30d
general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
```

业务代码从此只认 `http://litellm-gateway.internal/v1`，OpenAI SDK 直接指向它。

**成本观测闭环**：网关转发时按 OTel GenAI 语义约定打点（gen_ai.usage.input_tokens / output_tokens 等标准字段）→ 进 Prometheus → Grafana 按团队 / feature 两个维度做 showback（成本分摊展示，先看见再治理）。

**考察问题**：为什么业务代码不应该 import 厂商 SDK 的专有类型？（线索：防腐层；换模型是运营决策不是代码变更）

**经验之谈**：观点——「先统一网关，再谈优化」：厂商切换、缓存、降级、审计都发生在网关层，晚建不如早建（社区通行经验，具体引用待补）。

**架构师视角**：什么时候不需要网关——单人原型 / 单一模型无所谓；团队化之后网关是必选项。权衡：多一跳延迟（通常 < 10ms 量级）换管控力。

### 3.4 增强：GPU 指标接入与成本分摊

> 来源：[DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter) · [FinOps Foundation](https://www.finops.org/) · 验证环境：待验证

两步把 GPU 装进现有 Prometheus 体系：

1. DaemonSet 部署 dcgm-exporter（NVIDIA 官方，GPU 指标暴露标准方式），核心指标：`DCGM_FI_DEV_GPU_UTIL`（利用率）、显存使用/温度（字段名待验证）
2. ServiceMonitor 接入 → Grafana 看板按 namespace / Pod 维度看利用率

成本分摊（showback 思路）：GPU 利用率 × 卡单价 → 按 namespace 汇总 → 每月出「AI 账单」。这是 FinOps 的第一课：先让成本可见，再谈降本。进阶路线见 FinOps Foundation 的框架（[进阶]）。

---

## 四、Part 4 增补

### 4.6 增强：K8s 二开适配速查（client-go / CRD / Operator 使用者路线）

> 来源：[client-go](https://pkg.go.dev/k8s.io/client-go) · [sample-controller](https://github.com/kubernetes/sample-controller) · [Kubebuilder 书](https://book.kubebuilder.io/) · [API 约定](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)
> 参考书：《Programming Kubernetes》（O'Reilly，Hausenblas & Schimanski）[进阶]

原 4.6 大纲有了，补一段「使用者路线图」让读者不迷路：

**client-go 三套姿势（按需升级）**

| 姿势 | 场景 | 一句话 |
|---|---|---|
| 一次性 List/Get | 后台任务、低频同步 | 最简单，别用来做高频轮询 |
| Watch + Informer | 需要感知资源变化 | 本地缓存 + 增量事件，二开标配 |
| Dynamic client（unstructured） | 处理任意 CRD、写通用工具 | 无类型，代价是自己解析字段 |

**实操要点**

1. 官方 sample-controller 是 informer + workqueue 模式的最短可跑路径，先跑通再改
2. 使用者视角的 API 发现三板斧：`kubectl api-resources` / `kubectl get crd`（看 served versions）/ `kubectl explain <resource>.spec`
3. 工程坑：reconcile 里禁止长阻塞（外部 API 调用放 workqueue 异步做）；controller 单独建 ServiceAccount 配最小 RBAC；Lister 读的是本地缓存（最终一致，不是强一致）
4. Operator 判断力：很多「想写 Operator」的场景一个 CronJob 就够——Operator 的价值在「状态驱动的自愈」，一次性任务不是

**考察问题**：为什么 controller 的 reconcile 函数必须是幂等且快速返回？（线索：workqueue 重入、租约时间、级联重试）

**经验之谈**：观点——绝大多数「二开」的正确形态是「监听 CRD 变化 → 调内部平台 API 同步」，而不是修改平台组件本身；改内核 / 换组件是平台团队的地盘（通行工程共识，待补具体引用）。

**架构师视角**：优先级——CRD + Controller 表达需求 > Webhook 拦截 > 改平台。Webhook 是 API 链路上的强依赖：它挂了，全集群的对应操作都会阻塞，接入前先问平台方「高可用谁保障」。

---

## 五、Part 6 增补

### 6.1 增强：ADR 架构决策记录模板（可直接复制进方案文档）

> 来源：[adr.github.io](https://adr.github.io/)

ADR（Architecture Decision Record，架构决策记录）：一页纸记录一个决策的「为什么」，对抗半年后「当初为什么这么选」的失忆。模板：

```markdown
# ADR-001：向量库采用 pgvector

- 状态：已接受（2026-08-28）
- 背景：RAG 需要向量检索；当前数据量 < 500 万条；团队已运维 PostgreSQL 集群
- 决策：首期用 pgvector，不引入专用向量库
- 备选方案：Qdrant（部署简单但新增组件）/ Milvus（分布式能力暂用不上）
- 后果：延迟换运维 simplicity；若向量超过千万级需迁移（迁移点：重建索引，见防腐层设计）
- 复核触发条件：向量规模 > 1000 万 或 QPS > 500
```

方案文档模板（6.1）建议直接内嵌 3~5 篇 ADR——比长篇论证更可维护。

### 6.2 增强：SLO 起步模板（SLI → SLO → 错误预算）

> 来源：[Google SRE Book（免费全文）](https://sre.google/books/) · [DORA](https://dora.dev/)

- **SLI**（服务水平指标）：`好事件数 / 总事件数`。例：成功响应数 / 总请求数
- **SLO**（服务水平目标）：给 SLI 定目标。例：99.9%
- **错误预算**：1 − SLO。99.9% = 每月允许 43.2 分钟不可用（30 天 × 24h × 0.1%）
- **预算纪律**：预算烧完 → 冻结新功能发布，只修稳定性——这一条是 SLO 体系真正起作用的机制

起步三步：先挑 1 个用户可感知的 SLI（不要一上来 20 个指标）→ 定一个「敢公布」的 SLO → 每月复盘错误预算消耗。经典体系化来源是 Google SRE Book/Workbook；交付效能侧的权威是 DORA 年度报告（[进阶]）。

### 6.3 增强：案例研读固定信源

- [ACM Queue](https://queue.acm.org/)：资深工程师一手架构叙事，深且慢
- [Latent Space](https://www.latent.space/)：AI 工程实践访谈，工程视角而非学术
- InfoQ 架构栏目（中文）：[infoq.cn](https://www.infoq.cn/)
建议每季度从三个源各挑 1 篇做拆解（问题 → 约束 → 决策 → 教训），正好匹配「每季度更新」节奏。

---

## 六、附录增补

### 附录 A 增强：CKAD 备考三件套

| 项 | 入口 | 用法 |
|---|---|---|
| 官方考试页 | https://www.cncf.io/training/certification/ckad/ | 大纲与形式的唯一权威源；大纲会更新，引用时标注读取日期 |
| 官方模拟环境 | https://killer.sh/ | 考票附赠的官方模拟（KillerCoda 同源）；考前一周限时完整做一遍，练的是「查文档速度」不是「背命令」 |
| 开源题库 | https://github.com/dgkanatsios/CKAD-exercises | 按考点分类免费刷题 |

备考纪律一条：官方 curriculum 页每次考试版本可能调整，模拟题资源（尤其第三方课程）可能出现考点漂移，一切以官方页当日内容为准。

### 附录 B 与检索手册的关系（一页纸入口）

附录 B 不重复堆链接（AGENTS.md 已规定链接统一在 `knowledge-retrieval-guide.md` 维护），只放一段使用说明 + 指向手册的链接 + 「如何给手册贡献新来源」的三步（先验证可访问 → 按主题域归位 → 更新记录写日期）。

---

## 七、权威知识来源与检索指南增补

> 以下各表格式与 `knowledge-retrieval-guide.md` 一致，可直接合并。⏳ = 链接/路径需人工验证后采用。

### 7.1 增补进「1. Kubernetes 核心与演进」

| 问题类型 | 检索入口 | 说明 |
|---|---|---|
| 中文官方文档 | https://kubernetes.io/zh-cn/docs/ | 官方中文翻译，与 CKAD 可查的英文版同源，术语对读利器 |
| Pod 安全标准 | https://kubernetes.io/docs/concepts/security/pod-security-standards/ | PSA/PSS，PSP 继任机制 |
| Ingress Controller 事实标准 | https://kubernetes.github.io/ingress-nginx/ | ingress-nginx，动手章节默认选择 |
| Gateway API | https://gateway-api.sigs.k8s.io/ | SIG 官方项目；PLAN §5 演进主线，手册原有空白 |
| 集群原理（进阶） | https://github.com/kelseyhightower/kubernetes-the-hightower-way ⏳ | Kelsey Hightower 的 Kubernetes The Hard Way；正确地址 github.com/kelseyhightower/kubernetes-the-hard-way，平台使用者按需读 |
| 镜像扫描 | https://github.com/aquasecurity/trivy | Trivy，容器/依赖/IaC 扫描事实标准 |
| 镜像签名 | https://docs.sigstore.dev/ | Sigstore/cosign |
| 供应链等级 | https://slsa.dev/ | SLSA 框架（理解级别即可）[进阶] |
| 制品发现 | https://artifacthub.io/ | Helm chart/OPA 等 CNCF 制品中心 |

### 7.2 新增主题域「应用打包与分发」

| 问题类型 | 检索入口 | 说明 |
|---|---|---|
| Helm 官方 | https://helm.sh/docs/ | Chart 模板/values/仓库，分发事实标准 |
| Kustomize | https://kustomize.io/ | K8s 原生 overlay，GitOps 配套 |

### 7.3 新增主题域「K8s 二开与 API 编程」（服务 Part 4.6 / R28）

| 问题类型 | 检索入口 | 说明 |
|---|---|---|
| client-go API | https://pkg.go.dev/k8s.io/client-go | 官方 Go 客户端库 |
| controller 样例 | https://github.com/kubernetes/sample-controller | informer+workqueue 最短路径 |
| API 约定 | https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md | K8s API 设计约定（为什么字段这样设计） |
| CRD/Operator 开发 | https://book.kubebuilder.io/ | Kubebuilder 官方书 |
| Operator SDK | https://sdk.operatorframework.io/ | Operator 脚手架（Red Hat 维护） |
| 参考书 | 《Programming Kubernetes》O'Reilly | client-go/API machinery 成体系参考（英文）[进阶] |

### 7.4 增补进「4. LLM 应用与 Agent」

| 问题类型 | 检索入口 | 说明 |
|---|---|---|
| LLM 应用安全 | https://genai.owasp.org/ | OWASP Top 10 for LLM Applications，安全评审事实基线 |
| Agent 构建（Anthropic 一手） | https://www.anthropic.com/research/building-effective-agents | 「工作流 vs Agent」经典划分 |
| Agent 构建（OpenAI 一手） | https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf ⏳ | 官方 Agent 实践指南 |
| Prompt 系统教程 | https://www.promptingguide.ai/ | DAIR.AI 开源指南（含中文） |
| 厂商 prompt 指南 | https://platform.openai.com/docs/guides/prompt-engineering · https://docs.anthropic.com/ | 两家官方工程指南 |
| RAG 评估 | https://docs.ragas.com/ | RAGAS，RAG 质量评估事实标准 |
| LLM 回归测试 | https://www.promptfoo.dev/docs/ ⏳ | promptfoo，CI 级回归 |
| Embedding 选型 | https://huggingface.co/spaces/mteb/leaderboard | MTEB 权威基准 |
| 向量库 | https://github.com/pgvector/pgvector · https://qdrant.tech/documentation/ · https://milvus.io/docs | pgvector（中台优先）/ Qdrant / Milvus |
| 模型枢纽 | https://huggingface.co/docs | HF 文档（transformers/datasets/hub） |
| 官方示例库 | https://github.com/openai/openai-cookbook | 结构化输出/评估/RAG 官方模式集 |
| LLM 网关 | https://docs.litellm.ai/ ⏳ | LiteLLM，统一代理+key 治理+预算 |
| 提示注入一手跟踪 | https://simonwillison.net/tags/prompt-injection/ | Simon Willison 专题页 |

### 7.5 增补进「3. AI 基础设施」与「2. 云原生生态」

| 问题类型 | 检索入口 | 说明 |
|---|---|---|
| GPU 指标 | https://github.com/NVIDIA/dcgm-exporter | DCGM Exporter，GPU→Prometheus 标准方式 |
| 推理优化（NVIDIA） | https://nvidia.github.io/TensorRT-LLM/ | TensorRT-LLM [进阶] |
| disaggregated inference | https://github.com/NVIDIA/dynamo ⏳ | PLAN §5.2 点名的 Dynamo 上游 |
| CNCF 新秀 | https://github.com/llm-d/llm-d · https://github.com/kaito-project/kaito | 手册 §5 雷达已点名但缺链接 |
| OTel GenAI 语义约定 | https://opentelemetry.io/docs/specs/semconv/gen-ai/ | PLAN §5.1 可观测层支柱的规范原文 |
| LLM 推理追踪 | https://github.com/arize-ai/openinference | OpenInference（Langfuse/Arize 采用）⏳ 定位建议核验 |
| FinOps | https://finops.org/ | FinOps Foundation，GPU 成本治理 [进阶] |

### 7.6 增补进「5. 趋势与权威报告」+ 新增「架构方法论」

| 问题类型 | 检索入口 | 说明 |
|---|---|---|
| SLO/可靠性源头 | https://sre.google/books/ | Google SRE Book + Workbook，免费全文 |
| 交付效能 | https://dora.dev/ | DORA 年度报告 |
| 架构描述 | https://c4model.com/ | C4 模型（Context/Container/Component/Code 四层） |
| 决策记录 | https://adr.github.io/ | ADR 轻量实践 |
| 系统设计通识 | https://github.com/donnemartin/system-design-primer | 开源教材（英文）[进阶] |
| 架构访谈 | https://queue.acm.org/ | ACM Queue 一手叙事 |
| AI 工程访谈 | https://www.latent.space/ | Latent Space |

### 7.7 新增「经验之谈人物/信源补充」（服务章节模板第 6 节 / R20）

| 人物/信源 | 领域 | 适用章节 |
|---|---|---|
| Chip Huyen（[huyenchip.com](https://huyenchip.com/)，《AI Engineering》O'Reilly 2025） | ML/LLM 系统与工程化 | Part 2.1 / Part 5.2 |
| Eugene Yan（[eugeneyan.com](https://eugeneyan.com/)） | LLM 应用与评估实践 | Part 2.5 |
| Hamel Husain（[hamel.dev](https://hamel.dev/)） | evals 实战（「AI 产品需要评估」） | Part 2.5 |
| Latent Space（swyx & Alessio） | AI 工程访谈 | Part 6.3 |
| Simon Willison（已有，补 tag 页） | 提示注入一手研究 | Part 2.7 |
| Kelsey Hightower（KTHW） | K8s 原理与社区共识 | Part 1 / 4 |

引用纪律提醒：以上仅收录真实存在的人物与站点；具体观点写入「经验之谈」时须回原文核对并给链接，拿不准只写观点不署名（AGENTS.md 规范 7）。

### 7.8 新增「经典论文锚点」

| 论文 | arXiv | 对应章节 |
|---|---|---|
| Attention Is All You Need | https://arxiv.org/abs/1706.03762 | Part 2.1（Transformer 起点文档） |
| ReAct | https://arxiv.org/abs/2210.03629 | Part 2.4（手册已有） |
| PagedAttention（vLLM） | https://arxiv.org/abs/2309.06180 | Part 3.2（推理引擎核心论文） |
| RAG Survey | https://arxiv.org/abs/2312.10997 | Part 2.3（手册已有） |

作用：训练「读一手来源」习惯，框架文档只告诉你怎么用，论文告诉你为什么这样设计。

### 7.9 新增「中文入口」

| 入口 | 说明 |
|---|---|
| https://kubernetes.io/zh-cn/docs/ | 最重要的中文权威源（官方翻译） |
| https://www.infoq.cn/ | 架构/AI 落地中文一手报道 |
| KubeCon China（events.linuxfoundation.org）⏳ | 国内 AI on K8s 实践最密集来源，建议按年收录议程 |

### 7.10 建议的更新记录写法（合入后追加）

```markdown
- 2026-08-28（GLM 增补）：新增主题域「应用打包」「K8s 二开与 API 编程」「架构方法论」「中文入口」；
  LLM 域补 OWASP/RAGAS/MTEB/向量库/LiteLLM；AI 基础设施域补 DCGM/OTel GenAI/FinOps；
  新增经验之谈人物表与经典论文锚点。⏳ 标注项待人工验证后去除标记。
```

---

## 待人工验证清单（⏳ 汇总）

- kelseyhightower KTHW 正确仓库名（本文 7.1 表中已给出正确形式，删错误项）
- OpenAI building agents 指南 PDF 路径
- promptfoo / litellm 文档具体路径
- NVIDIA Dynamo 仓库主地址
- OpenInference 的定位表述
- KubeCon China 年度议程链接
- CKAD 当前大纲版本与附赠模拟环境使用次数（以官方页为准）
- 本文所有「验证环境：待验证」命令的本地 kind 实测

---

*合入方式：第一~六节 → `docs/part*/` 对应章节目录；第七节 → `docs/appendix/knowledge-retrieval-guide.md` 对应分节。合入时遵守 AGENTS.md：修改 PLAN.md 增补新章节编号（1.9 / 2.7 / 3.5）需同步更新 PLAN §3 大纲表。*
