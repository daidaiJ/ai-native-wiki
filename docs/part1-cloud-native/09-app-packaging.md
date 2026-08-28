# 1.9 应用打包与分发：Helm 与 Kustomize [实用]

> 来源：[Helm 官方文档](https://helm.sh/docs/) · [Kustomize](https://kustomize.io/) · [Artifact Hub](https://artifacthub.io/)
> 验证环境：待验证（kind v0.x + Helm v3.x，发布前须本地跑通）

## 概念：为什么需要

**业务痛点**：[已必备] 三份环境（dev/staging/prod）× 8 个资源对象 × 镜像 tag / 副本数 / 域名差异 = 裸 YAML 复制粘贴的维护噩梦——发布慢、环境不一致（「在我机器上是好的」）是团队规模化的第一道墙；[已必备] 消费者侧大促/秒杀/抢票/直播场景，服务不可用 = 直接营收损失与口碑崩塌，而多环境配置错误正是发布事故的头号来源。打包解决「一份资产，多环境差异」。

- **从哪来**：手工 sed 改 YAML → 社区分流两条路——Kustomize（K8s 原生 overlay 机制，无模板语言）与 Helm（模板 + 包管理器，K8s 生态的 apt/brew）
- **是什么**：Helm 用 Chart（应用包，含模板 + 默认值）和 values.yaml（环境差异化配置）实现「参数化 + 仓库分发」；Kustomize 用 base（公共清单）+ overlay（环境叠加层）实现「声明式打补丁」，产物仍是纯 YAML
- **往哪去**：Chart 走 OCI 化（OCI 是容器镜像与制品的通用仓库标准，chart 存进镜像仓库，与镜像同库同权限）；渲染从客户端移到 CD 端（CD 即持续部署流水线；GitOps——以 Git 仓库为唯一事实来源的声明式交付方式——下由 ArgoCD/Flux 在 CD 侧统一渲染，本地不再有「渲染产物漂移」问题）

## 动手（命令待验证）

```bash
# 1. 生成一个 Chart 骨架并看结构
helm create demo && tree demo

# 2. values 差异化安装（upgrade --install 幂等发布：重复执行结果一致）
helm upgrade --install demo ./demo -n demo --create-namespace --set image.tag=v1.0.0

# 3. 渲染预检：先看到最终 YAML 再进集群（dry-run = 预演，不真正提交）
helm template demo ./demo | kubectl apply --dry-run=server -f -

# 4. 出问题回滚
helm rollback demo 1

# Kustomize 路线：overlays/prod/kustomization.yaml 里改 replicas/镜像 tag，然后
kubectl apply -k overlays/prod
```

Kustomize 的 overlay 写法示例（`overlays/prod/kustomization.yaml`）：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
replicas:
  - name: demo
    count: 3
images:
  - name: demo
    newTag: v1.0.0
```

> 验证环境：待验证。以上命令与 YAML 发布前须在本地 kind 集群跑通。

### Helm 核心教程引导（日常操作速查 + 进阶路径）

**入门三步**（上方动手已覆盖）：`helm create` 生成骨架 → `helm upgrade --install` 幂等发布 → `helm rollback` 回退。日常使用只需掌握下面这张速查表：

| 场景 | 命令 | 说明 |
|---|---|---|
| 首次/重复部署 | `helm upgrade --install <release> <chart> -n <ns> --create-namespace` | 幂等，日常唯一入口（不用 `helm install`） |
| 查看发布列表 | `helm list -n <ns>` | 看 release 状态（deployed / failed） |
| 查看发布详情 | `helm status <release> -n <ns>` | 含 manifest 摘要与状态 |
| 升级前预检 | `helm template <release> <chart> \| kubectl apply --dry-run=server -f -` | 先看到最终 YAML 再进集群 |
| 升级前看差异 | `helm diff upgrade <release> <chart> -n <ns>` | 需 diff 插件，逐行差异防盲发 |
| 查生效配置 | `helm get values <release> --revision N -n <ns>` | 某次发布实际生效的配置 |
| 回退 | `helm rollback <release> <版本号> -n <ns>` | 出问题秒级回退 |
| 卸载 | `helm uninstall <release> -n <ns>` | 清理 release |

**进阶引导（学习者自己深入）**：

- Chart 模板语法（values / 模板函数 / 子 chart）：官方文档 Chart Template Guide——先会改 values，再学模板
- Chart 仓库与 OCI 分发：`helm repo add` / `helm pull`；上 Artifact Hub 找现成 chart 拆开学习写法
- 依赖管理：Chart.yaml 的 `dependencies`（多组件应用）
- 生产实践：chart 版本化、values 分层（环境差异用 `values-<env>.yaml`）、CI 里 `helm lint` + template 校验
- 学习路径：官方文档 https://helm.sh/docs/ 按「Getting Started → Chart Template Guide → Advanced」顺序推进

## 实用技巧

- `helm diff` 插件（`helm plugin install ...`）：升级前看逐行差异，防止盲发
- `helm get values demo --revision 2`：查某次发布实际生效的配置
- 上 Artifact Hub 找现成 chart（Bitnami 系质量较稳），`helm pull bitnami/nginx --untar` 拆开学习模板写法
- `kubectl diff -k overlays/prod`：先 diff 再 apply，GitOps 前的最后一道保险

## CKAD 考点对照

CKAD 不考 Helm/Kustomize；但 `kubectl apply -k` 考试可用，且「模板生成 → dry-run → apply」思路与考试 `--dry-run=client -o yaml` 一脉相承。

## 考察问题

- ArgoCD 为什么能同时支持 Helm 与 Kustomize？（线索：CD 端渲染 vs 客户端渲染；chart 依赖解析发生在哪一侧）
- 为什么全 GitOps 流水线的团队更倾向 Kustomize？（线索：声明式最终状态；渲染产物是否入库）

## 经验之谈

对自有应用 + GitOps，「Kustomize 更贴近声明式」是社区通行判断；Helm 的 Go template 调试成本高（模板错误难定位）也是长期争议点。对分发/安装第三方中间件，Helm 生态无可替代。（观点转述，具体引用待补）

## 架构师视角

- **解决什么问题**：一份资产多环境差异；第三方应用的标准化分发与升级
- **何时用 Helm**：装中间件（PG/Redis/Kafka 类）、差异维度多、团队已有 chart 资产
- **何时不用**：自家应用 + 纯 GitOps → Kustomize 更轻；甚至可以直接用 Flux 的 Kustomization API 在 CD 端渲染
- **权衡**：Helm 表达力强但复杂度换来的；Kustomize 简单但环境差异超过约三成时 overlay 会失控——差异规模是分界线
- **固定三问**：
  - 平台提供了什么能力边界？平台是否提供 chart 仓库 / OCI registry / CD 端渲染服务，决定本地要不要保留渲染步骤
  - 业务接入点在哪？应用清单以 chart 还是 kustomization 形式提交，取决于平台 CI/CD 的约定
  - 需要和基础设施团队对齐什么？chart 仓库权限、镜像与 chart 同库策略、CD 端渲染的产物校验方式

## 核心收获表

| 能力维度 | 具体收获 |
|---|---|
| 打包 | 能创建 Chart 骨架，用 values 差异化安装，回滚发布 |
| 分发 | 能从 Artifact Hub 拉取并拆解第三方 chart |
| 声明式打补丁 | 能用 base/overlay 组织多环境清单，`kubectl apply -k` 落地 |
| 选型判断 | 能按「自有应用 vs 第三方中间件」「差异规模」做出 Helm/Kustomize 决策 |

## 推荐开源项目

| 项目 | 链接 | 研读重点 |
|---|---|---|
| Helm | https://helm.sh/docs/ | Chart 模板/values/仓库，分发事实标准 |
| Kustomize | https://kustomize.io/ | K8s 原生 overlay，GitOps 配套 |
| Artifact Hub | https://artifacthub.io/ | 制品发现，找现成 chart |
| ArgoCD / Flux | https://argoproj.github.io/cd/ · https://fluxcd.io/ | GitOps 双雄，1.9 的「往哪去」 |

## 常见问题排查

| 高频报错 | 排查路径 |
|---|---|
| `Error: template: ...` 渲染报错 | 模板语法错误：`helm template` 单独渲染定位错误行；核对 values 键名与模板引用是否一致 |
| `release ... failed: ... already exists` | 用 `helm upgrade --install` 而非 `helm install`；或 `helm uninstall` 后重装 |
| `kubectl apply -k` 报 overlay 合并冲突 | 检查 base 与 overlay 的 patch 字段是否冲突；`kubectl diff -k` 先看差异 |
| `--dry-run=server` 被拒 | 先 `--dry-run=client` 排除语法问题，再看服务端报错（如被 PSA 拒绝，见 [1.3 增强](03a-pod-security.md)） |