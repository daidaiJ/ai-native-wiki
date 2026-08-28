# 1.10 企业云部署实践 [实用]

> 来源：[Kubernetes 官方文档](https://kubernetes.io/docs/)：Pod 生命周期（探针/优雅终止）· 镜像（imagePullPolicy/imagePullSecrets）· 调度与驱逐（反亲和/拓扑分布）· PDB
> 验证环境：待验证（kind v0.x + K8s v1.x，发布前须本地跑通）

## 为什么先学这个

本章的定位是**认知建立**，不是操作手册。企业生产部署的治理意识，比具体 YAML 更重要：遇到「镜像拉取超时」「启动被误杀」「发布断流」时，没有概念的人连问题都描述不清，更谈不上排查；和 AI 讨论部署设计时，问不到核心要点，得到的方案就不可信。先知道「有这回事」——每个主题解决什么问题、不治理会踩什么坑——再动手配，才是本章的学习顺序。每个主题末尾附「和 AI 沟通的提问要点」，把「问得到点子上」变成可练习的动作。

## 概念：为什么需要

**业务痛点**：[已必备] 大镜像首次调度到新节点拉取慢，启动超时、发布窗口拉长；[已必备] 服务启动慢被存活探针误杀，陷入「启动→被杀→重启」循环；[已必备] 发布与节点维护时服务直接断流。企业生产部署要过三道坎：**拉得动**（镜像预热）、**起得快**（启动治理）、**扛得住**（可用性配置）。

- **从哪来**：单副本裸跑、手动逐节点拉镜像、kill -9 式下线 → 多副本 + 探针 + 声明式预热 + 受控变更
- **是什么**：本章三个主题——镜像预热把「拉取时间」从发布路径挪到后台；启动治理用 startupProbe（启动探针）等机制给慢启动容器留出窗口；可用性配置用 PDB（自愿中断保护）、滚动更新策略、反亲和/拓扑分布、优雅终止把「能跑」变成「扛得住」
- **往哪去**：预热走向节点镜像预置/快照（托管集群 bootstrap 脚本）；启动治理走向启动就绪的标准化（探针 + 资源基线）；可用性配置走向平台化（平台强制 PDB/资源配额，业务只声明需求）

## 主题一：镜像预热（image pre-pull）

### 为什么需要

大镜像（AI 模型镜像 GB 级、基础镜像数百 MB）首次调度到新节点时，kubelet 要从镜像仓库拉取，拉取时间可能长达几分钟，直接拉长发布窗口、触发启动超时。节点池弹性扩缩容时新节点是「空盘」，冷启动拉镜像成为瓶颈。预热 = 在 Pod 真正调度之前，先把镜像拉到节点上。

### 认知要点（先建立意识）

- **不治理会踩的坑**：大镜像首次调度到新节点，拉取耗时超过启动窗口 → 发布超时、滚动更新卡住；节点池弹性扩容时新节点「空盘」，冷启动拉镜像成为瓶颈
- **要知道的关键事实**：imagePullPolicy 默认规则（tag 为 latest 默认 Always，显式 tag 默认 IfNotPresent）；预热是尽力而为，不保证新 Pod 一定命中已预热镜像；预热清单不跟版本同步 = 白预热；私有仓库没有 imagePullSecrets 时预热和调度都会失败
- **治理意识**：把「拉取时间」从发布路径挪到后台，是预热的核心思想；先确认镜像大小与节点池规模，再决定要不要预热、怎么预热

### 怎么做

**1. DaemonSet 预热（单镜像）**：DaemonSet 是确保每个（或部分）节点上运行一个 Pod 副本的工作负载对象。把目标镜像作为 initContainer（主容器启动前串行执行的初始化容器）的镜像，kubelet 在每个节点上自动拉取，预热 Pod 随节点加入自动调度、节点删除自动清理：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: image-puller
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: image-puller
  template:
    metadata:
      labels:
        name: image-puller
    spec:
      initContainers:
        - name: puller
          image: registry.example.com/ai/model:v1.0.0   # 预热目标镜像
          command: ["sh", "-c", "true"]
      containers:
        - name: pause
          image: registry.k8s.io/pause:3.9
```

> 验证环境：待验证。预热 Pod 完成即退出，DaemonSet 会把它重启——这是预期行为，预热动作发生在每次 Pod 创建时的镜像拉取阶段。

**2. 多镜像预热（脚本循环）**：预热清单放 ConfigMap，脚本循环拉取（containerd 运行时用 `crictl pull`）：

```bash
# 预热清单（ConfigMap 管理，发布流水线同步更新）
for img in $(cat /images.txt); do crictl pull "$img"; done
```

> 验证环境：待验证。脚本镜像需自带 crictl 或挂载宿主机运行时 socket，具体挂载方式随运行时/平台而异。

**3. imagePullPolicy 三值选择**（kubelet 拉取镜像的策略）：

- `Always`：每次创建容器都检查/拉取，保证最新，但慢——适合频繁发版、镜像 tag 易漂移的场景
- `IfNotPresent`：本地已有就不拉，启动快——配合预热的主力选项
- `Never`：只用节点本地镜像，不访问仓库——适合离线/测试环境
- 默认规则（官方文档事实）：tag 为 `latest` 或未指定时默认 `Always`，显式 tag 默认 `IfNotPresent`，`Never` 必须显式设置

**4. 私有仓库认证 imagePullSecrets**（Pod 访问私有镜像仓库的凭据）：

```bash
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=<user> --docker-password=<pass>
```

```yaml
spec:
  imagePullSecrets:
    - name: regcred
```

注意：imagePullSecrets 写在 Pod 模板的 spec 下（Deployment 里是 `template.spec`）；Secret 按 namespace 隔离，每个命名空间都要建（或由平台默认挂载）。

### 注意事项

- **与节点池扩缩容的配合**：DaemonSet 预热是「节点加入后」才拉，新 Pod 可能先于预热完成被调度——托管集群可在节点组 bootstrap 脚本里预拉关键镜像，或对关键节点池提前扩容；预热是尽力而为，不是硬保证
- **镜像版本漂移**：预热的是旧 tag，发布用新 tag（Always 策略）时预热白做；预热清单要与发布版本同步维护，否则「预热了但没拉到要用的版本」
- **仓库带宽与磁盘**：大镜像多节点同时拉会打爆仓库限流，可错峰/分批；预热镜像占用节点磁盘，注意 kubelet 镜像垃圾回收（imageGC 高/低水位，默认 85%/80%）可能回收预热镜像

### 和 AI 沟通的提问要点

向 AI 咨询镜像预热方案时，先说清：镜像大小（MB/GB 级）、节点池规模与扩容频率、拉取频率（每次发版都拉还是偶尔）、仓库是否私有/是否限流。核心问题：预热用 DaemonSet 还是节点 bootstrap 脚本？预热清单怎么跟发布版本同步？imagePullPolicy 该用 IfNotPresent 还是 Always？

## 主题二：Pod 快速启动治理

### 为什么需要

启动慢的常见原因：镜像拉取慢（见主题一）、初始化任务重（initContainer 下载模型/迁移数据）、依赖服务未就绪（DB/配置中心）、应用框架启动慢（JVM 等）。后果：发布窗口拉长、滚动更新卡住（新 Pod 不就绪旧 Pod 不摘流）、**被 liveness 误杀**——容器进程活着但还没就绪，存活探针失败触发重启，陷入「启动→被杀→重启」循环。

### 认知要点（先建立意识）

- **不治理会踩的坑**：慢启动容器被 liveness 误杀，陷入「启动→被杀→重启」循环；滚动更新时新 Pod 迟迟不就绪，旧 Pod 不摘流，发布卡死；就绪探针探测外部依赖，DB 一抖动全 Pod 摘流 → 雪崩
- **要知道的关键事实**：startupProbe 成功前 liveness/readiness 不生效；initContainer 串行执行且各自拉镜像，会叠加启动时间；CPU 超 limits 节流不杀，内存超 limits 直接 OOMKilled
- **治理意识**：启动慢要先定位根因（拉取/初始化/依赖/框架），再对症下药，不是一味调大探针参数

### 怎么做

**1. startupProbe**（启动探针，容器启动阶段专用的健康检查）：startupProbe 成功之前，liveness/readiness 不生效，给慢启动容器一个明确的启动窗口：

```yaml
spec:
  containers:
    - name: app
      image: registry.example.com/app:v1.0.0
      startupProbe:
        httpGet:
          path: /healthz
          port: 8080
        failureThreshold: 30
        periodSeconds: 10        # 启动窗口 = 30 × 10 = 300s
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /readyz
          port: 8080
        periodSeconds: 5
```

> 验证环境：待验证。livenessProbe 是存活探针（失败则重启容器），readinessProbe 是就绪探针（成功才接流量）。

**2. imagePullPolicy: IfNotPresent**：配合预热，跳过重复拉取（见主题一）。

**3. initContainer 最小化**：只放必须的初始化（等依赖、准备数据）；重的初始化移到应用启动流程或预热；多个 initContainer 串行执行且各自拉镜像，会叠加启动时间。

**4. 资源 requests 预留**：requests 是调度与驱逐的依据——设太低，节点超卖，运行时被 CPU 节流（throttling，限速）或内存 OOMKilled（被杀）；设太高，调度不出去（Pending）。

**5. 就绪探针与流量接入时机**：readinessProbe 成功才把 Pod 加入 Service 端点（接流量）；就绪探针要反映「真正能服务」，不是「进程活着」。

### 注意事项

- startupProbe 的窗口要覆盖最坏情况（冷启动 + 依赖等待），别和 liveness 用同一组短参数
- 就绪探针别探测外部依赖（DB 挂了全 Pod 摘流 → 雪崩）；就绪只查自身，外部依赖由调用方/平台兜底
- initContainer 串行 + 每个 initContainer 的镜像都要拉，数量越多启动越慢——能合并就合并
- requests 与 limits 分开设：CPU 超 limits 节流不杀，内存超 limits 直接 OOMKilled；limits 别拍脑袋，先压测拿基线

### 和 AI 沟通的提问要点

向 AI 咨询启动治理时，先说清：启动耗时（秒级/分钟级）、启动慢的环节（拉镜像/初始化/等依赖）、依赖哪些外部服务、探针路径与预期返回。核心问题：该用 startupProbe 还是调大 liveness 参数？initContainer 能不能合并或后移？requests 基线怎么定（有压测数据还是估算）？

## 主题三：服务可用性配置规范

### 为什么需要

生产可用性 = 冗余 + 受控变更 + 故障域隔离 + 优雅退出。单副本、无 PDB、无优雅终止的服务，在节点维护（drain）、发布、节点宕机时直接断流。从哪来→是什么→往哪去：单副本 → 多副本 + 反亲和 → 跨可用区拓扑分布；kill -9 → preStop 排空 + SIGTERM 优雅退出。

### 认知要点（先建立意识）

- **不治理会踩的坑**：节点维护（drain）时服务中断——没有 PDB，drain 直接把 Pod 全停；滚动更新期间流量中断——maxUnavailable 默认 25%，副本少时可能全停；单副本 + 无反亲和，节点宕机 = 服务全挂；没有优雅终止，存量请求被 SIGKILL 掐断
- **要知道的关键事实**：PDB 只保护自愿中断，不保护节点宕机；maxUnavailable: 0 + maxSurge: 1 是零中断滚动更新（需资源余量）；preStop + SIGTERM 处理时间必须小于 terminationGracePeriodSeconds（默认 30s）
- **治理意识**：可用性配置是「故障域思维」——副本要冗余，冗余要打散（节点/可用区），变更要受控（PDB/滚动更新），退出要体面（优雅终止）

### 怎么做

**1. 副本数冗余 + PDB**（PodDisruptionBudget，自愿中断保护——限制节点维护等自愿中断时最多能同时停几个 Pod）：

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: demo-pdb
spec:
  minAvailable: 2          # 或 maxUnavailable: 1
  selector:
    matchLabels:
      app: demo
```

> 验证环境：待验证。PDB 只保护自愿中断（节点 drain、滚动更新、主动删除），不保护非自愿中断（节点宕机、OOM）。

**2. 滚动更新策略**（maxSurge：最多超出期望副本数多少个；maxUnavailable：最多允许多少个不可用）：

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0     # 先起新的再杀旧的，零中断（需资源余量）
```

> 验证环境：待验证。默认值 maxSurge/maxUnavailable 均为 25%。

**3. 反亲和 + 拓扑分布**（把副本打散到不同节点/可用区，避免单点故障域一锅端）：

```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: demo
            topologyKey: kubernetes.io/hostname   # 尽量不跟同标签 Pod 同节点
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone    # 跨可用区均匀分布
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: demo
```

> 验证环境：待验证。反亲和是「尽量不跟同标签 Pod 同拓扑域」，拓扑分布约束（topologySpreadConstraints）是把 Pod 均匀打散到拓扑域（节点/可用区），maxSkew 控制最大不均衡。

**4. 优雅终止**（terminationGracePeriodSeconds：优雅终止宽限期，默认 30s；preStop：容器终止前执行的钩子）：

```yaml
spec:
  terminationGracePeriodSeconds: 30
  containers:
    - name: app
      lifecycle:
        preStop:
          exec:
            command: ["sh", "-c", "sleep 5"]   # 等 Service 端点摘除传播，或调用应用排空接口
```

删除流程：Pod 进入 Terminating → 从 Service 端点摘除 → 执行 preStop → 收到 SIGTERM → 应用处理完存量请求退出 → 超时被 SIGKILL。

**5. 资源 requests/limits**：

```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: "1"
    memory: 1Gi
```

requests 用于调度与 QoS 分级（requests=limits 为 Guaranteed 最高级，驱逐时最后动）；limits 用于运行时限制。

### 注意事项

- PDB 与滚动更新互相约束：滚动更新也属于自愿中断，受 PDB 限制；maxUnavailable: 0 的滚动更新在资源不足时可能卡住（新 Pod Pending，旧 Pod 不杀）
- 反亲和 preferred 是尽力而为，required 会降低调度成功率（小集群可能调度不出）；拓扑分布与反亲和叠加时注意约束冲突
- 单节点/单可用区集群没有 zone 标签，`topology.kubernetes.io/zone` 约束不生效（待验证）
- preStop sleep 是「等端点摘除」的社区通行做法，但 sleep + SIGTERM 处理时间必须小于 terminationGracePeriodSeconds，否则被 SIGKILL 强杀，优雅退出白配
- 优雅终止的正确姿势是应用自己处理 SIGTERM 排空，preStop 只是兜底

### 和 AI 沟通的提问要点

向 AI 咨询可用性配置时，先说清：副本数、可用性要求（能否接受短暂中断/中断多久）、是否跨可用区部署、节点维护频率、应用能否处理 SIGTERM 排空。核心问题：PDB 用 minAvailable 还是 maxUnavailable、值设多少？滚动更新 maxSurge/maxUnavailable 怎么配？反亲和用 preferred 还是 required？preStop 该 sleep 还是调排空接口？

## 实用技巧

- 预热清单用 ConfigMap 管理，发布流水线发版后触发一次预热更新，避免版本漂移
- 启动慢排查三板斧：`kubectl describe pod` 看 Events（Pulling/Pulled 时间戳、调度事件）、`kubectl logs` 看应用启动日志、`kubectl get events --sort-by=.lastTimestamp` 看全量事件
- 探针字段记不住就 `kubectl explain pod.spec.containers.startupProbe`
- 滚动更新卡住：`kubectl rollout status deployment/demo` + describe 新 Pod（Pending/ImagePullBackOff/CrashLoopBackOff 各有对应解法）
- 验证优雅终止：`kubectl delete pod` 后看应用日志是否收到 SIGTERM 并完成排空
- 节点维护前 `kubectl drain` 会受 PDB 约束，先 `kubectl describe pdb` 看 disruptionsAllowed（当前允许中断数）

## CKAD 考点对照

- 探针三件套（liveness/readiness/startup）是 CKAD 高频考点：会写、会解释 startup 与 liveness 的关系
- 滚动更新策略（maxSurge/maxUnavailable）与回滚（`kubectl rollout undo`）
- 资源 requests/limits 与 QoS 类
- initContainer 多容器 Pod 模式
- PDB、反亲和、拓扑分布属 CKA 考纲，CKAD 不直接考——但生产必备，按 CKA 标准学

## 考察问题

- 预热 DaemonSet 为什么能「自动跟随节点池扩容」？预热清单有 20 个镜像时怎么组织？（线索：DaemonSet 调度机制；脚本 + ConfigMap）
- startupProbe 与 livenessProbe 都用 failureThreshold: 3、periodSeconds: 10，慢启动容器会怎样？（线索：startup 未成功前 liveness 不生效）
- maxUnavailable: 0 + maxSurge: 1 的滚动更新在节点资源不足时会怎样？PDB 与滚动更新同时存在时谁约束谁？（线索：新 Pod Pending；PDB 管自愿中断总数）
- preStop sleep 5 秒为什么能降低流量损失？sleep 超过 terminationGracePeriodSeconds 会怎样？（线索：端点摘除传播延迟；SIGKILL）

## 经验之谈

- 探针是发布的第一道门：很多「启动崩溃」其实是 liveness 误杀，startupProbe 是慢启动容器的救命配置
- 预热是把拉取时间从发布路径挪到后台：对 GB 级 AI 大镜像收益最大，但清单不跟版本同步就是白预热
- 优雅终止是「最后 30 秒的体面」：preStop sleep 是通行 hack，正确做法是应用自己处理 SIGTERM；配了优雅终止却超时被强杀，等于没配
- 反亲和与拓扑分布是故障域思维：先看集群拓扑再配，单节点集群配了也白配
（观点转述，不署名；具体引用待补）

## 架构师视角

- **解决什么问题**：把「能跑」变成「扛得住」——启动快、发布稳、中断可控
- **何时用**：生产服务默认全配（探针 + 资源 + 滚动更新 + PDB + 优雅终止）；反亲和/拓扑分布按副本数与集群拓扑决定
- **何时不用**：可快速重建的批任务、单节点开发集群（PDB/反亲和无意义）
- **权衡**：可用性配置的复杂度 vs 收益；反亲和降低调度密度（资源利用率下降）；PDB 限制运维灵活性（drain 变慢）
- **固定三问**：
  - 平台提供了什么能力边界？节点池是否支持镜像预置/启动脚本？集群是否多可用区（zone 标签）？平台是否强制 PDB/资源配额/默认宽限期？
  - 业务接入点在哪？探针路径、排空接口、资源基线由业务提供；PDB/反亲和/拓扑分布由服务 owner 在清单里声明
  - 需要和基础设施团队对齐什么？节点池扩容与预热配合、镜像仓库带宽与限流、集群拓扑（可用区）、平台驱逐策略与默认 terminationGracePeriodSeconds

## 核心收获表

| 能力维度 | 具体收获 |
|---|---|
| 镜像预热 | 能写预热 DaemonSet，理解 imagePullPolicy 三值与 imagePullSecrets 认证 |
| 启动治理 | 能配 startupProbe 三件套，能定位启动慢根因（拉取/初始化/依赖/误杀） |
| 可用性规范 | 能配 PDB、滚动更新策略、反亲和、拓扑分布、优雅终止、资源限制 |
| 排障 | 能排查 Pending/ImagePullBackOff/CrashLoopBackOff/滚动更新卡住/发布断流 |

## 推荐开源项目

| 项目 | 链接 | 研读重点 |
|---|---|---|
| Kubernetes 官方文档（Pod 生命周期） | https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/ | 探针/优雅终止/重启策略，字段以官方为准 |
| Kubernetes 官方文档（镜像） | https://kubernetes.io/docs/concepts/containers/images/ | imagePullPolicy 默认规则/imagePullSecrets |
| Kubernetes 官方文档（调度与驱逐） | https://kubernetes.io/docs/concepts/scheduling-eviction/ | 反亲和/拓扑分布/驱逐 |
| Kubernetes 官方文档（PDB） | https://kubernetes.io/docs/tasks/run-application/configure-pdb/ | PDB 配置与自愿/非自愿中断 |
| kind | https://kind.sigs.k8s.io/ | 本地验证环境，`kind load docker-image` 模拟预热 |
| k9s | https://k9scli.io/ | 终端排障，快速看 Pod 状态与事件 |

## 常见问题排查

| 高频报错 | 排查路径 |
|---|---|
| Pod 一直 Pending | `kubectl describe pod` 看 Events：镜像拉取失败（认证/不存在/仓库不可达）还是资源不足（Insufficient cpu/memory） |
| ImagePullBackOff / ErrImagePull | 检查 imagePullSecrets、镜像 tag 是否存在、私有仓库地址与网络 |
| 启动后反复重启（CrashLoopBackOff） | `kubectl logs --previous` 看上次日志；怀疑 liveness 误杀就加 startupProbe 或调大 failureThreshold |
| 滚动更新卡住 | `kubectl rollout status` + describe 新 Pod；检查 maxSurge/maxUnavailable 与 PDB 约束 |
| 发布时流量中断 | 检查优雅终止配置（preStop/terminationGracePeriodSeconds）、readiness 是否在摘流前就绪 |
| 节点维护 drain 卡住 | PDB 阻止驱逐：`kubectl describe pdb` 看 disruptionsAllowed，确认 minAvailable 是否合理 |