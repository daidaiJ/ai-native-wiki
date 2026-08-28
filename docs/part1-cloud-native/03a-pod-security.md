# 1.3 增强：Pod Security Admission（PSA）[实用]

> 本文是 1.3 配置与安全（ConfigMap / Secret / ServiceAccount / ResourceQuota / SecurityContext）的增强小节，聚焦「Pod 安全基线」。
> 来源：[K8s 官方：Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) · 验证环境：待验证

## 概念：为什么需要

**业务痛点**：[已必备] 平台方统一安全基线后，业务 Pod 突然被拒（报错形如「violates PodSecurity」），发布卡死；[已必备] 安全合规审计要求「非 root 运行」等基线，业务侧不知道从哪改起。PSA（Pod Security Admission，Pod 安全准入）是 kube 内建准入控制（Admission Control，请求进入集群前的检查关卡），对业务开发者只需要三件事：

1. 三档安全级别：`privileged`（不限制）/ `baseline`（防明显提权）/ `restricted`（最严，要求非 root、只读根文件系统、去掉多余 capability）
2. 它是 namespace 上的标签，不是新对象
3. 排障识别：Pod 被拒的报错形如「violates PodSecurity "restricted:latest"」——去找平台方要该 ns 的 profile，对照改 SecurityContext（Pod/容器的安全上下文，配置运行身份与权限的字段）

- **从哪来**：PSP（PodSecurityPolicy，Pod 安全策略）已于 1.25 移除，继任者 PSA 是 kube 内建准入控制，无需额外部署
- **是什么**：namespace 标签 + 三档标准（PSS，Pod Security Standards）
- **往哪去**：策略引擎（Kyverno/OPA）做更细的准入控制，PSA 是平台默认基线

## 动手（命令待验证）

```bash
# 1. 查看 namespace 当前安全档位
kubectl get ns --show-labels

# 2. 平台方设档（业务侧一般无权限，此处演示机制）
kubectl label ns prod pod-security.kubernetes.io/enforce=restricted --overwrite

# 3. 部署后 Pod 被拒，看报错
kubectl describe pod <pod-name>
# 报错形如：Error creating: pods ... violates PodSecurity "restricted:latest"

# 4. 对照 restricted 要求改 SecurityContext（关键字段）：
#    runAsNonRoot: true（非 root 运行）
#    seccompProfile: RuntimeDefault（seccomp 是 Linux 系统调用过滤机制）
#    allowPrivilegeEscalation: false
#    capabilities.drop: ["ALL"]（capability 是 Linux 内核权限位，drop 即去掉多余权限）
#    readOnlyRootFilesystem: true（只读根文件系统）
```

一个 restricted 合规的容器 SecurityContext 示例：

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
  seccompProfile:
    type: RuntimeDefault
  capabilities:
    drop: ["ALL"]
  readOnlyRootFilesystem: true
```

> 验证环境：待验证。以上命令与 YAML 发布前须在本地 kind 集群跑通。

## 实用技巧

- 先 `audit`/`warn` 模式试点再 `enforce`：audit 只记录、warn 只告警，不拦 Pod——平台方上线新档位前先用这两种模式摸底存量 workload
- `kubectl explain pod.spec.securityContext` 查字段定义（kubectl 技巧贯穿）
- 被拒时先看 ns 标签（`kubectl get ns <ns> -o yaml` 的 `pod-security.kubernetes.io` 标签），再对照 PSS 文档逐条核对

## CKAD 考点对照

CKAD 不考 PSA 本身，但 `restricted` 的要求（`runAsNonRoot`、`seccompProfile` 等）与 Configuration 域的 SecurityContext 考点直接呼应——考试会写 SecurityContext，这里补上「平台为什么要求你写」。

## 考察问题

- 为什么 PSA 用 namespace 标签而不是新对象？（线索：标签即声明，无额外 CRD 依赖；与 RBAC 的 namespace 维度一致）
- enforce / audit / warn 三种模式分别适合什么阶段？（线索：先摸底、再告警、后拦截）
- 你的 Pod 被拒时，改 Pod 还是找平台方改档位？边界在哪？（线索：基线是平台职责，业务侧改到合规）

## 经验之谈

「先 audit 后 enforce」是社区通行做法：直接 enforce 会把存量 workload 全部拦在门外，先观察再收紧。（观点转述，具体引用待补）

## 架构师视角

- **解决什么问题**：平台统一安全基线，业务 Pod 合规运行
- **何时用**：平台已设档（enforce）时，业务侧按档位改 SecurityContext；自己搭实验集群时按需设档
- **何时不用**：单机实验、无安全要求的临时环境，privileged 即可
- **权衡**：restricted 最安全但约束多（只读根文件系统、非 root），部分老镜像不兼容——基线档位是平台与业务的共同决策
- **固定三问**：
  - 平台提供了什么能力边界？平台设了哪档 profile、是否提供豁免通道（如 ValidatingAdmissionPolicy 例外）
  - 业务接入点在哪？Pod 的 SecurityContext 字段 + namespace 标签（只读）
  - 需要和基础设施团队对齐什么？当前 ns 的档位、被拒 workload 的豁免流程、新档位上线时间表

## 核心收获表

| 能力维度 | 具体收获 |
|---|---|
| 识别 | 能读懂「violates PodSecurity」报错并定位 ns 档位 |
| 合规 | 能把 Pod 改到 restricted 合规（runAsNonRoot / seccomp / capabilities） |
| 协作 | 知道设档是平台职责，业务侧走豁免/对齐流程 |

## 推荐开源项目

| 项目 | 链接 | 研读重点 |
|---|---|---|
| Pod Security Standards（官方） | https://kubernetes.io/docs/concepts/security/pod-security-standards/ | 三档标准原文，排障对照表 |
| Kyverno | https://kyverno.io/ | 平台方可能用的策略引擎（L2 观察，不自己搭） |
| OPA | https://www.openpolicyagent.org/ | 策略引擎另一主流（L2 观察） |

## 常见问题排查

| 高频报错 | 排查路径 |
|---|---|
| `violates PodSecurity "restricted:latest"` | 看 ns 标签确认档位 → 对照 PSS 文档逐条核对 SecurityContext → 改到合规或走平台豁免 |
| 镜像要求 root 运行 | restricted 下无解：换镜像，或与平台对齐档位/申请豁免 |
| 只读根文件系统后应用写不了临时文件 | 挂 emptyDir（随 Pod 生命周期存在的临时卷）到可写路径（如 /tmp），而不是放开 readOnlyRootFilesystem |