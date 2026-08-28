# 3.4 增强：GPU 指标接入与成本分摊

> 来源：[DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter) · [FinOps Foundation](https://www.finops.org/) · 验证环境：待验证

## 概念：为什么需要

GPU 是 AI 应用上云后最贵的资源，但也是最「看不见」的资源：CPU 有 `kubectl top`，GPU 的利用率、显存占用、温度却默认不暴露给业务团队。三件事做不了，成本就失控：

- **利用率看不见**：卡在跑还是闲着，没人知道——「GPU 利用率低但账单高」是 AI 上云后的头号抱怨（[已必备]）
- **成本无法分摊**：多个团队共享一个 GPU 集群，月底账单是一笔糊涂账，说不清谁用了多少（[已必备]）
- **优化无从下手**：FinOps 的第一课是「先让成本可见，再谈降本」——看不见就谈不上治理

- **从哪来**：GPU 指标过去只有 NVIDIA 的 nvidia-smi 命令行，靠人肉巡检；DCGM（Data Center GPU Manager，NVIDIA 的数据中心 GPU 管理库）把指标标准化后，dcgm-exporter 成为「GPU → Prometheus」的事实标准路径
- **是什么**：dcgm-exporter 是 NVIDIA 官方的指标导出器，以 DaemonSet 形式跑在每个 GPU 节点上，把 DCGM 采集的 GPU 指标暴露成 Prometheus 格式；业务团队要做的只是「接入 + 看板 + 分摊」，装不装由平台决定
- **往哪去**：指标接入只是第一步，成本分摊（showback，成本分摊展示——先看见再治理）会演进为平台侧的「AI 账单」能力，与 3.5 的 token 成本合并成一张账单

**引导反思**：AI 应用的成本结构（GPU / token）决定商业模式是否成立——成本可见是治理的前提，不是锦上添花。

## 动手（验证环境：待验证）

两步把 GPU 装进现有 Prometheus 体系：

**第 1 步：DaemonSet 部署 dcgm-exporter**（DaemonSet：在每个节点上各跑一个 Pod 的工作负载类型，GPU 指标采集天然适合它；若平台已装，跳过此步直接要指标端点——装机是平台的事）

```yaml
# dcgm-exporter.yaml —— 每个 GPU 节点一个 Pod，暴露 9400 端口
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: dcgm-exporter
  namespace: gpu-monitoring
spec:
  selector:
    matchLabels:
      app: dcgm-exporter
  template:
    metadata:
      labels:
        app: dcgm-exporter
    spec:
      containers:
        - name: dcgm-exporter
          image: nvcr.io/nvidia/k8s/dcgm-exporter:latest
          ports:
            - containerPort: 9400
              name: metrics
```

**第 2 步：ServiceMonitor 接入 Prometheus + Grafana 看板**（ServiceMonitor：Prometheus Operator 里声明「采集哪些指标」的 CRD）

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: dcgm-exporter
  namespace: gpu-monitoring
spec:
  selector:
    matchLabels:
      app: dcgm-exporter
  endpoints:
    - port: metrics
```

```promql
# 单卡算力利用率（%）
DCGM_FI_DEV_GPU_UTIL

# 按 namespace 汇总平均利用率（需指标带 namespace 标签，见实用技巧）
avg by (namespace) (DCGM_FI_DEV_GPU_UTIL)
```

**成本分摊（showback）**：GPU 利用率 × 卡单价 → 按 namespace 汇总 → 每月出「AI 账单」。这是 FinOps 的第一课：先让成本可见，再谈降本。进阶路线见 [FinOps Foundation](https://www.finops.org/) 的框架 [进阶]。

## 实用技巧

- 核心指标就记三个：`DCGM_FI_DEV_GPU_UTIL`（算力利用率）、显存使用量、温度（显存 / 温度字段名待验证，以 `curl localhost:9400/metrics` 实际暴露为准）
- 利用率 ≠ 显存占用：算力利用率低但显存占满，说明模型常驻但请求稀疏——这是「利用率低但账单高」的典型形态
- 分摊的前提是标签：让指标带上 namespace / Pod 标签（或由平台在采集端补），否则按 namespace 汇总无从谈起——这是要和平台对齐的第一件事
- 与 3.5 网关的 token 成本合并：GPU 是「算力账单」，token 是「用量账单」，两张表合起来才是完整的 AI 成本视图

## CKAD 考点对照

无（Part 3 不在 CKAD 范围内；可呼应 Part 1.8 可观测性基础中的 Prometheus 概念）。

## 考察问题

- 为什么「GPU 利用率低但账单高」是常态？（线索：显存占用 vs 算力利用率；请求稀疏；批处理大小；排队策略）
- showback 和 chargeback 有什么区别？为什么多数团队从 showback 起步？（线索：财务流程复杂度；内部结算成本）
- dcgm-exporter 该由谁部署——业务团队还是平台团队？（线索：DaemonSet 需要节点级权限；本项目定位「使用者」）

## 经验之谈

- 观点（不署名）：「先让成本可见，再谈降本」——FinOps 的第一课不是省钱，是看清钱花在哪；多数团队 GPU 成本治理的第一步只是「把账单做出来」
- 观点转述：Charity Majors（Honeycomb 联合创始人，可观测性领域代表人物，见检索手册 §10）把可观测性定义为「能回答事先不知道的问题的能力」——成本治理同理：先让成本可问，再谈优化（出处：charity.wtf，其个人博客）

## 架构师视角

- 解决什么问题：GPU 成本不可见、多团队无法分摊、优化没有数据支撑
- 何时用：有 GPU 负载、多团队共享集群、月底要对账
- 何时不用：无 GPU 负载、单机原型阶段——先跑起来再说
- 权衡：指标粒度越细（按 Pod / 按秒）存储成本越高；showback 先粗后细，别一上来做 chargeback
- 固定三问：
  - 平台提供了什么能力边界？——dcgm-exporter 是否已部署、指标是否已暴露、是否已有成本视图
  - 业务接入点在哪？——Grafana 看板 / PromQL 查询 / 平台成本报表
  - 需要和基础设施团队对齐什么？——指标保留周期、namespace 标签规范、卡单价口径

## 核心收获表

| 能力维度 | 具体收获 |
|---|---|
| GPU 可观测 | 能查到 GPU 利用率 / 显存 / 温度，看懂「利用率低但账单高」 |
| 成本分摊 | 能按 namespace 汇总 GPU 成本，产出月度「AI 账单」 |
| 平台协作 | 知道哪些事找平台（装机 / 指标保留），哪些事自己做（看板 / 分摊） |

## 推荐开源项目表

| 项目 | 链接 | 研读重点 |
|---|---|---|
| DCGM Exporter | https://github.com/NVIDIA/dcgm-exporter | 指标字段、DaemonSet 部署方式 |
| Prometheus / Grafana | https://prometheus.io/ · https://grafana.com/ | 指标聚合查询、看板设计 |
| FinOps Foundation | https://www.finops.org/ | 成本治理框架 [进阶] |

## 常见问题排查

| 高频报错 / 现象 | 排查路径 |
|---|---|
| Grafana 查不到 GPU 指标 | dcgm-exporter Pod 是否 Running（`kubectl get pods -n gpu-monitoring`）→ ServiceMonitor 的 selector 是否匹配 → Prometheus target 是否发现 |
| DCGM_FI_DEV_GPU_UTIL 恒为 0 | 确认看的是算力利用率而非显存占用；换显存 / 温度字段对照 |
| 按 namespace 汇总为空 | 指标没有 namespace 标签——找平台在采集端补标签，或先按节点维度看 |
| 显存 / 温度字段名对不上 | 字段名待验证：`kubectl exec` 进 dcgm-exporter 容器 `curl localhost:9400/metrics` 看实际字段 |