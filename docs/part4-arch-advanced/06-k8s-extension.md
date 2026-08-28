# 4.6 增强：K8s 二开适配速查（client-go / CRD / Operator 使用者路线）[进阶]

> 来源：[client-go](https://pkg.go.dev/k8s.io/client-go) · [sample-controller](https://github.com/kubernetes/sample-controller) · [Kubebuilder 书](https://book.kubebuilder.io/) · [API 约定](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md) · 验证环境：待验证
> 参考书：《Programming Kubernetes》（O'Reilly，Hausenblas & Schimanski）[进阶]

## 概念：为什么需要

业务痛点驱动：

- **[已必备] 平台能力重复建设**：每个团队对接配置中心 / 网关 / 可观测各写各的适配器，同一件事 N 套实现，没人长期维护
- **[已必备] 系统耦合改不动**：业务代码里到处是平台 API 调用细节，平台一升级全链路跟着改
- **[潜在] 平台能力要「产品化」**：把重复的适配逻辑沉淀成可申请、可复用的服务，需求以 K8s 原生形态表达

场景实例：中台要做一个「配置变更自动同步」能力——没有 CRD 时，各团队自己写脚本轮询配置中心，轮询频率、失败重试、权限各搞一套；有 CRD 时，业务侧声明一个资源对象（我要什么），平台侧 controller 负责对账（怎么实现），业务侧不再关心实现细节。

- **从哪来**：手工脚本轮询 API → 官方 client-go 封装 → Informer 本地缓存 + 增量事件 → CRD + Controller 声明式对账 → Operator 模式（把运维知识编码进控制器）
- **是什么**：K8s 二开 = 基于 K8s API 做定制开发。使用者路线 = 不建设平台，但能读懂 CRD、能写适配器、能判断要不要 Operator
- **往哪去**：CRD 成为「平台能力的产品化接口」——业务侧声明需求，平台侧对账实现；AI 时代 Kagent 等「Agent 原生」项目同样以 CRD 声明式表达（呼应 Part 5.4）

**写作主线**（SPEC §7）：以 Kubebuilder 书为主线——先脚手架跑通 CRD + Controller，再回看 client-go 原理，避免陷入源码细节。

**引导反思**：二开的正确姿势是「在平台之上加一层适配」，不是「改平台本身」——改内核 / 换组件是平台团队的地盘。

## 动手（验证环境：待验证）

第一步：Kubebuilder 脚手架跑通 CRD + Controller（最短路径，命令待验证）：

```bash
# 1. 初始化项目（domain 是资源组名的域名后缀）
kubebuilder init --domain example.com --repo example.com/mycrd

# 2. 生成一个 CRD + Controller 骨架
kubebuilder create api --group apps --version v1 --kind MyResource --resource --controller

# 3. 装进本地 kind 集群并跑起来
make install        # 把 CRD 安装进集群
make run            # 本地运行 controller，连 kind 集群
```

CRD 由脚手架生成（`make install` 安装），不需要手写——这正是「先跑通再回看原理」的意义。

第二步：回看 client-go 三套姿势（按需升级）：

| 姿势 | 场景 | 一句话 |
|---|---|---|
| 一次性 List/Get | 后台任务、低频同步 | 最简单，别用来做高频轮询 |
| Watch + Informer | 需要感知资源变化 | 本地缓存 + 增量事件，二开标配 |
| Dynamic client（unstructured） | 处理任意 CRD、写通用工具 | 无类型，代价是自己解析字段 |

第三步：使用者视角的 API 发现三板斧：

```bash
kubectl api-resources            # 看集群有哪些资源类型
kubectl get crd                  # 看自定义资源（served versions）
kubectl explain <resource>.spec  # 查字段定义
```

## 实用技巧

- 官方 sample-controller 是 informer + workqueue 模式的最短可跑路径，先跑通再改
- 工程坑一：reconcile 里禁止长阻塞——外部 API 调用放 workqueue 异步做
- 工程坑二：controller 单独建 ServiceAccount 配最小 RBAC，别用集群管理员身份跑
- 工程坑三：Lister 读的是本地缓存（最终一致，不是强一致），别当实时查询用
- Operator 判断力：很多「想写 Operator」的场景一个 CronJob 就够——Operator 的价值在「状态驱动的自愈」，一次性任务不是
- 读 CRD 先看 spec 再看 status：spec 是用户声明（我要什么），status 是平台回写（对账结果）

## CKAD 考点对照

无（Part 4 不在 CKAD 范围内）。

## 考察问题

- 为什么 controller 的 reconcile 函数必须是幂等且快速返回？（线索：workqueue 重入、租约时间、级联重试）
- 为什么 Lister 读本地缓存而不是直接查 API server？（线索：最终一致 vs 强一致；API server 压力与限流）
- 什么场景该用 CronJob 而不是 Operator？（线索：状态驱动自愈 vs 定时触发；Operator 的运维成本）

## 经验之谈

- 观点（不署名）：绝大多数「二开」的正确形态是「监听 CRD 变化 → 调内部平台 API 同步」，而不是修改平台组件本身；改内核 / 换组件是平台团队的地盘（通行工程共识，待补具体引用）

## 架构师视角

- 解决什么问题：把「平台能力」产品化成 K8s 原生接口——业务侧声明、平台侧对账，消灭各团队重复造轮子
- 何时用：需要把重复的适配逻辑沉淀成平台能力；需要状态驱动的自愈（如配置同步、资源回收）
- 何时不用：一次性任务用 CronJob；低频同步用一次性 List/Get；别为写 Operator 而写 Operator
- 权衡：优先级——CRD + Controller 表达需求 > Webhook 拦截 > 改平台。Webhook 是 API 链路上的强依赖：它挂了，全集群的对应操作都会阻塞，接入前先问平台方「高可用谁保障」
- 固定三问：
  - 平台提供了什么能力边界？——平台是否已提供 CRD / 扩展 API / 现成 Operator，还是需要业务侧自建适配器；Webhook 高可用谁保障
  - 业务接入点在哪？——CRD 资源对象（声明式）/ client-go 或 Python 客户端（编程式）/ kubectl（人工操作）
  - 需要和基础设施团队对齐什么？——CRD 字段约定与版本演进、RBAC 权限、Webhook 高可用、对账频率与 API 限流

## 核心收获表

| 能力维度 | 具体收获 |
|---|---|
| client-go 三套姿势 | 能按场景选型：一次性 List/Get / Watch + Informer / Dynamic client |
| CRD 理解与使用 | 能读懂 CRD 的 spec / status，会用 kubectl explain 查字段 |
| Operator 判断力 | 能判断「该写 Operator 还是 CronJob」 |
| 工程坑规避 | reconcile 不阻塞、controller 最小 RBAC、Lister 最终一致 |
| 二开优先级 | CRD + Controller > Webhook > 改平台，Webhook 接入先问高可用 |

## 推荐开源项目表

| 项目 | 链接 | 研读重点 |
|---|---|---|
| sample-controller | https://github.com/kubernetes/sample-controller | informer + workqueue 最短可跑路径 |
| Kubebuilder 书 | https://book.kubebuilder.io/ | 二开入门最佳主线：先脚手架跑通，再回看原理 |
| client-go | https://pkg.go.dev/k8s.io/client-go | 官方 Go 客户端库 |
| controller-runtime | https://github.com/kubernetes-sigs/controller-runtime | 事实标准的控制器框架 [进阶] |
| Operator SDK | https://sdk.operatorframework.io/ | Operator 脚手架（Red Hat 维护）[进阶] |
| 《Programming Kubernetes》 | O'Reilly | client-go / API machinery 成体系参考（英文）[进阶] |

## 常见问题排查

| 高频报错 / 现象 | 排查路径 |
|---|---|
| reconcile 不执行 / 事件堆积 | 检查 workqueue 是否被长阻塞任务卡住；看 controller 日志的重试与限流 |
| Lister 读到旧数据 | 最终一致是设计行为——等缓存同步 / resync；确认 watch 连接是否断开 |
| 403 Forbidden | controller 的 ServiceAccount 权限不足——最小 RBAC 配错，找平台方核对角色 |
| CRD 版本对不上 | 检查 served / storage version；`kubectl get crd` 看实际生效版本 |
| Webhook 挂了，全集群操作阻塞 | 这是强依赖的代价——先找平台方确认高可用与恢复路径 |